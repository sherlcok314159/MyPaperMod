---
title: Generalized Linear Models
date: 2023-02-17 14:10:00 +0800
tags: ["linear models"]
math: true
canonicalURL: "https://canonical.url/to/page"
---
## 定义

若一个分布能够以下述方式进行表示，则称之为指数族（ Exponential Family）的一员

`$$
\begin{equation}
p(y; \eta ) = b(y)\exp(\eta^{\mathbf{T}}T(y) - a(\eta ))
\end{equation}
$$`
其中`$\eta$`被称为分布的自然参数（natural parameter）或标准参数（canonical parameter）；而`$T(y)$`被称为统计充分量（sufficient statistic），通常而言：`$T(y) = y$`；`$a(\eta)$`是对数配分函数（log partition）

## 例子

接下来展示下伯努利分布和高斯分布都是指数族的一员

期望为`$\phi$`的伯努利分布且`$y \in \{1, 0\}$`，那么：

`$$
p(y=1) = \phi; p(y=0) = 1-\phi
$$`

接着进行转换：

`$$
\begin{align}
p(y;\phi ) &  = \phi^{y}(1-\phi )^{1-y} \\
&= \exp \bigg(\log(\phi^{y}(1-\phi )^{1-y})\bigg) \\
&= \exp(y\log \phi + (1-y)\log(1-\phi )) \\
&= \exp\left( \left( \log \frac{\phi }{1-\phi } \right)y + \log(1-\phi) \right)
\end{align}
$$`

那么可以对比着指数族的定义，易得`$b(y)=1$`以及：

`$$
\begin{align}
\eta  & = \log\left( \frac{\phi }{1-\phi } \right) \\
e^{\eta }  & = \frac{\phi}{1-\phi } \\
{\color{red}e^{-\eta }} & = \frac{1-\phi}{\phi } = \frac{1}{\phi } - 1 \\
\phi  & = \frac{1}{1+e^{-\eta }}
\end{align}
$$`

发现很有趣的一点，`$\phi(\eta)$`不就是逻辑回归中的sigmoid函数嘛，继续比对将其他的参数写完整：

`$$
\begin{align}
a(\eta ) &  = -\log(1-\phi ) = -\log\left( 1-\frac{1}{1+e^{-\eta }} \right) \\
&= \log (1+e^{\eta })
\end{align}
$$`

接下来讨论高斯分布，`$\sigma^{2}$`对`$\theta, h_{\theta }(x)$`是不影响的（相当于常数），为了简化表示，约定`$\sigma^{2}=1$`

`$$
\begin{align}
p(y;\mu ) &  = \frac{1}{\sqrt{ 2\pi  }}\exp\left( -\frac{(y-\mu )^{2}}{2} \right) \\
&= \underbrace{ \frac{1}{\sqrt{ 2\pi  }} \exp\left( -\frac{y^{2}}{2} \right)  }_{ b(y) }\exp\left( \mu y - \frac{\mu^{2}}{2} \right)
\end{align}
$$`

比对定义，可以发现：

`$$
\eta = \mu; a(\eta ) = -\frac{\eta^{2}}{2}
$$`

## 性质

`$$
p(y;\eta ) = b(y)\exp (\eta^{\mathbf{T}}T(y) - a(\eta))
$$`

{{< notice notice-note >}}
1. 指数族分布的期望是`$a(\eta)$`对`$\eta$`的一阶微分
2. 指数族分布的方差是`$a(\eta)$`对`$\eta$`的二阶微分
3. 指数族分布的NLL loss是concave的
{{< /notice >}}

下面来证明上述观点：

`$$
\begin{align}
\frac{ \partial  }{ \partial \eta  } p(y;\eta )  & = b(y) \exp(\eta^{\mathbf{T}}y - a(\eta )) \left( y - \frac{ \partial  }{ \partial \eta  }a(\eta )  \right) \\
&= yp(y;\eta ) - p(y;\eta ) \frac{ \partial  }{ \partial \eta  } a(\eta )
\end{align}
$$`

那么：

`$$
y p(y;\eta ) = \frac{ \partial   }{ \partial \eta  } p(y;\eta ) + p(y;\eta ) \frac{ \partial  }{ \partial \eta  } a(\eta )
$$`

又因为：

`$$
\begin{align}
\mathbb{E}[Y;\eta ]  & = \mathbb{E}[Y|X;\eta]  \\
&= \int yp(y;\eta ) \,dy  \\
&= \int \frac{ \partial  }{ \partial \eta  } p(y;\eta ) + p(y;\eta )\frac{ \partial  }{ \partial \eta  }a(\eta )  \, dy \\
&= \int \frac{ \partial  }{ \partial \eta  }p(y;\eta )  \, dy  + \int p(y;\eta )\frac{ \partial  }{ \partial \eta  }a(\eta )  \, dy \\
&= \frac{ \partial  }{ \partial \eta  }  \int  p(y;\eta ) \, dy + \frac{ \partial  }{ \partial \eta  }a(\eta ) \int p(y;\eta ) \, dy \\
&= \frac{ \partial  }{ \partial \eta  }   \cdot 1 + \frac{ \partial  }{ \partial \eta  } a(\eta ) \cdot 1 \\
&= 0 + \frac{ \partial  }{ \partial \eta  } a(\eta ) \\
&= \frac{ \partial  }{ \partial \eta  } a(\eta) \quad \blacksquare
\end{align}
$$`

下面来证明方差：

`$$
\begin{align}
\frac{ \partial^{2}  }{ \partial \eta^{2} } p(y;\eta )  & =  \frac{ \partial  }{ \partial \eta  } \bigg(yp(y;\eta ) - p(y;\eta ) \frac{ \partial  }{ \partial \eta  } a(\eta )\bigg)  \\
&= y\frac{ \partial  }{ \partial \eta  }p(y;\eta )  - \frac{ \partial  }{ \partial \eta } a(\eta )\frac{ \partial  }{ \partial \eta  }p(y;\eta )  - p(y;\eta )\frac{ \partial^{2}  }{ \partial \eta^{2}  } a(\eta) \\
&= \frac{ \partial  }{ \partial \eta  }p(y;\eta ) \bigg(y - \frac{ \partial  }{ \partial \eta  }a(\eta ) \bigg) - p(y;\eta )\frac{ \partial^{2}  }{ \partial \eta^{2}  } a(\eta ) \\
&= \bigg(yp(y;\eta ) - p(y;\eta ) \frac{ \partial  }{ \partial \eta  } a(\eta )\bigg)\bigg(y- \frac{ \partial  }{ \partial \eta }a(\eta ) \bigg) - p(y;\eta ) \frac{ \partial^{2}  }{ \partial \eta^{2}  } a(\eta ) \\
&= y^{2}p(y;\eta ) - 2yp(y;\eta ) \frac{ \partial  }{ \partial \eta  }a(\eta ) + p(y;\eta ) (\frac{ \partial  }{ \partial \eta  }a(\eta ) )^{2} -  p(y;\eta ) \frac{ \partial^{2}  }{ \partial \eta^{2}  } a(\eta ) \\
&= \bigg(y - \frac{ \partial  }{ \partial \eta  }a(\eta) \bigg)^{2} p(y;\eta ) -p(y;\eta ) \frac{ \partial^{2}  }{ \partial \eta^{2} } a(\eta)
\end{align}
$$`

那么：

`$$
\bigg(y - \frac{ \partial  }{ \partial \eta  }a(\eta) \bigg)^{2} p(y;\eta ) = \frac{ \partial^{2}  }{ \partial \eta^{2} } p(y;\eta ) +p(y;\eta ) \frac{ \partial^{2}  }{ \partial \eta^{2} } a(\eta)
$$`

又因为：

`$$
\begin{align}
\mathbb{V}[Y;\eta ]  & = \mathbb{V}[Y|X;\eta] \\
&= \int \left( y - \frac{ \partial  }{ \partial \eta  } a(\eta ) \right)^{2} p(y;\eta)\, dy \\
&= \int \frac{ \partial^{2}  }{ \partial \eta^{2} }p(y;\eta) + p(y;\eta ) \frac{ \partial^{2}  }{ \partial \eta^{2} }a(\eta )   \, dy \\
&= \frac{ \partial^{2}  }{ \partial \eta^{2} }   \int p(y;\eta ) \, dy + \frac{ \partial ^{2} }{ \partial \eta^{2} }  a(\eta )\int  p(y;\eta ) \, dy \\
&= 0 + \frac{ \partial ^{2} }{ \partial \eta^{2} }  a(\eta ) \\
&=  \frac{ \partial^{2}  }{ \partial \eta^{2} }  a(\eta ) \quad \blacksquare
\end{align}
$$`

接下来证明NLL Loss是凸函数，其中`$a(\eta) \in \mathbb{R}^{m}, \mathbf{y} \in \mathbb{R}^{m}$`：

`$$
\begin{align}
J(\eta)  & = -\log \sum_{i}^{m} p(y^{(i)};\eta_{i} ) \\
&= -\log \sum_{i}^{m} b(y^{(i)})\exp(\eta_{i} T(y^{(i)}) - a(\eta_{i})) \\
&= -\log \sum_{i}^{m} b(y^{(i)}) \exp( \eta_{i} y^{(i)}- a(\eta_{i}))  \\
&= a(\eta) -\sum_{i}^{m}   \log b(y^{(i)}) + \eta_{i}y^{(i)}
\end{align}
$$`
同时因为自身的协方差矩阵是「半正定」的：

`$$
\begin{align}
\nabla_{\eta }^{2} J(\eta ) = \frac{ \partial^{2} }{ \partial \eta^{2} } a(\eta ) = \mathbb{V}[\mathbf{y};\eta ] \implies PSD\quad \blacksquare
\end{align}
$$`

由[[Critical Points#多变量]]可知，当Hessian矩阵是半正定时，NLL Loss是凸函数，有局部最小点

## 构建GLM

在现实生活中，根据我们需要预测的变量来选取合适的分布，那么，如何构建模型去预测它呢？这里模型又被称为广义线性模型（Generalized Linear Model），要构建GLM，需要先进行一些假设：

1. `$y|x; \theta \sim \mathrm{ExponentialFamily}(\eta )$`，也就是说，给定`$x, \theta$` 我们可以得到`$y$`的分布就是带有参数`$\eta$`的指数族分布
2. 给定`$x$`，我们需要去预测`$y$`，在指数族中，`$y=T(y)$`，也就是说我们得去预测期望，即学得的假设`$h$`需要去预测期望，即：`$h(x) = \mathbb{E}[y|x]$`，这个在线性回归和逻辑回归都是满足的，举个逻辑回归的例子：
`$$
h_{\theta }(x) = p(y=1|x;\theta ) = 0 \cdot p(y=0|x;\theta) + 1 \cdot p(y=1|x;\theta)  = \mathbb{E}[y|x;\theta ]
$$`
3. 自然参数`$\eta$`与`$x$`的联系是线性的，即`$\eta = \theta^{\mathbf{T}}x$`，若`$\eta \in \mathbb{R}^{n}$`，则`$\eta_{i} = \theta_{i}^{\mathbf{T}}{x}$`

根据变量的特点来选择合适的分布：

|          Variable          	|    Distribution    	|
|:-------------------------:	|:------------------:	|
| Real Numbers `$\mathbb{R}$` 	|      Gaussian      	|
|   Binary Classification   	|      Bernoulli     	|
|           Count           	|       Poisson      	|
|      `$\mathbb{R}^{+}$`     	| Gamma, Exponential 	|
|        Distribution       	|   Beta, Dirichlet  	|

## GLM例子

接下来举一些GLM的例子

### OLS

Ordinary Least Squares（线性回归）中的`$y|x;\theta  \sim \mathcal{N}(\mu, \sigma^{2})$`，即服从高斯分布，其中`$\eta = \mu$`，那么：
`$$
\begin{align}
h_{\theta }(x) &  = \mathbb{E}[y|x;\theta] \\
&  = \mu \\
&= \eta \\
&= \theta^{\mathbf{T}}{x}
\end{align}
$$`
第一个等号是因为第二个假设，第三个等号是根据指数族的定义来的，而第四个等号则是第三个假设

### 逻辑回归

逻辑回归中`$y \in \{0, 1\}$`，自然就想到了伯努利分布，即`$y|x;\theta  \sim \text{Bernoulli}(\phi )$`，其中`$\phi=1/1+ e^{-\eta }$`，那么：

`$$
\begin{align}
h_{\theta }(x) &  = \mathbb{E}[y|x;\theta ]  \\
&  = \phi  \\
&= \frac{1}{1 + e^{-\eta }} \\
&= \frac{1}{1 + e^{-\theta^{\mathbf{T}}x}}
\end{align}
$$`
是不是很神奇，那么关于为何逻辑回归中的假设函数取上述形式，又多了一种解释，即根据指数族分布和GLM的定义而来

`$g(\eta) = \mathbb{E}[T(y);\eta ]$`被称为响应函数（canonical response function），在深度学习中，常被称作为激活函数，而`$g^{-1}$`被称作为链接函数（canonical link function）。那么，对于高斯分布而言，响应函数就是单位函数；而对于伯努利分布而言，响应函数即为sigmoid函数（对于两个名词的定义，不同的文献可能相反）

## softmax回归

### 构建GLM

之前逻辑回归中是只有两类，当`$y \in \{1, 2, \dots, k\}$`，即现在是`$k$`分类，分布是multinomial distribution，接下来让我们构建GLM：

规定`$\phi_{i}$`规定了输出`$y_{i}$`的概率，那么`$\phi_{i}, \dots, \phi_{k-1}$`即是我们的参数，那你肯定好奇为什么`$\phi_{k}$`不是，因为输出所有类的概率之和为`$1$`，即`$\phi_{k}$`可被其他的概率表示：

`$$
\phi_{k} = 1-\sum_{i}^{k-1} \phi_{i}
$$`
以往的`$T(y)=y$`，对于多分类而言，我们采用独热编码（one-hot），即：

`$$
T(1) = \begin{bmatrix}
1 \\
0 \\
\vdots \\
0
\end{bmatrix}, T(2) = \begin{bmatrix}
0 \\
1 \\
\vdots \\
0
\end{bmatrix}, \dots ,T(k-1) = \begin{bmatrix}
0 \\
0 \\
\vdots \\
1
\end{bmatrix}, T(k) = \begin{bmatrix}
0 \\
0 \\
\vdots \\
0
\end{bmatrix}
$$`
注意`$T(y) \in \mathbb{R}^{k-1}$`，因为`$T(k)$`定义为全零向量，那么如何表示`$T(y)$`的第`$i$`个元素呢？

`$$
(T(y))_{i} = \mathbb{1}\{y=i\}
$$`

接下来来构建GLM，写出其概率密度表示，注意：这里容易误以为是MLE中的所有概率相乘，然而当`$y$`取一个具体值时，只有一个指示函数为`$1$`，其他为`$0$`，即`$\phi_{i}^{0} = 1$`

`$$
\begin{align}
p(y;\phi ) &  = \phi^{\mathbb{1}\{y=1\}}_{1} \phi_{2}^{\mathbb{1}\{y=2\}} \dots \phi^{\mathbb{1}\{y=k\}}_{k} \\
&  = \phi^{\mathbb{1}\{y=1\}}_{1} \phi_{2}^{\mathbb{1}\{y=2\}} \dots \phi^{1-\sum_{i}^{k-1}\mathbb{1}\{y=i\}}_{k} \\
&= \phi_{1}^{(T(y))_{1}} \phi_{2}^{(T(y)_{2})} \dots \phi_{k}^{1-\sum_{i}^{k-1}(T(y))_{i}}  \\
\end{align}
$$`
继续变形来跟定义做比较：

`$$
\begin{align}
p(y; \phi)&= \exp \bigg((T(y))_{1}\log \phi_{1} + (T(y))_{2} \log \phi_{2} + \dots +(1-\sum_{i}^{k-1}(T(y))_{i})\log \phi_{k} \bigg) \\
&= \exp \bigg((T(y))_{1}\log \frac{\phi_{1}}{\phi_{k}} + (T(y))_{2}\log \frac{\phi_{2}}{\phi_{k}} + \dots+ (T(y))_{k-1}\log \frac{\phi_{k-1}}{\phi_{k}} + \log \phi_{k}\bigg)
\end{align}
$$`

那么：

`$$
\eta = \begin{bmatrix}
\log (\phi_{1} / \phi_{k}) \\
\log (\phi_{2} / \phi_{k}) \\
\vdots \\
\log(\phi_{k-1} / \phi_{k})
\end{bmatrix}, a(\eta ) = -\log(\phi_{k}), b(y) =1
$$`
链接函数容易发现是：

`$$
\eta_{i} = \log \frac{\phi_{i}}{\phi_{k}}
$$`
接下来求响应函数：

`$$
\begin{align}
e^{\eta_{i}} &  = \frac{\phi_{i}}{\phi_{k}} \\
\phi_{k}e^{\eta_{i}} & = \phi_{i} \\
\phi_{k}\sum_{i}^{k} e^{\eta_{i}}  & = \sum_{i}^{k} \phi_{i} = 1
\end{align}
$$`
那么：

`$$
\phi_{k} = \frac{1}{\sum_{i}^{k} e^{\eta_{i}}}
$$`
将`$\phi_{k}$`代入上式：

`$$
\phi_{i} = \frac{e^{\eta_{i}}}{\sum_{j=1}^{k} e^{\eta_{j}}}
$$`
这就是我们的激活函数，在深度学习中常被称为「softmax」函数，接下来便可构建GLM：

`$$
\begin{align}
p(y=i|x;\theta ) &  = \phi_{i} \\
&= \frac{e^{\eta_{i}}}{\sum_{j=1}^{k} e^{\eta_{j}}} \\
&= \frac{e^{\theta_{i}^{\mathbf{T}}x}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x}}
\end{align}
$$`
多分类问题被看作是逻辑回归的推广版，又被称为「softmax regression」，我们的假设函数如下：

`$$
\begin{align}
h_{\theta }(x) &  = \mathbb{E}[T(y)|x;\theta ] \\
&= \mathbb{E}\left[\begin{array}{c|}
\mathbb{1}\{y=1\} \\
\mathbb{1}\{y=2\} \\
\vdots \\
\mathbb{1}\{y=k-1\}
\end{array} x;\theta \right] \\
\end{align}
$$`

又因为：

`$$
\mathbb{E}[(T(y))_{i}] =  \phi_{i}
$$`
为啥会这样呢？因为对于`$(T(y))_{i}$`只有两个可能，`$1$`或`$0$`，那么它的期望是不是：

`$$
\mathbb{E}[(T(y))_{i}] = 1 \cdot \phi_{i} + 0 \cdot (1-\phi ) = \phi_{i}
$$`

`$$
h_{\theta }(x)= \begin{bmatrix}
\phi_{1} \\
\phi_{2} \\
\vdots \\
\phi_{k-1}
\end{bmatrix}= \begin{bmatrix}
\frac{\exp ({\theta_{1}^{\mathbf{T}}x)}}{\sum_{j=1}^{k} \exp(\theta_{j}^{\mathbf{T}}x)} \\
\frac{\exp ({\theta_{2}^{\mathbf{T}}x})}{\sum_{j=1}^{k} \exp(\theta_{j}^{\mathbf{T}}x)} \\
\vdots \\
\frac{\exp ({\theta_{k-1}^{\mathbf{T}}x})}{\sum_{j=1}^{k} \exp(\theta_{j}^{\mathbf{T}}x)}
\end{bmatrix}
$$`
也就是说我们的假设函数需要输出每个类的概率，尽管只有`$k-1$`类，`$\phi_{k} = 1- \sum_{i}^{k-1} \phi_{i}$`得到

接下来进行最大似然估计并对`$\ell(\theta)$`进行化简：

`$$
\begin{align}
\ell(\theta )  & = \sum_{i} \log p(y^{(i)}|x^{(i)};\theta ) \\
&= \sum_{i} \log \prod_{l=1}^{k} \left( \frac{e^{\theta_{l}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right)^{\mathbb{1}\{y^{(i)}=l\}} \\
&= \sum_{i}^{m} \sum_{l=1}^{k} \log \left( \frac{e^{\theta_{l}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right)^{\mathbb{1}\{y^{(i)}=l\}} \\
&= \sum_{i}^{m} \sum_{l=1}^{k} {\color{red}\mathbb{1}\{y^{(i)}=l\}} \log \left( \frac{e^{\theta_{l}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right) \\
&= \sum_{i}^{m}\sum_{l=1}^{k} \mathbb{1}\{y^{(i)} = l\} \left( \log e^{\theta_{l}^{\mathbf{T}}x^{(i)}}- \log \sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}} \right) \\
&= \sum_{i}^{m}\sum_{l=1}^{k} \mathbb{1}\{y^{(i)} = l\} \left(\theta_{l}^{\mathbf{T}}x^{(i)}- \log \sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}\right) \\
&= \sum_{i}^{m}\sum_{l=1}^{k} \mathbb{1}\{y^{(i)} = l\} \theta_{l}^{\mathbf{T}}x^{(i)}-\left( \log \sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}\underbrace{ \sum_{l=1}^{k} \mathbb{1}\{y^{(i)} = l\} }_{ 1 }\right)  \\
&= \sum_{i}^{m} \bigg(\sum_{l=1}^{k} \mathbb{1}\{y^{(i)} = l\} \theta_{l}^{\mathbf{T}}x^{(i)} - \log \sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}\bigg)
\end{align}
$$`

上述化简主要利用了指示函数的性质以及`$\log$`的运算法则，同时`$\theta \in \mathbb{R}^{k \times n}$`，我们利用布局法来求：

`$$
\begin{align}
\frac{ \partial \ell(\theta ) }{ \partial \theta_{pq} }   & = \sum_{i}^{m} \mathbb{1}\{y^{(i)} = p\}  x^{(i)}_{q} -  \frac{1}{\sum_{j=1}^{k}e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} e^{\theta_{p}^{\mathbf{T}}x^{(i)}}x^{(i)}_{q}  \\
&= \sum_{i}^{m} \left( \mathbb{1}\{y^{(i)} = p\} - \frac{e^{\theta_{p}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right)x^{(i)}_{q}
\end{align}
$$`
因为这里是最大化`$\ell(\theta)$`，作为损失函数还应加个负号，这样才是最小，即

`$$
J(\theta ) = -\sum_{i} \log \prod_{l=1}^{k} \left( \frac{e^{\theta_{l}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right)^{\mathbb{1}\{y^{(i)}=l\}}
$$`

对应的微分如下：
`$$
\frac{ \partial J(\theta ) }{ \partial \theta_{pq} }   = \sum_{i}^{m} \left(  \frac{e^{\theta_{p}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} - \mathbb{1}\{y^{(i)} = p\} \right)x^{(i)}_{q}
$$`

### 交叉熵

我们常常称多分类的损失叫「交叉熵损失」（cross entropy loss），那么根据GLM推导的式子和交叉熵的联系是什么呢？

联想交叉熵的定义：

`$$
H(P, Q) = - \mathbb{E}_{x \sim P}[\log Q(x)]
$$`
即使得模型输出的分布尽可能靠近训练集原来的分布：

`$$
\theta^{\ast} = \mathop{\arg \min}_{\theta} -\mathbb{E}_{x \sim \mathcal{D}}[\log p_{model}(x)]
$$`
我们接下来展开期望的计算：

`$$
-\mathbb{E}_{x \sim \mathcal{D}}[\log p_{model}(x)] = \frac{1}{m}\underbrace{ -\sum_{i}^{m} \log \prod_{l=1}^{k} \left( \frac{e^{\theta_{l}^{\mathbf{T}}x^{(i)}}}{\sum_{j=1}^{k} e^{\theta_{j}^{\mathbf{T}}x^{(i)}}} \right)^{\mathbb{1}\{y^{(i)}=l\}}  }_{ J(\theta) }
$$`
两者其实就差一个常数，本质是一样的

代码实现也比较轻松：

```python
def CrossEntropy(y_pred, y_true):
    batch_size = y_pred.shape[0]
    y_pred = np.exp(y_pred)
    y_pred /= np.sum(y_pred, axis=1)[:, None]
    y_pred = np.take_along_axis(y_pred, y_true[:, None], axis=1)
    y_pred = np.log(y_pred)
    return -np.sum(y_pred) / batch_size
```