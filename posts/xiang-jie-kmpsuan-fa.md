<!-- 
.. link: 
.. description: 
.. tags: 算法, 字符串
.. date: 2013/05/07 20:09:38
.. title: 详解KMP算法
.. slug: xiang-jie-kmpsuan-fa
-->

最近要实现关键字过滤功能，小看了一些经典的字符串匹配算法。  
本文要介绍的是KMP算法（**Knuth–Morris–Pratt Algorithm**）,那个Knuth应该再熟悉不过了。  

KMP算法是从朴素匹配算法改进而来。回忆一下朴素的匹配算法是怎么完成字符串的匹配的呢？  
1. 将原串和模式串左对齐，然后一位一位比较，直到有一个字符不匹配  
![KMP1.png](../galleries/KMP/KMP1.png "")  
2. 发现第二位的B和C不匹配，模式串右移一位  
![KMP2.png](../galleries/KMP/KMP2.png "")  
3. 重复这个流程，直到找到完全匹配的子串或者匹配失败。

但这过程中显然有多比较的地方。如，原串为ABCDEABCDF，模式串为ABCDF。第一轮可以发现E和F不匹配  
![KMP3.png](../galleries/KMP/KMP3.png "")  
很显然右移一位必定不匹配，这时模式串可以直接右移4位跳过ABCD，从E开始再次比较  
![KMP4.png](../galleries/KMP/KMP4.png "")  

那是不是跳过的长度就是前面相同部分的长度呢？其实不是这样的，这种直接跳过前面相同部分的做法在某些情况下会有问题。如，原串为ABCDABCDABF，模式串为ABCDABF，直接跳过相同部分就会遗漏匹配的串  
![KMP5.png](../galleries/KMP/KMP5.png "")  
所以跳过的长度并不是前面完全匹配的部分，可以跳过的长度一般存储在模式串的partial match table中，即KMP算法需要对模式串进行预处理。

先来看看这个partial match table在跳过的过程中是怎么用的，然后再来考察计算partial match table的算法。  
ABCDABF的partial match table如下：  
![KMP6.png](../galleries/KMP/KMP6.png "")  
可以跳过的长度 = 当前已匹配长度 - 最后一个字母在partial match table中的值。例如：  
![KMP7.png](../galleries/KMP/KMP7.png "")  
当发现C和F不匹配时，根据公式，当前已匹配串ABCDAB长度为6, 最后一个字母B在partial match table中的对应值为2，所以可以跳过的长度 = 6 - 2 = 4，即：  
![KMP8.png](../galleries/KMP/KMP8.png "")  
这样就能正确匹配了。

partial match table中每个位置i记录的其实就是从0到i的子串中，同时出现在子串前缀和后缀中的最大长度。还是以上面那个例子为例：  

* A没有前缀或后缀，所以长度为0
* AB前缀为A，后缀为B，没有相同部分，长度为0
* ABC前缀为A、AB，后缀为BC、C，没有相同部分，长度为0
* ABCD前缀为A、AB、ABC，后缀为BCD、CD、D，没有相同部分，长度为0
* ABCDA前缀为A、AB、ABC、ABCD，后缀为BCDA、CDA、DA、A，相同部分为A，长度为1
* ABCDAB前缀为A、AB、ABC、ABCD、ABCDA，后缀为BCDAB、CDAB、DAB、AB、B，相同部分为AB，长度为2
* ABCDABF前缀为A、AB、ABC、ABCD、ABCDA、ABCDAB，后缀为BCDABF、BCDAB、CDAB、DAB、AB、B，没有相同部分，长度为0

具体代码参考[https://github.com/hbprotoss/codejam/blob/master/kmp.py](https://github.com/hbprotoss/codejam/blob/master/kmp.py "")

全文完。