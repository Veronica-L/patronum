---
title: "梯度下降法"
subtitle: "深度学习-数值计算"
layout: post
author: "Veronica"
header-style: text
tags:
  - 深度学习
---

参考链接：https://blog.csdn.net/qq_41800366/article/details/86583789



##### 梯度下降的目的

通过迭代找到目标函数的最小值，或者收敛到最小值

<img src="/Users/veronica/Downloads/20190121201301798.png" alt="20190121201301798" style="zoom:50%;" />

##### 梯度下降法

###### 梯度

梯度就是分别对每个变量进行微分，然后用逗号分割开，梯度是用<>包括起来，说明梯度其实一个向量。

$ J(\theta)=0.55-(5\theta_1+2\theta_2-12\theta_3) $

$ \bigtriangledown J(\theta)= <\frac{\partial{J}}{\partial{\theta_1}},\frac{\partial{J}}{\partial{\theta_2}},\frac{\partial{J}}{\partial{\theta_3}}> $



###### 梯度的意义

在单变量的函数中，梯度其实就是函数的微分，代表着函数在某个给定点的切线的斜率

**在多变量函数中，梯度是一个向量，向量有方向，梯度的方向就指出了函数在给定点的上升最快的方向**



###### 数学解释

数学公式：

$ \theta^1=\theta^0+\alpha\bigtriangledown J(\theta)\to $ evaluated at $\theta^0$

J是关于$\theta$的一个函数，我们当前所处的位置为$\theta^0$点，要从这个点走到J的最小值点，也就是山底。首先我们先确定前进的方向，也就是梯度的反向，然后走一段距离的步长，也就是$\alpha$，走完这个段步长，就到达了$\theta^1$这个点

**$\alpha$:**

学习率/步长。α不能太大也不能太小，太小的话，可能导致一直走不到最低点，太大的话，会导致错过最低点

**梯度乘一个负号：**

梯度前加一个负号，意味着朝着梯度相反的方向前进。梯度的方向实际就是函数在此点上升最快的方向，而我们要得到的是函数在此点下降最快的方向

