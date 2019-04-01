# 常见字符串匹配算法以及JS的Sring.prototype.indexOf()源码分析
提到常见的字符串匹配算法，一般来说我们会想到一个是朴素算法（暴力破解），一个是比较巧妙的KMP算法
## KMP算法图解
如果要在一个主串ABCDABCDEF里找关键字ABCDABE，有哪些方法呢？
我们很容易想到用暴力破解方法实现关键字搜索，如下图所示

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic1.png" height="100">

比较i指针和j指针指向的变量是否相同，如果相同，i和j同时后移

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic2.png" height="100">

本文约定待查找的字符串为主串master用M表示，关键字就是需要匹配的模式pattern，用P表示。   
到上图发现M[i]!=P[j],于是j回到开头，将关键字右移一位，然后继续重复昨天的故事。

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic3.png" height="100">

上述是最坏的情况，一直比较到最后一位才不匹配，这种最坏情况下时间复杂度是M*N，或许有人会觉得就这么暴力破解也行，不会这么巧每次都要到最后几位才发现不匹配。   
然而数据在计算机里都是以0101这种二进制形式存储的，比如字符1是00110001，字符2是00110010，很容易发生直到最后几个字符才出现不匹配的情况，所以对于这样低效的算法,我们是不能容忍的，因此有了本文接下来介绍的KMP算法。   
当发现最后一个字符不匹配时，如果我们人为查找，不会重头一个一个匹配，而是会像下图这样查找，i不动，j移动到下图中位置。   
因为根据已经匹配的信息，我们知道M中指针i前面的两个AB和P中j前面的两个AB是相同的

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic4.png" height="100">

KMP算法就是利用已经部分匹配这个有效信息，保持i指针不回溯，移动模式串的j指针到有效的位置。   

*关键点在于当pattern中某个字符与主串不匹配时，j指针要移动到哪？*

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic5.png" height="100">

由上图，我们可以发现，当匹配失败时，j要移动的下一个位置k。存在着这样的性质：

**pattern中最前面的k个字符和j之前的最后k个字符是一样的。**

***即P[0,k-1] == P[j-k,j-1]***

那么到底为什么可以移动到k呢？

可以证明：   
当M[i] != P[j]   
已经有M[i-j,i-1] == P[0,j-1]   
又有P[0,k-1] == P[j-k,j-1]   
***可得 M[i-k,i-1] == P[0,k-1]***  

所以我们可以直接从P[k]开始比较。

同时，我们还发现 *k是与主串无关的，只与pattern有关*，看下图更明显。

k可以完全由pattern得到，k只要**满足P[0,k-1] == P[j-k,j-1]即可**。

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic6.png" height="100">

那么我们完全可以根据pattern预处理生成一个next数组，next[j]的值（即k）表示，当P[j] != M[i]时，j指针的下一步移动位置。

由下图可以发现一个规律，当P[k]==P[j]时，next[j+1]==k+1

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic7.png" height="100">

那么当P[k]!=P[j]时呢？

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic8.png" height="100">

上图看不出规律，转换成下图这样就比较明朗了

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic9.png" height="100">

当就j!=k时，意味着没有ABAEABAH这样长的前缀，但我们可以退而求其次，在ABAEABAH中寻找最长的前缀。   
则问题可以转化为在搜索pattern ABAEABAH时，H和主串不一样了。   
因此这时k=next[k]，然后重复上述步骤，继续把现在的P[k]和P[j]比较，

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic10.png" height="100">

走到上图发现现在的P[k]和P[j]还是不相等，则继续重复上述步骤，此时k=next[next[k]]

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic11.png" height="100">

发现终于P[k]==P[j]了，因此我们可以得到next[j+1]==k+1（此时k=next[next[next[j]]])

总结一下：   
比较P[k]和P[j]，当P[k]！=P[j]时，令k=next[k]，继续比较P[k]和P[j]，一直重复上述步骤，直到P[k]==P[j]或者k不能再前移为止,则next[j+1]==k+1

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic12.png" height="100">

代码如下：
```
/* 
*  @brief 计算部分匹配表，即 next 数组.
*  @param[in] p 模式串 
*  @param[in] next 数组 
*  @return 无
*/ 
function compute_prefix (p, next) {
  next[0] = -1;  //注释1 
  let j = 0; 
  let k = -1; 
  while (j < p.length - 1) {
    if (k == -1 || p[j] == p[k]) {
      next[++j] = ++k; //注释2
    } else {
      k = next[k];
    }
  } 
}
```
**代码注释:**   
+ *注释1* 当pattern第一位就不匹配时，j无法左移应保持不变，只需移动i，所以初始化next[0]=-1

+ *注释2* 如下图，当j=1时,只能移动到0位置。而进入循环前初始化k=-1,进入循环首先就计算next[1]的值，所以当k==-1时next[++j] = ++k

    <img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic13.png" height="100">

到现在，我们基本完成了生成预处理数组的任务，上面的代码看上去很不错了，然而它还有一个小缺陷。   
如下图，当P[j]==P[k]时，之前的P[j]匹配不上，现在的P[k]肯定也匹配不上，所以这时的k要前移。

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/pic14.png" height="100">


所以在P[j]==P[next[j]]的特殊情况下，我们需要小小地修改一下代码
```
function compute_prefix (p, next) {
  next[0] = -1; let j = 0; let k = -1;
  while (j < p.length - 1) {
    if (k == -1 || p[j] == p[k]) {
      if (p[++j] == p[++k]) { 
        next[j] = next[k];
      } else {
        next[j] = k;
      }
    } else {
      k = next[k];
    }
  } 
}
```
完整的KMP算法如下
```
/* 
*  @brief KMP算法
*  @param[in] P pattern模式串 
*  @param[in] M master主串 
*  @return 无
*/
function KMP (M, P) {
  let i = 0; // 主串的位置
  let j = 0; // 模式串的位置
  let next = [];
  compute_prefix (P, next);
  while (i < M.length && j < P.length) {
    if (j == -1 || M[i] == P[j]) { // 当j为-1时，要移动的是i，当然j也要归0
      i++;
      j++;
    } else {
      j = next[j]; // i不需要回溯了,j回到指定位置
    }
  }
  if (j == p.length) {
    return i - j;
  } else {
    return -1;
  }
}
```
看了上述KMP算法分析，各位童鞋是否觉得KMP在字符串搜索算法算很不错呢，我以前也这样以为，直到后来看到Boyer-Moore算法才发现，啊~当时我还太天真，Boyer-Moore算法平均要比KMP快3-5倍，在实际的工业生产中，比如GNU grep,还有各种文本编辑器的查找功能（Ctrl+F）大多用的BM（Boyer-Moore）算法。

KMP算法利用的是pattern的前缀，而Robert S. Boyer教授和J Strother Moore教授发明的BM算法利用的则是pattern的后缀，关于BM算法的介绍，限于篇幅，本文就不详细介绍了，推荐阮一峰老师的博文，里面对于BM算法有详细的介绍[Boyer-Moore](http://www.ruanyifeng.com/blog/2013/05/boyer-moore_string_search_algorithm.html),基于BM算法还有Horspool算法，感兴趣的同鞋可以自行查阅资料

下图是朴素算法，KMP算法，简化BM算法，BM算法，Horspool算法关于不同长度pattern的性能比较

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/compare.png" height="200">

由图我们可以清楚发现，当pattern长度越长，BM算法的表现越好，当长度大于7时，BM简直是一骑绝尘，把KMP狠甩在后面

那么现在，在了解常用的字符串匹配算法后，我们一起来看看js中indexOf()是怎么实现的
v8引擎源码实现[源码地址看这里](https://github.com/v8/v8/blob/master/src/string-search.h)

在src/string-search.h文件中一共定义了五种搜索算法：
1. SingleCharSearch
2. LinearSearch
3. InitialSearch
4. BoyerMooreHorspoolSearch
5. BoyerMooreSearch

具体使用哪种，是在初始化StringSearch时根据pattern长度定义的

部分代码如下

```
StringSearch(Isolate* isolate, Vector<const PatternChar> pattern)
      : isolate_(isolate),
        pattern_(pattern),
        start_(Max(0, pattern.length() - kBMMaxShift)) {
    if (sizeof(PatternChar) > sizeof(SubjectChar)) {
      if (!IsOneByteString(pattern_)) {
        strategy_ = &FailSearch;
        return;
      }
    }
    int pattern_length = pattern_.length();
    if (pattern_length < kBMMinPatternLength) {
      if (pattern_length == 1) {
        strategy_ = &SingleCharSearch;
        return;
      }
      strategy_ = &LinearSearch;
      return;
    }
    strategy_ = &InitialSearch;
  }
```
v8是c++实现的，不写c++的同鞋看着可能不太习惯，没关系，上面的代码总结一下就是下图（设pattern长度为length，其中kBMMinPatternLength在源码中为7)

<img src="https://raw.githubusercontent.com/earEastYue/markdownPhotos/master/photos/KMPpic/indexOf.png" height="200">

可以看到indexOf()根据传入的pattern选择相应长度下效率最高的匹配算法，真真是少林武功，集各家之所长啊~

总之一句话indexOf()，你值得拥有~

