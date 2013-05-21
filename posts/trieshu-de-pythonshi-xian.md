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

但这个版本的实现有两个问题：

1. 非常耗内存，一个节点下必须有TREE_WIDTH个子节点，不管子节点代表的字母是否出现在Trie树里。这是非常暴力的哈希。。。
2. 设定了这个TREE_WIDTH也就意味着这个实现只支持ASCII表中的字符作键，不支持中文。

Python内置的dict是用哈希实现的，正好可以解决这两个问题。（dict的基本原理可以参考[《Python源码剖析》阅读笔记：第五章-dict对象](http://blog.csdn.net/digimon/article/details/7875789 "")）

1. dict采用的是开放寻址法解决冲突，节省了内存，但时间复杂度还是O(1)。
2. dict这个哈希表里可以放任意字符作为键，中文当然也不例外。

Python版的关键改造就是节点的next表用dict代替，维护的是`字符->子节点`的映射。查找时，若待查询字符是next里的一个键就说明该字符在Trie树里，以这个键得到值就能找到下一节点。插入时也只要插入`字符->子节点`的映射就可以了。

具体代码在：[https://github.com/hbprotoss/codejam/blob/master/trie.py](https://github.com/hbprotoss/codejam/blob/master/trie.py "")

全文完。