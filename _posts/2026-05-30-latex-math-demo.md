---
layout: post
title: 'LaTeX 公式渲染演示'
subtitle: 'MathJax · 暗色模式 · 公式灯箱'
date: 2026-05-30 10:00:00 +0800
author: 松鼠探长
tags:
- LaTeX
- MathJax
- 博客
---

&emsp;&emsp;这篇用于验证主题新增的 LaTeX 渲染能力：行内公式、独立公式块、暗色模式配色，以及点击公式后的灯箱（复制 LaTeX 源码 / 保存 PNG / 保存 SVG）。

# 行内公式

&emsp;&emsp;质能方程为 $E = mc^2$，这是最著名的物理公式之一。梯度下降的更新规则是 $\theta_{t+1} = \theta_t - \eta \nabla_\theta J(\theta)$，其中 $\eta$ 是学习率。

&emsp;&emsp;也可以使用转义写法：\(a^2 + b^2 = c^2\)，效果与 $a^2+b^2=c^2$ 一致。

# 独立公式块

&emsp;&emsp;下面用 `$$ ... $$` 包裹：

$$
\frac{\partial J}{\partial w_{jk}^{(l)}} = \delta_j^{(l)} \, a_k^{(l-1)}
$$

&emsp;&emsp;Softmax 与交叉熵：

$$
\hat{y}_i = \frac{e^{z_i}}{\sum_{j=1}^{K} e^{z_j}},
\qquad
L = -\sum_{i=1}^{K} y_i \log \hat{y}_i
$$

&emsp;&emsp;多行对齐环境：

$$
\begin{aligned}
z^{(l)} &= W^{(l)} a^{(l-1)} + b^{(l)} \\
a^{(l)} &= \sigma\!\left(z^{(l)}\right) \\
\sigma(x) &= \frac{1}{1 + e^{-x}}
\end{aligned}
$$

&emsp;&emsp;矩阵、求和与积分：

$$
\mathbf{W} =
\begin{pmatrix}
w_{11} & w_{12} & \cdots & w_{1n} \\
w_{21} & w_{22} & \cdots & w_{2n} \\
\vdots & \vdots & \ddots & \vdots \\
w_{m1} & w_{m2} & \cdots & w_{mn}
\end{pmatrix},
\qquad
\int_{-\infty}^{+\infty} e^{-x^2}\,\mathrm{d}x = \sqrt{\pi}
$$

&emsp;&emsp;卷积神经网络里的卷积运算：

$$
(f * g)(t) = \int_{-\infty}^{\infty} f(\tau)\, g(t - \tau)\,\mathrm{d}\tau
$$

&emsp;&emsp;代码块里的 `$` 不会被渲染：

```python
# 这里的 $x$ 是普通字符
cost = -(y * np.log(y_hat)).sum() / len(y)
learning_rate = 0.01  # $\eta$
```

# 灯箱验证要点

1. 明亮模式下点击公式，检查放大后的公式是否清晰、底色是否为白色；
2. 切换到暗色模式，检查正文公式是否为浅色（不再与背景糊在一起）；
3. 点击「复制 LaTeX」，粘贴到文本编辑器检查源码是否完整；
4. 点击「保存 SVG」「保存 PNG」，打开下载的文件检查是否为白底黑字、矢量清晰；
5. 点击下面这张透明背景的 SVG 公式图，检查 Fancybox 灯箱是否为白底（暗色模式下尤其明显）。

![透明背景 SVG 公式图，用于验证灯箱白底](/assets/img/softmax.svg)
