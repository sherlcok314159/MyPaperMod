---
title: Fast Greedy MAP Inference for DPP
date: 2023-05-16 10:20:00 +0800
tags: [ML]
math: true
canonicalURL: "https://canonical.url/to/page"
---
## 问题

先规定一些术语：记选中元素构成的集合为`$\mathcal{S}$`，未选中构成的元素记为`$\mathcal{R}$`，`$\mathbf{L}$`是核矩阵（核函数是内积），`$\mathbf{L_{V}}$`是由集合`$S$`的元素构成的子矩阵

在 [Determinatal Point Process](/posts/determinantal_point_process/) 中我们提到在大小为`$n$`的集合里去挑选`$k$`个物品构成集合`$S$`, 使得`$\det(\mathbf{L}_{\mathbf{V}})$`最大便是我们的目标，然而，怎么去里面挑选`$\mathbf{V}$`却是 NP-Hard 问题，为此，Chen et al., 2018 提出了一篇比较巧妙的贪婪算法作为近似解，并且整个算法的复杂仅有`$\mathcal{O}(nk^{2})$`

## 暴力求解

我们人为规定了要选择`$k$`个，这相当于是一种前验分布，那么 k-DPP 其实就是最大化后验概率（MAP）的一种，每一步的目标就是选择会让新矩阵的行列式变得最大的元素

`$$
j = \mathop{ \arg \max}_{i \in \mathcal{R}} \log \det(\mathbf{L}_{\mathcal{S} \cup \{i\}}) - \log \det(\mathbf{L}_{\mathcal{S}})
$$`

对于一个`$n\times n$`的方阵而言，求它的行列式需要`$\mathcal{O}(n^{3})$`（每一轮消元的复杂度是`$\mathcal{O}(n^{2})$`，而要进行`$n-1$`轮消元）

这里的话，每次要对`$\mathcal{R}$`所有的元素求一次行列式，而行列式的为`$\mathcal{O}(|\mathcal{S}|^{3})$`，同时需要选`$k$`个，复杂度变为了`$\mathcal{O}(|\mathcal{S}|^{3} \cdot |\mathcal{R}| \cdot k)$`，即为`$\mathcal{O}(nk^{4})$`，暴力求解的话复杂度很大，此时原作者便提出了利用 Cholesky 分解的方式来进行求解，巧妙地将复杂度降到了`$\mathcal{O}(nk^{2})$`

## Cholesky 分解

`$\mathbf{L}_{\mathcal{S}}$`是对称半正定矩阵，证明如下：`$\forall \boldsymbol{z} \in \mathbb{R}^{n}$`

`$$
\boldsymbol{z}^{\top}\mathbf{L}_{\mathcal{S}}\boldsymbol{z}   = \boldsymbol{z}^{\top} \mathbf{V}^{\top}\mathbf{V} \boldsymbol{z} = \|\mathbf{V}\boldsymbol{z}\|_{2}^{2} \geq 0 \quad \blacksquare
$$`

那么`$\mathbf{L}_{\mathcal{S}}$`存在 Cholesky 分解，即`$\mathbf{L}_{\mathcal{S}}=\mathbf{U}\mathbf{U}^{\top}$`，这里的`$\mathbf{U}$`是大小为`$|\mathcal{S}|\times|\mathcal{S}|$`的下三角矩阵，`$\mathbf{L}_{\mathcal{S}\cup \{ i \}}$`比`$\mathbf{L}_{\mathcal{S}}$`多了一行和一列，即为：

`$$
\mathbf{L}_{\mathcal{S}\cup \{ i \}} = \begin{bmatrix}
\mathbf{L}_{\mathcal{S} } & \boldsymbol{u}_{i} \\
\boldsymbol{u}_{i}^{\top} & \boldsymbol{u}_{i}^{\top}\boldsymbol{u}
\end{bmatrix}
$$`

而这里默认每个向量是经过归一化的，即`$\boldsymbol{u}_{i}^{\top}\boldsymbol{u}=1$`，那么`$\mathbf{L}_{\mathcal{S}\cup \{ i \}}$`的 Cholesky 分解即为下式，其中`$\boldsymbol{c}_{i} \in \mathbb{R}^{1 \times |\mathcal{S}| }, d_{i} \geq 0$`：

`$$
\mathbf{L}_{\mathcal{S}\cup \{ i \}} = \begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{i} & d_{i}
\end{bmatrix}\begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{i} & d_{i}
\end{bmatrix}^{\top}
$$`

结合上面两式：

`$$
\mathbf{L}_{\mathcal{S}\cup \{ i \}} = \begin{bmatrix}
\mathbf{L}_{\mathcal{S} } & \boldsymbol{u}_{i} \\
\boldsymbol{u}_{i}^{\top} & \boldsymbol{u}_{i}^{\top}\boldsymbol{u}
\end{bmatrix} = \begin{bmatrix}
\mathbf{U}\mathbf{U}^{\top} & \mathbf{U}\boldsymbol{c}_{i}^{\top} \\
\boldsymbol{c}_{i}\mathbf{U}^{\top} & \boldsymbol{c}_{i}\boldsymbol{c}_{i}^{\top} + d_{i}^{2}
\end{bmatrix}
$$`

可得：

`$$
\boldsymbol{u}_{i} = \mathbf{U}\boldsymbol{c}_{i}^{\top}, 1 = \boldsymbol{c}_{i}\boldsymbol{c}_{i}^{\top} + d_{i}^{2}
$$`

那么：

`$$
\det(\mathbf{L}_{\mathcal{S}\cup \{ i \}}) = \det \bigg(\begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{i} & d_{i}
\end{bmatrix}\begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{i} & d_{i}
\end{bmatrix}^{\top}\bigg) = \det(\mathbf{U}\mathbf{U}^{\top})\cdot d_{i}^{2}= \det(\mathbf{L}_{\mathcal{S}}) \cdot d_{i}^{2}
$$`

这样我们一开始的优化目标就可以简化为：

`$$
j = \mathop{\arg \max}_{i \in \mathcal{R}} \log(d_{i}^{2})
$$`

接下来，当我们得到`$d_j$`时，便可以算出`$c_{j}$`，那么添加`$j$`之后的新集合`$\mathcal{S}'$`的 Cholesky 分解便可以求得：

`$$
\mathbf{L}_{\mathcal{S}'} = \begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{j} & d_{j}
\end{bmatrix}\begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{j} & d_{j}
\end{bmatrix}^{\top}
$$`

## 增量更新

接下来便是重头戏，这一轮我们已经得到了最好的`$d_{j}$`了，下一轮我们怎么求出最大的`$d_{i}$`呢？

可以利用之前求出的`$c_{i}, d_{i}$`来获取当前的`$c_{i}', d_{i}'$`，这便是论文的核心：增量更新

在我们选择了`$j$`后，`$c_{i}$`多了一个元素，不妨记`$\boldsymbol{c}_{i}' = [\boldsymbol{c}_{i}, {e}_{i}]$`，回忆上面的式子：

`$$
\mathbf{L}_{\mathcal{S}\cup \{ i \}} = \begin{bmatrix}
\mathbf{L}_{\mathcal{S} } & \boldsymbol{u}_{i} \\
\boldsymbol{u}_{i}^{\top} & \boldsymbol{u}_{i}^{\top}\boldsymbol{u}
\end{bmatrix} = \begin{bmatrix}
\mathbf{U}\mathbf{U}^{\top} & \mathbf{U}\boldsymbol{c}_{i}^{\top} \\
\boldsymbol{c}_{i}\mathbf{U}^{\top} & \boldsymbol{c}_{i}\boldsymbol{c}_{i}^{\top} + d_{i}^{2}
\end{bmatrix}
$$`

也就是说，其中`$\boldsymbol{u}_{i}$`就是第`$i$`个元素与集合`$\mathcal{S}$`对应向量做内积的结果

`$$
\boldsymbol{u}_{i} = \mathbf{U}\boldsymbol{c}_{i}^{\top}
$$`

那么，类比一下，`$\mathbf{U}_{j}$`是`$\mathbf{L}_{\mathcal{S}'}$`的 Cholesky 分解

`$$
\underbrace{\begin{bmatrix}
\mathbf{U} & \boldsymbol{0} \\
\boldsymbol{c}_{j} & d_{j}
\end{bmatrix}}_{\mathbf{U}_{j}} \boldsymbol{c}_{i}'^{\top} = \boldsymbol{u}_{i}' = \begin{bmatrix}
\boldsymbol{u}_{i} \\
\mathbf{L}_{ji}
\end{bmatrix}
$$`

继而：

`$$
\langle \boldsymbol{c}_{j},\boldsymbol{c} _{i}\rangle  + d_{j}e_{i} = \mathbf{L}_{ji} \implies e_{i} = \frac{\mathbf{L}_{ji}-\langle \boldsymbol{c}_{j},\boldsymbol{c}_{i} \rangle }{d_{j}}
$$`

求出`$e_{i}$`之后，我们便可以求出`$d_{i}'$`：

`$$
d_{i}'^{2} = 1 - \|\boldsymbol{c}_{i}'\|_{2}^{2} =\underbrace{  1- \|\boldsymbol{c}_{i}\|_{2}^{2} }_{ {\color{#337dff}d_{i}^{2} }} - e_{i}^{2} = d_{i}^{2} - e_{i}^{2}
$$`

## 流程

{{< figure src="https://png.yunpengtai.top/2024/11/c49eb805d3b1e13f0a39e6a7540d6a9d.png" caption="算法整体流程" align="center" width=580px height=260px >}}

那我们来分析一下复杂度，每选一个`$j$`需要进行`$|\mathcal{R}| \cdot |\mathcal{S}|$`次操作，而`$|\mathcal{R}|\leq n, |\mathcal{S}| \leq k$`，也就是`$\mathcal{O}(nk)$`，得进行`$k$`次迭代，那么总的复杂度即为`$\mathcal{O}(nk^{2})$`，由`$\mathcal{O}(nk^{4})$`降到`$\mathcal{O}(nk^{2})$`，是不错的进步

## 代码

熟悉了整个流程之后，代码想必也是呼之欲出了

```python
import math
import numpy as np

def fast_map_dpp(kernel_matrix, max_length):
    cis = np.zeros((max_length, kernel_matrix.shape[0]))
    di2s = np.copy(np.diag(kernel_matrix))
    selected = np.argmax(di2s)
    selected_items = [selected]
    while len(selected_items) < max_length:
        idx = len(selected_items) - 1
        ci_optimal = cis[:idx, selected]
        di_optimal = math.sqrt(di2s[selected])
        elements = kernel_matrix[selected, :]
        eis = (elements - ci_optimal @ cis[:idx, :]) / di_optimal
        cis[idx, :] = eis
        di2s -= np.square(eis)
        di2s[selected] = -np.inf
        selected = np.argmax(di2s)
        selected_items.append(selected)
    return selected_items
```

这里实现比较有趣的点就是，尽管伪代码中是`$i \in \mathcal{R}$`，这里其实是全部算了，但他对已选的进行了后处理，置之为`$-\infty$`

接下来我们实操一下，从句子对匹配 BQ Corpus（Bank Question Corpus）拿出一条来看一下效果，首先是将其用预训练模型转换为表征向量，接着进行归一化操作，为了更好地看出 DPP 的效果，我们先用最大化内积来召回 50 个样本，再用 DPP 从这里召回 10 个具有多样性的样本：

```
原句：我现在申请微粒货？

['我现在申请微粒货？', '申请微贷粒', '申请微贷粒', '我想申请微粒贷', '可以么想申请微粒贷',
 '微粒貸申请', '微粒貸申请', '如何申请微粒', '我现在需要申请', '我可以申请微粒贷吗',
 '怎么申请微粒货', '微粒貸申请', '如何申请微粒', '我可以申请微粒贷吗', '什么情况下才能申请微粒',
 '我要求申请', '开通微粒货', '开通微粒貨', '开通微粒货', '可以申请开通吗', '开通微粒货',
 '开通微粒货', '怎么申请微粒货', '申请贷款', '如何申请微粒贷', '怎么申请微粒货', '开通微粒貨',
 '如何申请微粒', '想办理微粒贷业务', '申请贷款', '可以申请开通吗', '我要微粒贷', '我要微粒贷',
 '可以么想申请微粒贷', '开通微米粒', '想开通', '我要微粒贷', '如何申请微粒', '想开通', '开通微粒貨',
 '开通粒微贷', '何时才能申请啊', '现在想获取资格', '怎么申请微粒货', '开通申请', '开通', '开通',
 '你好我申请借款', '开通微']
```

可以看到有不少重复且意思一样的样本：

接着看 DPP 的效果：

```
['我现在申请微粒货？', '开通', '何时才能申请啊', '怎么申请微粒货', '你好我申请借款', 
 '现在想获取资格', '我可以申请微粒贷吗', '我要微粒贷', '微粒貸申请', '什么情况下才能申请微粒']
```

可以发现里面没有重复的情况，而且语义具备多样性，而值得注意的是，此时就有一些和我们的原句意思不匹配的情况，在应用时可以自定义新的 kernel，让它同时注意相似性和多样性；或者可以对 DPP 的样本进行后处理等

## Sliding Window

当`$|\mathcal{S}|$`相当大的时候，就会有相似的样本开始出现，即超平行体会开始坍缩，不妨我们将`$\mathcal{S}$`缩小成一个滑动窗口`$\mathcal{W}$`，我们仅仅需要保证窗口内的样本具备多样性即可，即：

`$$
j = \mathop{\arg \max}_{i \in \mathcal{R}} \log \det(\mathbf{L}_{\mathcal{W} \cup \{ i \}}) - \log \det(\mathbf{L}_{\mathcal{W}})
$$`

推荐系统中有短序推荐（Short Sequence Recommendation）的说法，推荐的时候只考虑用户短期内的一些行为，而长序推荐会考虑一个较长时间跨度来进行推荐

Window size 的选择也是比较重要的，不妨看一些 demo：

| 窗口 | \#不同样本 | \#重复样本 |
| :--: | :--------: | :--------------: |
|  2   |     2      |        0         |
|  3   |     2      |        1         |
|  4   |     1      |        2         |
|  5   |     1      |        1         |
|  7   |     1      |        0         |
|  9   |     0      |        0         |

如果我们的目的是为了通过 Sliding Window 获取与直接 DPP 召回不一样的结果，窗口的大小要适当地小一些，然而小了导致看的范围小了，很有可能最后结果出现重复的情况，最好是将窗口设置到召回样本数目的`$20\% \sim 30\%$`

同时，为了防止样本重复，可以多召回一些，比较直觉的做法可以再加上一个 window 的大小，然后去重：

```
w/o window
['开通', '怎么申请微粒货', '何时才能申请啊', '现在想获取资格', '我要微粒贷', '微粒貸申请', 
 '我可以申请微粒贷吗', '什么情况下才能申请微粒', '你好我申请借款', '我现在申请微粒货？']
```

```
w/ window
['开通', '开通申请', '怎么申请微粒货', '何时才能申请啊', '现在想获取资格', '我要微粒贷', 
 '我可以申请微粒贷吗', '可以申请开通吗', '如何申请微粒', '我现在申请微粒货？']
```

可以看到会有 3 个不一样的样本，还是比较有效的
