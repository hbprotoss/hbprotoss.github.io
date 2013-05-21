<!-- 
.. link: 
.. description: 
.. tags: 数据结构, 算法, Python
.. date: 2013/05/21 19:48:53
.. title: Trie树的Python实现
.. slug: trieshu-de-pythonshi-xian
-->

又是一个由需求驱动的算法学习的例子。

最近[weii](https://github.com/hbprotoss/weibo "")需要实现一个这样的功能：在发送AT好友的时候能给出自动补全的列表。

最先想到的是当我给出一个用户名的前几个字的时候能自动提示以这个字开头的所有用户名列表（虽然最后发现这是个很2的解决方案-_-），所以最理想的数据结构就是Trie树，也就是字典树。

以前只听过Trie树，现在实际要用了就得把他落到实处了。

首先是[Trie树在维基百科上的定义](http://zh.wikipedia.org/wiki/Trie "")：在计算机科学中，trie，又称前缀树，是一种有序树，用于保存关联数组，其中的键通常是字符串。与二叉查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

说白了就跟在字典里查单词一样：先拿第一个字母在根节点查找下一结点的位置，如果找到就拿第二个字母在刚刚找到的节点下继续往下查找；如果找不到就说明这个字符串在树中不存在。

C语言实现的代码也在维基百科上：[http://zh.wikipedia.org/wiki/Trie#.E5.AE.9E.E4.BE.8B](http://zh.wikipedia.org/wiki/Trie#.E5.AE.9E.E4.BE.8B "")

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
 
#define TREE_WIDTH 256
 
#define WORDLENMAX 128
 
struct trie_node_st {
        int count;
        struct trie_node_st *next[TREE_WIDTH];
};
 
static struct trie_node_st root={0, {NULL}};
 
static char *spaces=" \t\n/.\"\'()";
 
static int
insert(const char *word)
{
        int i;
        struct trie_node_st *curr, *newnode;
 
        if (word[0]=='\0') {
                return 0;
        }
        curr = &root;
        for (i=0; ; ++i) {
                if (word[i] == '\0') {
                        break;
                }
                if (curr->next[ word[i] ] == NULL) {
                        newnode=(struct trie_node_st*)malloc(sizeof(struct trie_node_st));
                        memset(newnode, 0, sizeof(struct trie_node_st));
                        curr->next[ word[i] ] = newnode;
                } 
                curr = curr->next[ word[i] ];
        }
        curr->count ++;
 
        return 0;
}
 
static void
printword(const char *str, int n)
{
        printf("%s\t%d\n", str, n);
}
 
static int
do_travel(struct trie_node_st *rootp)
{
        static char worddump[WORDLENMAX+1];
        static int pos=0;
        int i;
 
        if (rootp == NULL) {
                return 0;
        }
        if (rootp->count) {
                worddump[pos]='\0';
                printword(worddump, rootp->count);
        }
        for (i=0;i<TREE_WIDTH;++i) {
                worddump[pos++]=i;
                do_travel(rootp->next[i]);
                pos--;
        }
        return 0;
}
 
int
main(void)
{
        char *linebuf=NULL, *line, *word;
        size_t bufsize=0;
        int ret;
 
        while (1) {
                ret=getline(&linebuf, &bufsize, stdin);
                if (ret==-1) {
                        break;
                }
                line=linebuf;
                while (1) {
                        word = strsep(&line, spaces);
                        if (word==NULL) {
                                break;
                        }
                        if (word[0]=='\0') {
                                continue;
                        }
                        insert(word);
                }
        }
 
/* free(linebuf); */
 
        do_travel(&root);
 
        exit(0);
}
```

但这个版本的实现有两个问题：

1. 非常耗内存，一个节点下必须有TREE_WIDTH个子节点，不管子节点代表的字母是否出现在Trie树里。这是非常暴力的哈希。。。
2. 设定了这个TREE_WIDTH也就意味着这个实现只支持ASCII表中的字符作键，不支持中文。

Python内置的dict是用哈希实现的，正好可以解决这两个问题。（dict的基本原理可以参考[《Python源码剖析》阅读笔记：第五章-dict对象](http://blog.csdn.net/digimon/article/details/7875789 "")）

1. dict采用的是开放寻址法解决冲突，节省了内存，但时间复杂度还是O(1)。
2. dict这个哈希表里可以放任意字符作为键，中文当然也不例外。

Python版的关键改造就是节点的next表用dict代替，维护的是`字符->子节点`的映射。查找时，若待查询字符是next里的一个键就说明该字符在Trie树里，以这个键得到值就能找到下一节点。插入时也只要插入`字符->子节点`的映射就可以了。

具体代码在：[https://github.com/hbprotoss/codejam/blob/master/trie.py](https://github.com/hbprotoss/codejam/blob/master/trie.py "")

```python
#!/usr/bin/env python3

class Trie:
    root = dict()

    def insert(self, string):
        index, node = self.findLastNode(string)
        for char in string[index:]:
            new_node = dict()
            node[char] = new_node
            node = new_node

    def find(self, string):
        index, node = self.findLastNode(string)
        return (index == len(string))

    def findLastNode(self, string):
        '''
        @param string: string to be searched
        @return: (index, node).
            index: int. first char(string[index]) of string not found in Trie tree. Otherwise, the length of string
            node: dict. node doesn't have string[index].
        '''
        node = self.root
        index = 0
        while index < len(string):
            char = string[index]
            if char in node:
                node = node[char]
            else:
                break
            index += 1
        return (index, node)

    def printTree(self, node, layer):
        if len(node) == 0:
            return '\n'

        rtns = []
        items = sorted(node.items(), key=lambda x:x[0])
        rtns.append(items[0][0])
        rtns.append(self.printTree(items[0][1], layer+1))

        for item in items[1:]:
            rtns.append('.' * layer)
            rtns.append(item[0])
            rtns.append(self.printTree(item[1], layer+1))

        return ''.join(rtns)

    def __str__(self):
        return self.printTree(self.root, 0)

if __name__ == '__main__':
    tree = Trie()
    while True:
        src = input()
        if src == '':
            break
        else:
            tree.insert(src)
        print(tree)
```

全文完。