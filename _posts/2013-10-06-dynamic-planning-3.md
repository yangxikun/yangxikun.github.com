---
layout: post
title: "动态规划--最大子段和问题与最大子矩阵和问题"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

>最大子段和:给定由n个整数组成的序列a1,a2,...,an,求该序列形如ai+...+aj的子段和的最大值.

>算法是基于数据结构的,定义了好的数据结构才能更好的求解问题.所以先定义数据结构:`b[j]=max{ai+...+aj}`其中`1<=i<=j<=n`,表示b[j]是以a[j]为结尾的子段所能达到的最大值.在求解出b[1],b[2],...,b[n]之后,从b[1...n]中求出一个最大的值就可以得到原问题的最优解.

###分析最优子结构

>上面已经定义了数据结构,接着就要证明其是否具有最优子结构性质,这里我们所证明的最优子结构是指以a[k]为结尾的子段所能达到的最大值,虽然与原问题的最优解不一样,但我们却可以从这个最优子结构中得出原问题的解,所以在动态规划算法中,最优子结构并不是依据原问题的最优解而设定,应该是根据定义的数据结构,但又得确保最后能归纳出原问题的最优解.反证法:

>当`b[j-1]>0`时,设X为b[j]的值,则X-a[j]是b[j-1]的值,即以a[j-1]为结尾的子段所能达到的最大值,否则b[j]的值将大于X.

>当`b[j-1]<=0`时,b[j] = a[j].

>b[j]的子问题就是a[1...j]的所有子段,显然由上面两种情况,可以得出b[j]确实包含了它的子问题的最优解.

###建立递归关系

>由最优子结构的分析可得:

>`b[j] = max{ai+...+aj} = max{b[j-1]+a[j], a[j]}`,进一步分析可得

>当`b[j-1]>0`时,`b[j] = a[j] + b[j-1]`;

>当`b[j-1]<=0`时,`b[j]=a[j]`.

###算法代码

{% highlight cpp linenos %}
int MaxSum(int n, int *a){
    int sum = 0, b = 0;
    for (int i = 1; i <= n; i++) {
        if (b > 0) b += a[i];//这段判断语句相当于
        else b = a[i];//max{b[j-1]+a[j], a[j]}
        if (b > sum) sum = b;
    }
    return sum;
}
{% endhighlight %}

>最大子矩阵和问题是最大子段和问题推广到高维的情形*\(对我来说真的有点难度...\)*

>给定一个m行n列的整数矩阵A,试求A的一个子矩阵,使其各元素之和为最大.

>定义数据结构:设子矩阵的左上角和右下角行列坐标分别为(i1,j1)和(i2,j2),则此子矩阵的和为

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mi>s</mi>
  <mo stretchy="false">(</mo>
  <mi>i</mi>
  <mn>1</mn>
  <mo>,</mo>
  <mi>i</mi>
  <mn>2</mn>
  <mo>,</mo>
  <mi>j</mi>
  <mn>1</mn>
  <mo>,</mo>
  <mi>j</mi>
  <mn>2</mn>
  <mo stretchy="false">)</mo>
  <mo>=</mo>
  <munderover>
    <mo>&#x2211;<!-- ∑ --></mo>
    <mrow class="MJX-TeXAtom-ORD">
      <mi>i</mi>
      <mn>1</mn>
    </mrow>
    <mrow class="MJX-TeXAtom-ORD">
      <mi>i</mi>
      <mn>2</mn>
    </mrow>
  </munderover>
  <mi>a</mi>
  <mo stretchy="false">[</mo>
  <mi>i</mi>
  <mo stretchy="false">]</mo>
  <mo stretchy="false">[</mo>
  <mi>j</mi>
  <mo stretchy="false">]</mo>
</math>

>最大子矩阵和问题的最优解为

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block"> <munder> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>m</mi> </mrow> </munder> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>m</mi> </mrow> </munder> <mi>s</mi> <mo stretchy="false">(</mo> <mi>i</mi> <mn>1</mn> <mo>,</mo> <mi>i</mi> <mn>2</mn> <mo>,</mo> <mi>j</mi> <mn>1</mn> <mo>,</mo> <mi>j</mi> <mn>2</mn> <mo stretchy="false">)</mo> </math>

>如果直接穷举的话,需要<math xmlns="http://www.w3.org/1998/Math/MathML" display="inline"> <mi>O</mi> <mo stretchy="false">(</mo> <msup> <mi>m</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msup> <msup> <mi>n</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msup> <mo stretchy="false">)</mo> </math>时间.

>对最优值公式进行变换:

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block"> <munder> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>m</mi> </mrow> </munder> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>n</mi> </mrow> </munder> <mi>s</mi> <mo stretchy="false">(</mo> <mi>i</mi> <mn>1</mn> <mo>,</mo> <mi>i</mi> <mn>2</mn> <mo>,</mo> <mi>j</mi> <mn>1</mn> <mo>,</mo> <mi>j</mi> <mn>2</mn> <mo stretchy="false">)</mo> <mo>=</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>m</mi> </mrow> </munder> <mo fence="false" stretchy="false">{</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>n</mi> </mrow> </munder> <mi>s</mi> <mo stretchy="false">(</mo> <mo stretchy="false">(</mo> <mi>i</mi> <mn>1</mn> <mo>,</mo> <mi>i</mi> <mn>2</mn> <mo>,</mo> <mi>j</mi> <mn>1</mn> <mo>,</mo> <mi>j</mi> <mn>2</mn> <mo stretchy="false">)</mo> <mo fence="false" stretchy="false">}</mo> <mo>=</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>i</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>m</mi> </mrow> </munder> <mi>t</mi> <mo stretchy="false">(</mo> <mi>i</mi> <mn>1</mn> <mo>,</mo> <mi>i</mi> <mn>2</mn> <mo stretchy="false">)</mo> </math>

>式中,<math xmlns="http://www.w3.org/1998/Math/MathML" display="block"> <mi>t</mi> <mo stretchy="false">(</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>,</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo stretchy="false">)</mo> <mo>=</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>&lt;=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo>&lt;=</mo> <mi>n</mi> </mrow> </munder> <mi>s</mi> <mo stretchy="false">(</mo> <mo stretchy="false">(</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>,</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo>,</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>,</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo stretchy="false">)</mo> <mo>=</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>1</mn> <mo>&lt;=</mo> <mi>j</mi> <mn>2</mn> <mo>&lt;=</mo> <mi>n</mi> </mrow> </munder> <munderover> <mo>&#x2211;<!-- ∑ --></mo> <mrow class="MJX-TeXAtom-ORD"> <mi>j</mi> <mo>=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> </mrow> <mrow class="MJX-TeXAtom-ORD"> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> </mrow> </munderover> <munderover> <mo>&#x2211;<!-- ∑ --></mo> <mrow class="MJX-TeXAtom-ORD"> <mi>i</mi> <mo>=</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> </mrow> <mrow class="MJX-TeXAtom-ORD"> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> </mrow> </munderover> <mi>a</mi> <mo stretchy="false">[</mo> <mi>i</mi> <mo stretchy="false">]</mo> <mo stretchy="false">[</mo> <mi>j</mi> <mo stretchy="false">]</mo> </math>

>设<math xmlns="http://www.w3.org/1998/Math/MathML" display="block"> <mi>b</mi> <mo stretchy="false">[</mo> <mi>j</mi> <mo stretchy="false">]</mo> <mo>=</mo> <munderover> <mo>&#x2211;<!-- ∑ --></mo> <mrow class="MJX-TeXAtom-ORD"> <mi>i</mi> <mo>=</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> </mrow> <mrow class="MJX-TeXAtom-ORD"> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> </mrow> </munderover> <mi>a</mi> <mo stretchy="false">[</mo> <mi>i</mi> <mo stretchy="false">]</mo> <mo stretchy="false">[</mo> <mi>j</mi> <mo stretchy="false">]</mo> </math>,则<math xmlns="http://www.w3.org/1998/Math/MathML" display="block"> <mi>t</mi> <mo stretchy="false">(</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>,</mo> <msub> <mi>i</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo stretchy="false">)</mo> <mo>=</mo> <munder> <mo movablelimits="true">max</mo> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> <mo>&lt;=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> <mo>&lt;=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> <mo>&lt;=</mo> <mi>n</mi> </mrow> </munder> <munderover> <mo>&#x2211;<!-- ∑ --></mo> <mrow class="MJX-TeXAtom-ORD"> <mi>j</mi> <mo>=</mo> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>1</mn> </mrow> </msub> </mrow> <mrow class="MJX-TeXAtom-ORD"> <msub> <mi>j</mi> <mrow class="MJX-TeXAtom-ORD"> <mn>2</mn> </mrow> </msub> </mrow> </munderover> <mi>b</mi> <mo stretchy="false">[</mo> <mi>j</mi> <mo stretchy="false">]</mo> </math>.

>可以看出,这是一维情形的最大子段和问题*\(想到这步,没有解题经验谈何容易...\)*,由此,借助于最大子段和问题的动态规划算法MaxSum,可设计出解最大子矩阵和问题的动态规划算法MaxSum2如下:

{% highlight cpp linenos %}
int MaxSum2(int m, int n, int **a) {
    int sum = 0;
    int *b = new int[n+1];
    for (int i = 1; i <= m; i++) {//相当于i1
        for (int k = 1; k <= n; k++) b[k] = 0;
        for (int j = i; j <= m; j++) {//相当于i2
            for (int k = 1; k <= n; k++) b[k] += a[j][k];//注意这里是累加,相当于公式b[j] = a[i1][j]+...+a[i2][j]
            int max = MaxSum(n, b);//max相当于得到一个t(i1,i2)的值
            if (max > sum) sum =max;//这一步相当于max t(i1, i2) 1<=i1<=i2<=m
        }
    }
    return sum;
}
{% endhighlight %}