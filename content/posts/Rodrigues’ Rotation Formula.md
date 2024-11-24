---
tags:
  - 数学
title: Rodrigues’ Rotation Formula
date: 2022-12-17
---

关于三维空间中的旋转，我们以前提到过基于欧拉角的旋转表达矩阵，它们分别描述了围绕 x 轴、y 轴、z 轴旋转后坐标应当如何变化。事实上，我们可以更进一步，推导出一个通用的、围绕过原点的任意轴旋转的公式。

## 题设

这一节我们来描述我们已知的条件和待求的目标：

给定一个方向向量$\vec{u}$作为旋转轴，$\vec{v}$为待旋转的向量，我们希望得到$\vec{v}$围绕着$\vec{u}$逆时针旋转$\theta$角度之后的向量$\vec{v'}$。注意，$\vec{v'}$的表达式必须用已知条件$\vec{u}$、$\vec{v}$和$\theta$来表达。

![](<Rodrigues’ Rotation Formula/rodrigues_r_start.png>)

## 思路

旋转前后向量的长度不会变化，反而是如何计算旋转后向量的方向成为了一个难题。我们采用分解向量的思路来解决向量旋转的问题——把$\vec{v}$分解为平行于$\vec{u}$的向量$\vec{v_{//}}$和垂直于$\vec{u}$的向量$\vec{v_\bot}$，分解的示意图如下：

![](<Rodrigues’ Rotation Formula/rotation_diagram.png>)

这样定义向量的分解方式是有好处的，我们可以发现，旋转前后$\vec{v_{//}}$没有变化，而旋转前的$\vec{v_\bot}$、旋转后的$\vec{v'_\bot}$都在一个平面内，同一个平面内的旋转比起三维旋转要好解决的多。

## 公式推导

$\vec{v_{//}}$实际上是$\vec{v}$在$\vec{u}$上的正交投影，因此我们可以得出：
$$
\begin{align}
\vec{v_{//}}&=|\vec{v}|\frac{\vec{u}\cdot\vec{v}}{|\vec{u}|\cdot|\vec{v}|}\cdot\frac{\vec{u}}{|\vec{u}|}\\
&=\frac{\vec{u}\cdot\vec{v}}{|\vec{u}|^2}\vec{u}\\
&=(\vec{u}\cdot{\vec{v}})\vec{u}
\end{align}
$$
然后，我们运用向量减法给出$\vec{v_{\bot}}$的表达式：
$$
\vec{v_{\bot}}=\vec{v}-\vec{v_{//}}
$$
接下来，我们需要给出$\vec{v'_\bot}$的表达式。给出一个直观的旋转俯视图：

![](<Rodrigues’ Rotation Formula/rotate_top_view.png>)

先定义$\vec{w}$。引入这个向量是为了正交分解$\vec{v'_\bot}$，因此它必须垂直于$\vec{v_{\bot}}$。如果你对叉乘很熟悉的话，很快就能想到我们可以用$\vec{u}\times\vec{v}$来得到具有这样的性质的向量。注意叉乘的顺序，根据旋转示意图和右手定则，$\vec{u}\times\vec{v_{\bot}}$的向量方向才是俯视图中的$\vec{w}$方向。

然后，我们把$\vec{v'_\bot}$正交分解成平行于$\vec{v_{\bot}}$的$\vec{v'_{v}}$和平行于$\vec{w}$的$\vec{v'_w}$，并给出$\vec{v'_\bot}$的表达式。
$$
\displaylines{
\vec{v'_\bot}=\vec{v'_{v}}+\vec{v'_w}\\
\vec{v'_{v}}=|\vec{v_\bot}|\cos\theta\cdot\frac{\vec{v_\bot}}{|\vec{v_\bot}|}=\vec{v_\bot}\cos\theta\\
\vec{v'_w}=|\vec{v_\bot}|\sin\theta\cdot\frac{\vec{u}\times\vec{v_{\bot}}}{|\vec{u}\times\vec{v_{\bot}}|}\\
|\vec{u}\times\vec{v_{\bot}}|=|\vec{u}||\vec{v_{\bot}}|\sin(\pi/2)=|\vec{v_{\bot}}|\\
\vec{v'_w}=(\vec{u}\times\vec{v_{\bot}})\sin\theta\\
\vec{v'_\bot}=\vec{v_\bot}\cos\theta+(\vec{u}\times\vec{v_{\bot}})\sin\theta
}
$$
上面的式子可以进一步被化简，我们代入$\vec{v_{\bot}}$的表达式，并运用叉乘的分配律：
$$
\displaylines{
\vec{v'_\bot}=(\vec{v}-\vec{v_{//}})\cos\theta+(\vec{u}\times(\vec{v}-\vec{v_{//}}))\sin\theta\\
使用分配律，(\vec{u}\times(\vec{v}-\vec{v_{//}}))=\vec{u}\times\vec{v}-\vec{u}\times\vec{v_{//}}\\
因为共线，\vec{u}\times\vec{v_{//}}=0\\
\vec{v'_\bot}=\vec{v}\cos\theta-\vec{v_{//}}\cos\theta+(\vec{u}\times\vec{v})\sin\theta
}
$$
最后，我们终于可以给出$\vec{v'}$的表达式：
$$
\begin{aligned}
\vec{v'}&=\vec{v_{//}}+\vec{v'_{\bot}}\\
&=\vec{v_{//}}+\vec{v}\cos\theta-\vec{v_{//}}\cos\theta+(\vec{u}\times\vec{v})\sin\theta\\
&=\vec{v}\cos\theta+(1-\cos\theta)\vec{v_{//}}+(\vec{u}\times\vec{v})\sin\theta\\
&=\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}+(\vec{u}\times\vec{v})\sin\theta
\end{aligned}
$$
因此：
$$
\vec{v'}=\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}+(\vec{u}\times\vec{v})\sin\theta \tag{1}
$$
公式（1）即为标题中的 Rodrigues’ Rotation Formula。

## 化简为矩阵

我们需要进一步化简公式，得到其等价的矩阵表达形式，才方便代码的实现。

首先，我们需要知道向量三重积公式：
$$
\vec{a}\times(\vec{b}\times\vec{c})=(\vec{a}\cdot\vec{c})\cdot\vec{b}-(\vec{a}\cdot\vec{b})\cdot\vec{c} \tag{2}
$$
然后，我们还需要知道，向量的叉乘与矩阵之间的联系：
$$
\vec{a}\times\vec{b}=\begin{pmatrix}y_az_b-z_ay_b\\z_ax_b-x_az_b\\x_ay_b-y_ax_b\end{pmatrix}=A\cdot b=
\begin{pmatrix}
0 & -z_a & y_a\\
z_a& 0 & -x_a\\
-y_a& x_a & 0
\end{pmatrix}\begin{pmatrix}x_b\\y_b\\z_b\end{pmatrix} \tag{3}
$$
向量的叉乘可以化为矩阵与向量的乘积，而且需要注意的是，矩阵只与左边的向量有关。

--------------------

我们化简的目标是得到下述形式的表达式，其中M是矩阵。
$$
\vec{v'}=M\cdot\vec{v}
$$
再次观察公式（1）：
$$
\vec{v'}=\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}+(\vec{u}\times\vec{v})\sin\theta \tag{1}
$$
对于第一项和第三项来说，它们都可以快速化为等价的矩阵形式，其中，$\vec{v}\cos\theta$可以化为：
$$
\vec{v}\cos\theta=\begin{pmatrix}
\cos\theta & 0 & 0\\
0 & \cos\theta & 0\\
0 & 0 & \cos\theta
\end{pmatrix}\cdot\vec{v}
$$
正如前文所说，$(\vec{u}\times\vec{v})\sin\theta$可以利用公式（3）将叉乘化成矩阵乘向量的形式，这里记向量$\vec{u}$形成的矩阵为$R_u$，可以得到：
$$
(\vec{u}\times\vec{v})\sin\theta=R_u\sin\theta\cdot\vec{v} \tag{4}
$$
比较难以化简的是第二项，处于外部的是向量$\vec{u}$而不是$\vec{v}$，这给我们带来了一些麻烦。观察$(\vec{u}\cdot{\vec{v}})\vec{u}$这一项和已知的三重积公式（2），或许可以想办法配凑另外一项，从而把点乘变为叉乘，再运用叉乘的性质化作矩阵。有好几种可能的叉乘式，最终我们选择配凑出这样的叉乘：$\vec{u}\times(\vec{u}\times\vec{v})$。其中一个理由是$\vec{u}$都在左边，我们可以复用前面提到的矩阵$R_u$；另一个理由是缺失的那一项很好配凑：
$$
\vec{u}\times(\vec{u}\times\vec{v})=(\vec{u}\cdot\vec{v})\vec{u}-(\vec{u}\cdot\vec{u})\vec{v}=(\vec{u}\cdot\vec{v})\vec{u}-\vec{v}
$$
我们已经有了$(\vec{u}\cdot{\vec{v}})\vec{u}$这一项，而且它的系数是$(1-\cos\theta)$，那么接下来就要配凑出$-(1-\cos\theta)\vec{v}$这一项了。这很简单，我们结合公式（1）的第一项 $\vec{v}\cos\theta$：
$$
\begin{aligned}
\vec{v}\cos\theta
&=\vec{v}\cos\theta+\vec{v}-\vec{v}\\
&=\vec{v}-(1-\cos\theta)\vec{v}
\end{aligned}
$$
因此，我们可以将公式（1）的前两项$\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}$化简为：
$$
\begin{aligned}
\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}
&=\vec{v}-(1-\cos\theta)\vec{v}+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}\\
&=\vec{v}+(1-\cos\theta)((\vec{u}\cdot{\vec{v}})\vec{u}-\vec{v})\\
&=\vec{v}+(1-\cos\theta)(\vec{u}\times(\vec{u}\times\vec{v}))
\end{aligned}
$$
最终我们如愿以偿得到了两重叉乘：$\vec{u}\times(\vec{u}\times\vec{v})$。不过，两重叉乘也可以运用公式（3）吗？当然可以，运用两遍即可。而且，由于矩阵内的所有参数只和叉乘式的左边有关，因此我们会得到两个相同的矩阵：
$$
\begin{aligned}
\vec{u}\times(\vec{u}\times\vec{v})
&=\vec{u}\times(R_u\cdot\vec{v})\\
&=R_u\cdot (R_u\cdot\vec{v})\\
&=R_u^2\cdot\vec{v}
\end{aligned}
$$
我们得以将公式（1）的前两项化作矩阵向量乘法：
$$
\vec{v}\cos\theta+(1-\cos\theta)(\vec{u}\cdot{\vec{v}})\vec{u}=\vec{v}+(1-\cos\theta)\cdot R_u^2\cdot\vec{v} \tag{5}
$$
最后，结合（4）与（5），我们将公式（1）化简为：
$$
\displaylines{
\vec{v'}=\vec{v}+(1-\cos\theta)\cdot R_u^2\cdot\vec{v}+R_u\sin\theta\cdot\vec{v}\\
记I为单位矩阵，则\\
\vec{v'}=(I+(1-\cos\theta)\cdot R_u^2+R_u\sin\theta)\cdot\vec{v}\\
}
$$
矩阵
$$
M=I+(1-\cos\theta)\cdot R_u^2+R_u\sin\theta \tag{6}
$$
即为我们所求的矩阵。

## Refer

[罗德里格旋转公式（Rodrigues' rotation formula）](https://zhuanlan.zhihu.com/p/451579313)

[四元数与三维旋转](https://github.com/Krasjet/quaternion)