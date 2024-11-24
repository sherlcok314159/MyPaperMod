---
title: Numerical Stability
date: 2023-06-25 21:10:00 +0800
tags: [training]
math: true
canonicalURL: "https://canonical.url/to/page"
---
## Why

当计算涉及到实数域时，比如圆周率的`$\pi$`，因为小数部分是无穷的，计算机是无法准确表示，因而只会用近似的值进行替代，这种情况下，误差相对较小影响不大；然而，如果数值大于某个特定值之后变成了`$\inf$`（数值上溢 Overflow ），还有一种情况是当数值特别小时，会被近似为`$0$`（数值下溢 Underflow），这两种情形若继续计算误差将会进一步累积，那么就可能导致原本理论上成立的到实现时就不行了，此时就有必要对数值稳定性进行分析

举个例子：

```python
import numpy

a = np.array([65599.], dtype=np.float16)
print(a)
b = np.array([1e-10], dtype=np.float16)
print(b)

[inf] # Overflow
[0.] # Underflow
```


## 基础知识

再进一步解释解决方法时还是先铺垫基础知识：

众所周知，各种数值在计算机底层是通过比特（bit）来进行表示的，有`$0$`和`$1$`，比如用`$8$`个比特就可以表示`$[-2^{7}, 2^{7}-1]$`（有一位是符号位）

那么 `float16` 的意思是用`$16$`位比特来表示浮点数，同理 `float32` 的意思是用`$32$`位比特

按照表示精度来看：
`$$
\text{float64 > float32 > float16}
$$`
按照占用空间来看：

`$$
\text{float16 < float32 < float64 }
$$`


`$\inf$`的意思是超过了可以表示的范围，有`$-\inf$`和`$\inf$`两种，而NaN的产生大概可以分为几种情况：

- 对负数开根号
- 对`$\inf$`进行运算
- 除以`$0$`

那么如何查看不同表示的范围呢？

```python
np.finfo(np.float16)
# finfo(resolution=0.001, min=-6.55040e+04, max=6.55040e+04, dtype=float16)
```

当然，numpy 这里有个小坑，就是你输入的值较大或略微超过范围时，反而会用另一个数来表示，只有大到一定程度时，才会用`$\inf$`，详见官网的[release notes](https://docs.scipy.org/doc/numpy-1.14.0/release.html#many-changes-to-array-printing-disableable-with-the-new-legacy-printing-mode)

{{< notice notice-note >}}
Floating-point arrays and scalars use a new algorithm for decimal representations, giving the shortest unique representation. This will usually shorten float16 fractional output, and sometimes float32 and float128 output. float64 should be unaffected.
{{< /notice >}}

```python
a = np.array([65504.], dtype=np.float16)
print(a)
b = np.array([65388.], dtype=np.float16)
print(b)
c = np.array([65700.], dtype=np.float16)
print(c)

# [65500.]
# [65380.]
# [inf]
```

注意，这里的大是相对于计算精度来说的，而不是你感觉的，当你把精度调成`float32`，上面都会打印原来输入的结果

## e之殇

在上高中时，特别喜爱`$e^{x}$`，有很多好的性质，比如求导等于本身，然而在很多数值溢出的情形，总有它的参与，这是因为机器学习中很多东西都会和它挂钩，比如各种激活函数便有它的影子

看个例子：

```python
a = np.exp(np.array([654.], dtype=np.float16))
print(a)

# [inf]
```

那么我如何知道输入大概多大会导致溢出呢？可以参见下述公式，当输入为`$x$`，计算`$e^{x}$`用十进制数表示大概有多少位

`$$
\log_{10}(e^x) = x \log_{10}(e)
$$`

举个例子，上面我们看到`float16`当最大位数是`$5$`位（科学计数法后面得`$+1$`），那么根据上述公式你就可以算出最大可被接受的`$x$`，进而进行一些后处理：

```python
def compute_max(n):
    return int(n * math.log(10))

print(compute_max(5)) # 11
```

我们来试试看：

```python
a = np.exp(np.array([11.], dtype=np.float16))
print(a)
b = np.exp(np.array([12.], dtype=np.float16))
print(b)

# [59870.]
# [inf]
```


接下来按照消除溢出的方法看看几大类常见例子：

## 归一化指数

当`$\exp(x_{i})$`形式出现，就会考虑通过代数恒等变换使得指数上多一些部分来进行归一化

### softmax

不妨假设`$\boldsymbol{x} \in \mathbb{R}^{n}$`，当某个`$x_{i}$`特别大时，`$\exp(x_{i})$`就会出现数值上溢，然后当分子分母都是`$\inf$`的时候就会出现NaN，`$0$`是因为分母是`$\inf$`导致的

```python
import numpy as np

def softmax(x):
    exp = np.exp(x)
    return exp / exp.sum(-1)

x = np.array([-1., 20000, 0.1], dtype=np.float16)
softmax(x)

# array([ 0., nan, 0.])
```

那么，有什么好的方法来防止这种数值溢出呢，我们通过一些不改变原式的代数运算可以做到：

`$$
\begin{align}
\mathrm{softmax}(x_{j}) &  = \frac{e^{x_{j}}}{\sum_{i} e^{x_{i}}} \\
&= {\color{#337dff}\frac{c}{c}} \cdot \frac{ e^{x_{j}}}{ \sum_{i} e^{x_{i}}} \\
&= \frac{e^{x_{j} + \log c}}{ \sum_{i} e^{x_{i} +\log c}}
\end{align}
$$`
观察上式，不难发现，我们可以控制常数`$c$`来对`$x_{i}$`进行规范化，相当于加上偏移量（offset）

比较简单的做法即为设置`$\log c = -\max(\boldsymbol{x})$`

那么：

`$$
\mathrm{softmax}(x_{j}) = \frac{e^{x_{j} - \max(\boldsymbol{x})}}{\sum_{i} e^{x_{i} - \max(\boldsymbol{x})}}
$$`

代码实现即为：

```python
def softmax(x):
    x -= max(x)
    exp = np.exp(x)
    return exp / exp.sum(-1)

softmax(x)
# array([0., 1., 0.])
```

因为最大的数减去自身变为了`$0$`，`$e^{0} = 1$`，就不会有什么影响了

这个与PyTorch官方实现也是一致的：

```python
import torch
import torch.nn.functional as F

x = torch.tensor([-1., 20000, 0.1])
F.softmax(x)
# tensor([0., 1., 0.])
```

### Logsumexp

同理再看一个类似的：

`$$
\begin{align}
\text{Logsumexp}(\boldsymbol{x}) &= \log \sum_{i} e^{x_{i}} \\
&= \log \sum_{i} e^{x_{i}} \frac{c}{c} \\
&= \log \left( \frac{1}{c} \sum_{i} e^{x_{i}} e^{\log c}\right) \\
&= -\log c + \log \sum_{i} e^{x_{i} + \log c}
\end{align}
$$`

取`$\log c=-\max(\boldsymbol{x})$`，那么：

`$$
\text{logsumexp}(\boldsymbol{x}) = \max(\boldsymbol{x}) + \log \sum_{i} e^{x_{i}-\max(\boldsymbol{x})}
$$`

```python
def logsumexp(x):
    maximum = max(x)
    x -= maximum
    exp = np.exp(x)
    return maximum + np.log(exp.sum(-1))
```

跟 PyTorch 官方实现一致：

```python
print(torch.logsumexp(x, dim=-1, keepdim=False))
print(logsumexp(x.numpy()))

# tensor(200000.)
# 200000.0
```

## 展开log内部

就是将log明显可能出现数值溢出的部分拆出来，刚刚logsumexp就是利用了这个道理

### log-softmax

尽管softmax现在稳定了，然而log-softmax还是有风险溢出，比如上面的`$\log 0$`，这也是数值上溢

`$$
\mathrm{LogSoftmax}(x_{j}) = \log\left(\frac{e^{x_{j}}}{\sum_{i} e ^{x_{i}}}\right)
$$`
我们将其拆开：
`$$
\begin{align}
\mathrm{LogSoftmax}(x_{j})  & = \log \left( \frac{e^{x_{j} - \max(\boldsymbol{x})}}{\sum_{i} e^{x_{i}-\max(\boldsymbol{x})}} \right) \\
&= \log(e^{x_{j} - \max(\boldsymbol{x})}) - \log \left( \sum_{i} e^{x_{i}-\max(\boldsymbol{x})}\right) \\
&= x_{j} - \max(\boldsymbol{x}) - \log \underbrace{ \left( \sum_{i} e^{x_{i}-\max(\boldsymbol{x})}\right) }_{ \ge 1 }
\end{align}
$$`

因为所有的`$x_{i}$`中肯定有最大的一个，那么`$\exp(x_{i} -\max(\boldsymbol{x}))=1$`，剩下的肯定是正数


```python
def log_softmax(x):
    x -= max(x)
    exp = np.exp(x)
    return x - np.log(exp.sum(-1))
```

同样与PyTorch一致，后面是两个框架显示机制不同，从这个角度也可以看出，都是32位时，PyTorch会更精准

```python
x = torch.tensor([-1., 200000, 0.1])
print(F.log_softmax(x, dim=-1))
print(log_softmax(x.numpy()))
# tensor([-200001.0000, 0.0000, -199999.9062])
# array([-200001. , 0. , -199999.9], dtype=float32)
```

## 截断

当输入大于某种阈值，直接输出原来的输入

### Softplus

下面是PyTorch官方对于Softplus的[实现](https://pytorch.org/docs/stable/generated/torch.nn.Softplus.html#torch.nn.Softplus)

`$$
\text{Softplus}({x}) = \begin{cases}
\log (1 + e^{x}),  & x \leq  \text{threshold} \\
x,  & \text{otherwise}
\end{cases}
$$`
官方的意思很简单，当`$x$`大于阈值（这里是置之为`$20$`），直接不变输出

## eps 急救包

当分母可能出现为`$0$`时，给它加上一个较小的正数`$\varepsilon$`

### Layer Normalization

`$$
\text{LayerNorm}(\boldsymbol{x}) = \gamma \left(\frac{\boldsymbol{x} - \bar{\boldsymbol{x}}}{\sigma + {\color{#337dff}\varepsilon}} \right) + \beta
$$`
分母加上一个`$\varepsilon$`来防止变为`$0$`：

```python
class LayerNorm(nn.Module):
    def init(self, features, eps=1e-6):
        super().__init__()
        self.gamma = nn.Parameter(torch.ones(features))
        self.beta = nn.Parameter(torch.zeros(features))
        self.eps = eps

    def forward(self, x):
        mean = x.mean(-1, keepdim=True)
        std = x.std(-1, keepdim=True)
        return self.gamma * (x - mean) / (std + self.eps) + self.beta
```