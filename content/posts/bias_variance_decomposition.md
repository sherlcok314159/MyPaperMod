---
title: Bias Variance Decomposition
date: 2023-06-21 18:10:00 +0800
tags: [ML]
math: true
canonicalURL: "https://canonical.url/to/page"
---
## 引言

我们规定，训练集记为`$\mathcal{D}$`，我们从中取一个样本`$\boldsymbol{x}$`，其训练集标签为`$y_{\mathcal{D}}$`，一般假设训练集是从一个真实的分布中采样而来，而真实分布不可见，算法通过训练集来近似整个分布。而采样的过程中存在噪声`$\epsilon$`，也就是说有可能因为噪声`$y_{\mathcal{D}}$`和真实标签的期望`$y$`不一致，举个例子，手写数字体识别数据集中存在不少标注错误，此时它们真实的标签和训练集中的就不一致。

`$$
y = \frac{1}{m} \sum_{i=1}^{m} y^{(i)}
$$`
这里为了讨论方便，不妨将噪声置为`$0$`，即：

`$$
\varepsilon = \mathbb{E}_{\mathcal{D}} (y - y_{\mathcal{D}}) = 0
$$`

我们还有个在`$\mathcal{D}$`上训练好的模型，其对于`$\boldsymbol{x}$`的预测记为`$f(\boldsymbol{x})$`，在整个训练集上预测的期望为：

`$$
\bar{f}(\boldsymbol{x}) = \mathbb{E}_{\mathcal{D}} f(\boldsymbol{x})
$$`

## 偏差-方差分解

那么，我们将期望损失函数进行拆解：

`$$
\begin{align}
\mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - y_{\mathcal{D}})^{2} &= \mathbb{E}_{\mathcal{D}} \left(f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}) - (y_{\mathcal{D}} - \bar{f}(\boldsymbol{x}) )\right)^{2} \\
&= \mathbb{E}_{\mathcal{D}} \left((f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}))^{2} + (y_{\mathcal{D}} - \bar{f}(\boldsymbol{x}) )^{2}-2(f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x})(y_{\mathcal{D}}-\bar{f}(\boldsymbol{x}))\right) \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}} (y_{\mathcal{D}} - \bar{f}(\boldsymbol{x}) )^{2} - 2\mathbb{E}_{\mathcal{D}}(f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))(y_{\mathcal{D}} - \bar{f}(\boldsymbol{x}))
\end{align}
$$`

对于式中最后一项，展开看看：

`$$
\begin{align}
\mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))(y_{\mathcal{D} }-\bar{f}(\boldsymbol{x})) &= \mathbb{E}_{\mathcal{D}} \left((f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))y_{\mathcal{D}} - (f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}))\bar{f}(\boldsymbol{x})\right) \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))y_{\mathcal{D}} - \mathbb{E}_{\mathcal{D}}(f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))\bar{f}(\boldsymbol{x}) \\
&=  \mathbb{E}_{\mathcal{D}} f(\boldsymbol{x}) y_{\mathcal{D}} - \bar{f}(\boldsymbol{x})\mathbb{E}_{\mathcal{D}}  y_{\mathcal{D}} - \bar{f}(\boldsymbol{x})\mathbb{E}_{\mathcal{D}} f(\boldsymbol{x}) + \bar{f}(\boldsymbol{x})^{2} \\
&= {\color{#337dff}\mathbb{E}_{\mathcal{D}}f(\boldsymbol{x}) \cdot\mathbb{E}_{\mathcal{D} } y_{\mathcal{D}}} - \bar{f}(\boldsymbol{x}) \mathbb{E}_{\mathcal{D}} y_{\mathcal{D}} - \bar{f}(\boldsymbol{x})^{2} + \bar{f}(\boldsymbol{x}) ^{ 2} \\
&= (\mathbb{E}_{\mathcal{D}}f(\boldsymbol{x})-\bar{f}(\boldsymbol{x})) \mathbb{E}_{\mathcal{D}} y_{\mathcal{D}} \\
&= 0
\end{align}
$$`

上述推导过程中，主要利用了两个信息：

1. `$\bar{f}(\boldsymbol{x})$`是个常量，因而可以直接提出来
2. 训练集本身的分布和模型预测的情况两者是相互独立的，因而就有了蓝色部分的分解

接下来我们还得继续把偏差凑出来，这里的`$y$`应该是指真实标签的期望：

`$$
\begin{align}
\mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - y_{\mathcal{D}})^{2} &= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}} (y_{\mathcal{D}} - \bar{f}(\boldsymbol{x}) )^{2} \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x})-\bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}} (y_{\mathcal{D}} - y + y - \bar{f}(\boldsymbol{x}))^{2}  \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}\left((y_{\mathcal{D}}-y)^{2}+ (y-\bar{f}(\boldsymbol{x}))^{2} + 2(y_{\mathcal{D}}-y)(y-\bar{f}(\boldsymbol{x}))\right) \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}(y_{\mathcal{D}}-y)^{2} + \mathbb{E}_{\mathcal{D}}(y-\bar{f}(\boldsymbol{x}))^{2} + 2\mathbb{E}_{\mathcal{D}}(y_{\mathcal{D}}-y)(y-\bar{f}(\boldsymbol{x}))  \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}(y_{\mathcal{D}}-y)^{2} + \mathbb{E}_{\mathcal{D}}(y-\bar{f}(\boldsymbol{x}))^{2} + 2({\color{#337dff}y-\bar{f}(\boldsymbol{x})})\underbrace{ \mathbb{E}_{\mathcal{D}}(y_{\mathcal{D}}-y) }_{\varepsilon=0 }   \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}(y_{\mathcal{D}}-y)^{2} + \mathbb{E}_{\mathcal{D}}(y-\bar{f}(\boldsymbol{x}))^{2} \\
&= \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}(y_{D} - y)^{2} + (y - \bar{f}(\boldsymbol{x}))^{2}
\end{align}
$$`

那么：

`$$
\begin{align}
\mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - y_{\mathcal{D}})^{2}  & = \mathbb{E}_{\mathcal{D}} (f(\boldsymbol{x}) - \bar{f}(\boldsymbol{x}))^{2} + (y - \bar{f}(\boldsymbol{x}))^{2} + \mathbb{E}_{\mathcal{D}}(y_{D} - y)^{2} \\
&= \mathbb{V}(\boldsymbol{x}) + bias^{2}(\boldsymbol{x}) + \varepsilon^{2}
\end{align}
$$`

## 泛化能力

- 偏差衡量了算法本身对于真实标签的拟合情况，the deviation of the expected estimator value from the true value
- 方差衡量了在不同训练集`$\mathcal{D}$`情况下，即采样不同的训练集跟期望预测的偏差，也就是受扰动性影响，the deviation from the expected estimator value that any particular sampling of data is likely to cause（注意`$f(\boldsymbol{x})$`是随着训练集而变化的，而`$\bar{f}(\boldsymbol{x})$`则是取一个任意训练集然后固定）
- 噪声则是衡量了期望误差的下界，也就是说，无论算法怎样，这个任务难度的初始值就是这样