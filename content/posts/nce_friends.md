---
title: NCE 的朋友们
date: 2023-07-08 09:10:00 +0800
tags: [ML]
math: true
canonicalURL: "https://canonical.url/to/page"
---
在[Noise Contrastive Estimation](/posts/noise_contrastive_estimation/)中，我们详细介绍了 NCE 算法，其实还有很多跟它类似的算法，继续以文本生成为例，基于上下文`$\boldsymbol{c}$`，要去词表`$\mathcal{V}$`挑选`$\boldsymbol{w}$`来生成：

## Negative Sampling

其实要说 NCE 的缺点，就是需要花时间调参数，比如说`$k$`和噪声分布`$p_{n}$`的选择，而负采样则固定下来这两个参数，不同的负采样对于这两个参数的选择也不尽相同，这里介绍比较简单的一种

我们规定，当`$\mathcal{D}=1$`时代表从训练集采样，而`$\mathcal{D}=0$`则代表从噪声中采样，为了表达简便，对`$\boldsymbol{c}$`进行省略，即`$p(\mathcal{D}=1|\boldsymbol{c}, \boldsymbol{w}) = p(\mathcal{D}=1|\boldsymbol{w})$`

NCE 会这样求样本来自哪个采样的概率：

`$$
\begin{align}
p(\mathcal{D}=1|\boldsymbol{w})  & = \frac{p_{\boldsymbol{\theta }}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \\
p(\mathcal{D}=0|\boldsymbol{w})  & = \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}
\end{align}
$$`

而负采样说，如果词表大小是`$|\mathcal{V}|$`，不妨就采样`$|\mathcal{V}|$`个噪声，而且每个噪声是等概率出现的，那么：

`$$
\begin{align}
p(\mathcal{D}=1|\boldsymbol{w})  & = \frac{p_{\boldsymbol{\theta }}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+1} \\
p(\mathcal{D}=0|\boldsymbol{w})  & = \frac{1}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+1}
\end{align}
$$`

综合上面的式子来看，其实负采样是 NCE 的一种

## Importance Sampling

我们记需要计算的分母（配分函数）为`$Z(\boldsymbol{\theta })$`，注意：`$u_{\boldsymbol{\theta}}(\boldsymbol{w}, \boldsymbol{c})=\exp(s_{\boldsymbol{\theta}}(\boldsymbol{w}, \boldsymbol{c}))$`，`$s_{\boldsymbol{\theta}}$`为打分函数

`$$
p_{\boldsymbol{\theta }}(\boldsymbol{w}|\boldsymbol{c}) = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{\underbrace{ \sum_{\boldsymbol{w}' \in \mathcal{V}}u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c}) }_{ Z(\boldsymbol{\theta }) }}  = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{Z(\boldsymbol{\theta })}
$$`

那么：

`$$
\begin{align}
Z(\boldsymbol{\theta })  & = \sum_{\boldsymbol{w}' \in \mathcal{V}} u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c}) = \sum_{\boldsymbol{w}' \in \mathcal{V}} u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c}) {\color{#337dff}\frac{p_{n}(\boldsymbol{w}')}{p_{n}(\boldsymbol{w}')}} \\
&= \sum_{\boldsymbol{w}' \in \mathcal{V}} p_{n}(\boldsymbol{w}') \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c})}{p_{n}(\boldsymbol{w}')} \\
&= \mathbb{E}_{\boldsymbol{w}' \sim p_{n}} \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c})}{p_{n}(\boldsymbol{w}')}
\end{align}
$$`

跟 NCE 不同的是，重要性采样并没有把配分函数当成参数，而是利用噪声数据去拟合它：

`$$
p_{\boldsymbol{\theta }}(\boldsymbol{w}|\boldsymbol{c}) = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{Z(\boldsymbol{\theta })} = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{\mathbb{E}_{\boldsymbol{w}' \sim p_{n}} u_{\boldsymbol{\theta }}(\boldsymbol{w}', \boldsymbol{c})/p_{n}(\boldsymbol{w}')}
$$`

同样地操作，我们可以采样`$k$`个噪声样本来去近似期望，即：

`$$
p_{\boldsymbol{\theta }}(\boldsymbol{w}|\boldsymbol{c}) \approx \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{\sum_{i=1}^{k}p_{n}(\boldsymbol{w}_{i})u_{\boldsymbol{\theta }}(\boldsymbol{w}_{i}, \boldsymbol{c})/p_{n}(\boldsymbol{w}_{i})} = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{\sum_{i=1}^{k}u_{\boldsymbol{\theta }}(\boldsymbol{w_{i}}, \boldsymbol{c})}
$$`

仔细看，其实重要性采样是利用蒙特卡洛采样将分母从一开始的`$|\mathcal{V}|$`项降低到`$k$`项，另外还有[InfoNCE](/posts/infonce/)