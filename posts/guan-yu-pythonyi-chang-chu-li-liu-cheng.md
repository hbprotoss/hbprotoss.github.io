<!-- 
.. title: 关于Python异常处理流程
.. slug: guan-yu-pythonyi-chang-chu-li-liu-cheng
.. date: 2014-10-23 12:27:56 UTC+08:00
.. tags: 
.. link: 
.. description: 
.. type: text
-->


```python
def f():
    try:
        print 'try'
        raise NameError
    except Exception, e:
        print 'except'
        raise KeyError
    else:
        print 'else'
        raise IOError
    finally:
        print 'finally'
        raise ValueError
```


在以上代码中，无论try里有没有异常，走得是except还是else，最终抛出的都是finally中的ValueError。为此小记一下Python里的异常处理流程。

Python当前状态的异常信息存储在线程状态PyThreadState中，包括type, value, traceback。无论是主动raise还是bug触发，异常信息最终都会通过PyErr_Restore(Python/errors.c)被修改。

异常处理的主要流程在PyEval_EvalFrameEx(Python/ceval.c)里，也是Python虚拟机的核心处理流程。


先说明几个概念以便理解：

1. Python虚拟机里PyEval_EvalFrameEx干的活就是物理机里CPU干的活，根据指令作出相应动作
2. frame：一个函数的执行环境，可以理解为物理机上EBP, ESP等寄存器确定的函数调用栈，外加一点别的状态信息
3. blockstack：也是一个栈，隶属于一个frame。每当遇到循环或者try异常处理块都会压入一个block对象，块流程完了弹出
4. traceback：一个链表，记录发生异常时候的调用栈信息

```c

PyObject *
PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{

     register enum why_code why; // 每一条指令处理完之后的状态，用来标记是否有异常

     switch(opcode){} // 一个处理字节码指令的巨大switch
     ...

     // 记录traceback，用来在异常最终没有处理时打印调用栈信息
     // 这个why ==WHY_EXCEPTION条件很重要，如果在finally中发生异常，则why的值是WHY_RERAISE，因此不会进入这个条件记录新的traceback，而是直接覆盖当前的traceback，所以在最终的调用栈里，finally异常之前try/except/else中的异常信息也就被覆盖了
       if (why == WHY_EXCEPTION) {
            PyTraceBack_Here(f);

            if (tstate->c_tracefunc != NULL)
                call_exc_trace(tstate->c_tracefunc,
                               tstate->c_traceobj, f);
        }

     // 这种是finally中有异常的状态，reraise
       if (why == WHY_RERAISE)
            why = WHY_EXCEPTION;

     // 栈帧展开，主要为了在当前frame，沿着block栈向上寻找有没有try/except来接住异常

fast_block_end:
        while (why != WHY_NOT && f->f_iblock > 0) {
            /* Peek at the current block. */
            PyTryBlock *b = &f->f_blockstack[f->f_iblock - 1];

           ......

            /* Now we have to pop the block. */
            f->f_iblock--;

           ......

          // 在finally块中
            if (b->b_type == SETUP_FINALLY ||
                ......
                ) {

               // 有异常(raise xxxError或bug触发的情况)
                if (why == WHY_EXCEPTION) {
                    PyObject *exc, *val, *tb;

                    // 取出异常信息，压入执行栈
                    PyErr_Fetch(&exc, &val, &tb);
                    if (tb == NULL) {
                        Py_INCREF(Py_None);
                        PUSH(Py_None);
                    } else
                        PUSH(tb);
                    PUSH(val);
                    PUSH(exc);
                }
                else {
                    // return或者continue语句
                    if (why & (WHY_RETURN | WHY_CONTINUE))
                        PUSH(retval);
                    v = PyInt_FromLong((long)why);
                    PUSH(v);
                }
               // 跳转到END_FINALLY指令进行扫尾工作
                why = WHY_NOT;
                JUMPTO(b->b_handler);
                break;
            }
        } /* unwind stack */

     // 函数返回值为null说明函数中发生异常未处理

    if (why != WHY_RETURN)
        retval = NULL;

     // 栈帧回退，取回上一层函数执行信息

    tstate->frame = f->f_back;

    return retval;
}
```


补充一点：

dis模块可以“反汇编”Python字节码变成“Python汇编语言”，下面就是示例代码的反汇编结果：

```

  6           0 SETUP_FINALLY           63 (to 66)
              3 SETUP_EXCEPT            15 (to 21)

  7           6 LOAD_CONST               1 ('try')
              9 PRINT_ITEM
             10 PRINT_NEWLINE

  8          11 LOAD_GLOBAL              0 (NameError)
             14 RAISE_VARARGS            1
             17 POP_BLOCK
             18 JUMP_FORWARD            30 (to 51)

  9     >>   21 DUP_TOP
             22 LOAD_GLOBAL              1 (Exception)
             25 COMPARE_OP              10 (exception match)
             28 POP_JUMP_IF_FALSE       50
             31 POP_TOP
             32 STORE_FAST               0 (e)
             35 POP_TOP

 10          36 LOAD_CONST               2 ('except')
             39 PRINT_ITEM
             40 PRINT_NEWLINE

 11          41 LOAD_GLOBAL              2 (KeyError)
             44 RAISE_VARARGS            1
             47 JUMP_FORWARD            12 (to 62)
        >>   50 END_FINALLY

 13     >>   51 LOAD_CONST               3 ('else')
             54 PRINT_ITEM
             55 PRINT_NEWLINE

 14          56 LOAD_GLOBAL              3 (IOError)
             59 RAISE_VARARGS            1
        >>   62 POP_BLOCK
             63 LOAD_CONST               0 (None)

 16     >>   66 LOAD_CONST               4 ('finally')
             69 PRINT_ITEM
             70 PRINT_NEWLINE

 17          71 LOAD_GLOBAL              4 (ValueError)
             74 RAISE_VARARGS            1
             77 END_FINALLY
             78 LOAD_CONST               0 (None)
             81 RETURN_VALUE
```