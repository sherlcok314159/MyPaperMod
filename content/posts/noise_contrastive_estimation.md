---
title: Noise Contrastive Estimation
date: 2023-05-29 10:10:00 +0800
tags: [ML]
math: true
canonicalURL: "https://canonical.url/to/page"
---
## 难以承受之重

文本生成是 NLP 任务中比较典型的一类，记参数为`$\boldsymbol{\theta }$`，给定的 context 为`$\boldsymbol{c}$`，需要生成的文本记为`$\boldsymbol{w}$`，我们通常通过最大似然法来使得模型预测的分布`$p_{\boldsymbol{\theta}}$`尽可能接近训练集分布`$p_{d}(\boldsymbol{w})$`

`$$
\boldsymbol{\theta ^{\ast}} = \mathop{\arg \max}_{\boldsymbol{\theta }} \  \mathbb{E}_{\boldsymbol{c}, \boldsymbol{w} \sim p_{d}} \log p_{\boldsymbol{\theta }}(\boldsymbol{w}|\boldsymbol{c};\boldsymbol{\theta })
$$`

而在建模时，我们通常会在模型加入 Softmax 来将 score 转换为概率，使得对词表`$\mathcal{V}$`所有词预测概率相加为 1：

`$$
p(\boldsymbol{w}|\boldsymbol{c};\boldsymbol{\theta}) = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{\underbrace{ {\color{#337dff}\sum_{\boldsymbol{w}' \in \mathcal{V}} u_{\boldsymbol{\theta}}(\boldsymbol{w}', \boldsymbol{c})} }_{ \text{Partition Function} }}, u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c}) = \exp(s_{\theta }(\boldsymbol{w}, \boldsymbol{c}))
$$`

分母是用来归一化的，也被称作配分函数（Partition Function），为了使得表达更简便，我们将上述公式进一步压缩，将分母统称为`$Z(\boldsymbol{\theta })$`

若是普通的多分类问题，参数量不大，求`$Z(\boldsymbol{\theta })$`感觉不到压力，可若是文本生成任务，例如，「文本\_\_\_\_是自然语言处理的任务」，此时你得去整个词表`$\mathcal{V}$`中来挑选词来填空，去计算`$Z(\boldsymbol{\theta })$`就是十分昂贵的事情了

## 丢给参数

那么，有人就说了，不行那就直接交给参数处理吧，让模型自己去学，看看模型自己能不能学出归一化：

`$$
p(\boldsymbol{w}|\boldsymbol{c};\boldsymbol{\theta}) = \frac{u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})}{Z(\boldsymbol{\theta })} = u_{\boldsymbol{\theta }}(\boldsymbol{w}, \boldsymbol{c})\exp(z^{\boldsymbol{c}}), \, z^{\boldsymbol{c}} = -\log Z(\boldsymbol{\theta })
$$`

接着应用最大似然：

`$$
\begin{align}
\boldsymbol{\theta ^{\ast}}  & = \mathop{\arg \max}_{\boldsymbol{\theta }} \  \mathbb{E}_{\boldsymbol{c}, \boldsymbol{w} \sim p_{d}} \log p_{\boldsymbol{\theta }}(\boldsymbol{w}|\boldsymbol{c};\boldsymbol{\theta }) \\
&=  \mathop{\arg \max}_{\boldsymbol{\theta }, \boldsymbol{z}} \ \mathbb{E}_{\boldsymbol{c}, \boldsymbol{w} \sim p_{d}} \log u_{\boldsymbol{\theta }}(\boldsymbol{w},\boldsymbol{c})\exp(z^{\boldsymbol{c}})
\end{align}
$$`

这样的结果就是，为了最大化期望，会使得`$Z(\boldsymbol{\theta }) \to 0$`，效果会很不好

## 曲径通幽

那么 Noise Contrastive Estimation（NCE）说，既然这样，我们能不能引入参数的同时也可以出色地预估`$Z(\boldsymbol{\theta })$`呢？于是乎，它将问题从原本的多分类问题转换为二分类问题
{{< sidenote >}}
Proxy Problem，指用新的任务或指标来完成对原本任务的建模
{{< /sidenote >}}
具体如下：

首先，存在一个噪声分布`$p_{n}$`和经验概率分布`$p_{d}$`，这里`$p_{d}$`是从训练集提取的，就类似 word2vec 的训练，将句子切分成词
{{< sidenote >}}
现代 NLP 基本都是 token，这里是为了表达简便
{{< /sidenote >}}
，统计某几个词一起出现的概率，那么`$p(\boldsymbol{w}|\boldsymbol{c})$`就是对于`$\boldsymbol{c}$`而言，下一个词是`$\boldsymbol{w}$`的概率。举个例子，对于 love 而言：`$p_{d}=\{ \text{games}: 0.9, \text{study}: 0.1 \}$`

每次从`$p_{d}$`中抽出一个候选词，从`$p_{n}$`中抽取`$k$`个候选词。模型的任务即为区分候选词是从训练集还是噪声中采样而来的，通过这个代理任务使得`$p_{\boldsymbol{\theta}}(\boldsymbol{c})$`去逼近于`$p_{d}(\boldsymbol{c})$`

我们规定，当`$\mathcal{D}=1$`时代表从训练集采样，而`$\mathcal{D}=0$`则代表从噪声中采样，那么：

`$$
\begin{align}
p(\boldsymbol{w}|\mathcal{D} =1, \boldsymbol{c})  & = p_{d}(\boldsymbol{w})\\
p(\boldsymbol{w}|\mathcal{D} = 0, {\boldsymbol{c}})  & = p_{n}({\boldsymbol{w}})
\end{align}
$$`

那么总概率即为：

`$$
p_{joint}(\boldsymbol{w}) =\frac{1}{k+1}p_{d}(\boldsymbol{w}) + \frac{k}{k+1} p_{n}(\boldsymbol{w})
$$`

接下来求一下来自哪个采样的条件概率，为了表达简便，对`$\boldsymbol{c}$`进行省略：`$p(\mathcal{D}=1|\boldsymbol{c}, \boldsymbol{w}) = p(\mathcal{D}=1|\boldsymbol{w})$`：

`$$
\begin{align}
p(\mathcal{D}=1|\boldsymbol{w})  & = \frac{p(\mathcal{D}=1,\boldsymbol{w})}{p_{joint}(\boldsymbol{w})} = \frac{p(\boldsymbol{w}|\mathcal{D}=1)p(\mathcal{D}=1)}{p_{joint}(\boldsymbol{w})} \\
 & = \frac{p_{d}(\boldsymbol{w})}{p_{d}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}
\end{align}
$$`

同理：

`$$
p(\mathcal{D}=0|\boldsymbol{w}) = \frac{kp_{n}(\boldsymbol{w})}{p_{d}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}
$$`

`$\mathcal{D}$`代表训练集和噪声集的集合，那么 NCE 的目标即为最大化期望：

`$$
\boldsymbol{\theta }^{\ast} = \mathop{\arg \max}_{\boldsymbol{\theta }}\ \mathbb{E}_{\boldsymbol{w} \sim \mathcal{D}} \log p(\mathcal{D}|\boldsymbol{w})
$$`

又因为我们想要让模型分布`$p_{\boldsymbol{\theta }}$`尽可能接近训练集分布`$p_{d}$`，于是我们在求条件概率时，将`$p_{d}$`换成`$p_{\boldsymbol{\theta }}$`，即：

`$$
\begin{align}
p(\mathcal{D}=1|\boldsymbol{w})  & = \frac{p_{\boldsymbol{\theta }}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \\
p(\mathcal{D}=0|\boldsymbol{w})  & = \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}
\end{align}
$$`

我们展开期望看看：

`$$
\begin{align}
\mathbb{E}_{\boldsymbol{w}\sim \mathcal{D}} \log p(\mathcal{D}|\boldsymbol{w})  & = \int_{\boldsymbol{w}} p(\boldsymbol{w})\log p(\mathcal{D}|\boldsymbol{w}) \, d\boldsymbol{w} \\
&= \int_{\boldsymbol{w}} \frac{1}{k+1}(p_{d}(\boldsymbol{w})+kp_{n}(\boldsymbol{w}))\log p(\mathcal{D}|\boldsymbol{w})\, d \boldsymbol{w} \\
&= \frac{1}{k+1} \left( \int_{\boldsymbol{w}} p_{d}(\boldsymbol{w}) \log p(\mathcal{D}=1|\boldsymbol{w})  \, d \boldsymbol{w} + \int _{\boldsymbol{w}} k p_{n}(\boldsymbol{w}) \log p(\mathcal{D}=0|\boldsymbol{w}) \, d\boldsymbol{w}  \right) \\
&= \frac{1}{k+1}\bigg(\mathbb{E}_{\boldsymbol{w} \sim p_{d}} \log p(\mathcal{D}=1|\boldsymbol{w})+k\mathbb{E}_{\boldsymbol{w} \sim p_{n}}\log p(\mathcal{D}=0|\boldsymbol{w})\bigg)
\end{align}
$$`

上述期望计算初看肯定有两处疑问：

第一、为啥要基于`$\boldsymbol{w}$`而非`$\boldsymbol{c}$`来展开概率计算呢？当然两者都可以，但是我们的目标是为了让模型分布去拟合训练集分布，若是按照`$\boldsymbol{c}$`展开，也就是`$p_{\boldsymbol{\theta }}(\boldsymbol{c})\approx p_{d}(\boldsymbol{c})$`，让模型预测输入的 feature 不合理

第二、不是说好用模型分布代替数据分布吗？为什么`$p(\boldsymbol{w})$`还是用的数据分布，我想可能是为了训练方便考虑，若两处都是模型分布，训练势必更难；同时，数据分布是一个既定事实，可以充当额外的信息量给模型，加快收敛

因为`$k$`是常数，对优化目标函数无影响，下式省略之，那么，我们的目标函数即为：

`$$
J(\boldsymbol{\theta }) =\mathbb{E}_{\boldsymbol{w} \sim p_{d}} \log p(\mathcal{D}=1|\boldsymbol{w})+k\mathbb{E}_{\boldsymbol{w} \sim p_{n}}\log p(\mathcal{D}=0|\boldsymbol{w})
$$`

## 极限的视角

你肯定好奇 Proxy Problem 是否可以近似原来的建模，目标函数相对于`$\boldsymbol{\theta }$`的微分告诉了我们答案

`$$
J(\boldsymbol{\theta })   =\mathbb{E}_{\boldsymbol{w} \sim p_{d}} \log p(\mathcal{D}=1|\boldsymbol{w})+k\mathbb{E}_{\boldsymbol{w} \sim p_{n}}\log p(\mathcal{D}=0|\boldsymbol{w})
$$`

那么，我们求关于参数`$\boldsymbol{\theta }$`的微分：

`$$
\begin{align}
\frac{ \partial  }{ \partial \boldsymbol{\theta } } J(\boldsymbol{\theta })  & = \frac{ \partial  }{ \partial \boldsymbol{\theta } } \mathbb{E}_{\boldsymbol{w} \sim p_{d}} \log p(\mathcal{D}=1|\boldsymbol{w})+\frac{ \partial  }{ \partial \boldsymbol{\theta } } k\mathbb{E}_{\boldsymbol{w} \sim p_{n}}\log p(\mathcal{D}=0|\boldsymbol{w}) \\
&= \mathbb{E}_{\boldsymbol{w}\sim p_{d}} \frac{ \partial  }{ \partial \boldsymbol{\theta } }  \log \frac{p_{\boldsymbol{\theta }}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} + k\mathbb{E}_{\boldsymbol{w}\sim p_{n}} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}
\end{align}
$$`

那么接下来我们拆开来求目标函数相对于参数的微分：

`$$
\begin{align}
\frac{ \partial  }{ \partial \boldsymbol{\theta } } \log \frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} &  = \frac{p_{\theta }(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}  \\
&= \frac{p_{\theta }(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})} \frac{p_{\boldsymbol{\theta}}'(\boldsymbol{w})(p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w}))-p_{\boldsymbol{\theta}}(\boldsymbol{w})p_{\boldsymbol{\theta}}'(\boldsymbol{w})}{(p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w}))^{2} } \\
&= \frac{p_{\boldsymbol{\theta }}'(\boldsymbol{w})kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})(p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w}))} \\
&= \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} {\color{#337dff}\frac{p_{\boldsymbol{\theta }}'(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})}} \\
&=\frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log p_{\boldsymbol{\theta}}(\boldsymbol{w})
\end{align}
$$`

另一部分：

`$$
\begin{align}
\frac{ \partial  }{ \partial \boldsymbol{\theta } } \log \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}  & =\frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}{kp_{n}(\boldsymbol{w})}  \frac{ \partial  }{ \partial \boldsymbol{\theta } }  \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \\
&= \frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}{kp_{n}(\boldsymbol{w})} \frac{0-kp_{n}(\boldsymbol{w})p_{\boldsymbol{\theta}}'(\boldsymbol{w})}{(p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w}))^{2}} \\
&= -\frac{p_{\boldsymbol{\theta}}'(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} {\color{#337dff}\frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})}}\\
&= - \frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log p_{\boldsymbol{\theta}}(\boldsymbol{w})
\end{align}
$$`

合起来看看：

`$$
\begin{align}
\frac{ \partial  }{ \partial \boldsymbol{\theta } } J(\boldsymbol{\theta })  & = \mathbb{E}_{\boldsymbol{w} \sim p_{d}} \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log p_{\boldsymbol{\theta}}(\boldsymbol{w}) -k \mathbb{E}_{\boldsymbol{w} \sim p_{n}}\frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log p_{\boldsymbol{\theta}}(\boldsymbol{w}) \\
&= \sum_{\boldsymbol{w}} p_{d}(\boldsymbol{w}) \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } }  \log p_{\boldsymbol{\theta }}(\boldsymbol{w}) -k \sum_{\boldsymbol{w}}p_{n}(\boldsymbol{w})\frac{p_{\boldsymbol{\theta}}(\boldsymbol{w})}{p_{\boldsymbol{\theta}}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})} \frac{ \partial  }{ \partial \boldsymbol{\theta } } \log p_{\boldsymbol{\theta}}(\boldsymbol{w}) \\
&= \sum_{\boldsymbol{w}} \underbrace{ {\color{#337dff}\frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}} }_{ k \to \infty, ratio \to 1 }\bigg(p_{d}(\boldsymbol{w}) - p_{\boldsymbol{\theta }}(\boldsymbol{w})\bigg) \frac{ \partial  }{ \partial \boldsymbol{\theta } }  \log p_{\boldsymbol{\theta }}(\boldsymbol{w}) \\
&\approx \sum_{\boldsymbol{w}}\bigg(p_{d}(\boldsymbol{w}) - p_{\boldsymbol{\theta }}(\boldsymbol{w})\bigg) \frac{ \partial  }{ \partial \boldsymbol{\theta } }  \log p_{\boldsymbol{\theta }}(\boldsymbol{w})
\end{align}
$$`

对于第四步的近似，可以举个例子，比如`$10000/(1+10000)$`，其实因为`$1$`太小了，可以忽略不计

这也是为什么这个 Proxy Problem 可以 work 的原因，当采样的噪声样本足够多时，NCE 的梯度就接近于一开始我们想要直接去做最大似然的梯度

## 两次近似

尽管通过引入参数可以去估计`$Z(\boldsymbol{\theta })$`很巧妙，但有一个很大的问题，对于每一组词而言，尽管`$\mathcal{V}$`是一样的，然而基于的`$\boldsymbol{c}$`不一致，那么`$p(\boldsymbol{w}|\boldsymbol{c})$`也是不同的，即每组词你得去保存一个参数`$z^{\boldsymbol{c}}$`

这个时候作者发现了「神之一手」，直接令`$Z(\boldsymbol{\theta })\approx 1$`，也就是俗称的 self-normalization，换句话说，压根没有转换为概率，你看到这肯定会露出不屑的表情，我也一样

自归一化 work 的原因是什么呢？引用原著的说法：

{{< notice notice-note >}}
We believe this is because the model has so many free parameters that meeting the approximate per-context normalization constraint encouraged by the objective function is easy.
{{< /notice >}}

作者的意思就是参数很多，于是就有了 power，模型自己可以去学习归一化，当然，原著中做了对比，发现效果几乎没影响，才这么做的

其实我看来还是目标函数选的好，因为当梯度近似为`$0$`时，`$p_{d}$`和`$p_{\boldsymbol{\theta }}$`很接近

注意，当采用自归一化时：`$p(\boldsymbol{w}|\boldsymbol{c}) = \exp(s_{\boldsymbol{\theta}}(\boldsymbol{w}, \boldsymbol{c})) / 1$`

这里其实有个容易误解的点，其实这个目标函数只是为了拟合一对词`$(\boldsymbol{c}, \boldsymbol{w})$`：

`$$
J(\boldsymbol{\theta })  =\mathbb{E}_{\boldsymbol{w} \sim p_{d}} \log p(\mathcal{D}=1|\boldsymbol{w})+k\mathbb{E}_{\boldsymbol{w} \sim p_{n}}\log p(\mathcal{D}=0|\boldsymbol{w})
$$`

对于每组词都要计算期望，即考虑所有候选可能太过奢侈，所以原著进行了第二次近似，也有一些资料是说抽取`$k$`个是蒙特卡洛模拟的一种

`$$
J^{\boldsymbol{c}}(\boldsymbol{\theta }) = \log p(\mathcal{D}=1|\boldsymbol{w}_{0}) + \sum_{i=1}^{k} \log p(\mathcal{D}=0|\boldsymbol{w}_{i})
$$`

那么对于所有的词组该如何建模呢？我们定义一个全局 NCE 进行优化就行了：

`$$
J(\boldsymbol{\theta }) = \sum_{\boldsymbol{c}} p(\boldsymbol{c)}J^{\boldsymbol{c}}(\boldsymbol{\theta })
$$`

## sigmoid 客串

当然，如果你看现在很多机器学习库的实现，你会发现跟上面的式子可能有点不一样？

进行变形一下：

`$$
\begin{align}
p(\mathcal{D}=1|\boldsymbol{w}) &  =  \frac{p_{\boldsymbol{\theta }}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})+kp_{n}(\boldsymbol{w})}  \\
&= \frac{1}{1+ \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})}} \\
&= \frac{1}{1+ \exp\left(\log \left( \frac{kp_{n}(\boldsymbol{w})}{p_{\boldsymbol{\theta }}(\boldsymbol{w})} \right)\right)} \\
&= \frac{1}{1+\exp(\log kp_{n}(\boldsymbol{w})-\log p_{\boldsymbol{\theta }}(\boldsymbol{w}))} \\
&= \frac{1}{1+\exp({\color{red}-}(\underbrace{ \log p_{\boldsymbol{\theta }}(\boldsymbol{w})-\log kp_{n}(\boldsymbol{w})) }_{ x })}
\end{align}
$$`

将里面看成一个参数`$x$`，那么就看到了 sigmoid 函数：

`$$
p(\mathcal{D}=1|\boldsymbol{w}) = \sigma(\log p_{\boldsymbol{\theta}}(\boldsymbol{w})-\log kp_{n}(\boldsymbol{w}))
$$`

同理：

`$$
p(\mathcal{D}=0|\boldsymbol{w}) = \sigma(\log kp_{n}(\boldsymbol{w}) - \log p_{\boldsymbol{\theta}}(\boldsymbol{w}))
$$`

损失函数基本就呼之欲出了：

`$$
\begin{align}
L(\boldsymbol{\theta })  & = - \sum_{\boldsymbol{c}} p(\boldsymbol{c})\left( \log p(\mathcal{D}=1|\boldsymbol{w}_{0})+\sum_{i=1}^{k} \log p(\mathcal{D}=0|\boldsymbol{w}_{i}) \right) \\
&= -\sum_{\boldsymbol{c}} p(\boldsymbol{c}) \left( \log \sigma(\log p_{\boldsymbol{\theta}}(\boldsymbol{w}_{0})-\log kp_{n}(\boldsymbol{w}_{0})) + \sum_{i=1}^{k} \log \sigma(\log kp_{n}(\boldsymbol{w}_{i})- \log p_{\boldsymbol{\theta}}(\boldsymbol{w}_{i})))\right)  \\
&= -\sum_{\boldsymbol{c}} p(\boldsymbol{c})\bigg(\log\sigma(s_{\boldsymbol{\theta }}( \boldsymbol{w}_{0}, \boldsymbol{c} )-\log kp_{n}(\boldsymbol{w}_{0}))+\sum_{i=1}^{k} \log \sigma(\log kp_{n}(\boldsymbol{w}_{i})- s_{\boldsymbol{\theta }}(\boldsymbol{w}_{i}, \boldsymbol{c} ))\bigg)
\end{align}
$$`
因为`$u_{\boldsymbol{\theta}}(\boldsymbol{w}, \boldsymbol{c}) = \exp(s_{\boldsymbol{\theta}}(\boldsymbol{w}, \boldsymbol{c}))$`，所以可以抵消掉`$\log$`；同时因为采用自归一化，所以`$p_{\boldsymbol{\theta}}(\boldsymbol{w}_{0}) = \exp(s_{\boldsymbol{\theta}}(\boldsymbol{w_{0}}, \boldsymbol{c}))/1$`

## 上代码

这里的`$p_{n}$`选的是 log-uniform，类别越往后出现的概率就越小，所以如果要用，可以将类别按照数目进行排序，将多的放在前面，举个例子，类别 A, B, C, D 分别出现的数目为 10, 20, 100, 15，那么类别排序就应该是 C B D A，类别 0 对应的就是 C。range_max 对应的就是类别总数，这里就是 4

`$$
\log_{uniform}(class) = \frac{\log(class + 2) - \log (class + 1)}{\log(range_{max} + 1)}
$$`

下面是训练的 loss，eval 的时候没有 noise，找出 labels 对应的 logits，然后算指标就行了，同时，这里考虑数值稳定性，用 pytorch 官方的 softplus 来取代 logsigmoid，详见[[Numerical Stability#Softplus]]

```python
import math
from einops import repeat
import torch.nn.functional as F
from torch import arange, randn, tensor, log, multinomial

def nce_loss(logits_pos, logits_neg, log_pn_pos, log_pn_neg, k):
    """Compute the noise contrastive estimation loss in
    https://arxiv.org/abs/1806.03664.

    Params:
        - logits_pos: Tensor. Shape: (bs, 1). Logits corresponding to labels.
        - logits_neg: Tensor. Shape: (bs * k, 1). Logits corresponding to sampled classes.
        - log_pn_pos: Tensor. Shape: (bs, 1). Log-probability of labels sampled from noise distribution.
        - log_pn_neg: Tensor. Shape: (bs * k, 1). Log-probability of noise candidates sampled from noise distribution.
        - k: int. The number of noise candidates per training example.

    Note:
        This implementation assumes each context is equally shown which leads to final averge."""

    logk = math.log(k)
    # For numerical stability, replace logsigmoid by the torch softplus
    # for it considers the overflow situation.
    # log(sigmoid(x)) = -softplus(-x)
    # final return also contains minus(-), thus remove all the minus(-)
    pos = F.softplus((logk + log_pn_pos) - logits_pos).mean()
    neg = F.softplus(logits_neg - (logk + log_pn_neg)).mean()
    return pos + neg


def log_uniform(num_sampled, range_max, replacement=True):
    """Sample classes from log-uniform distribution.:
    p(class) = (log(class + 2) - log(class + 1)) / (log(range_max + 1)).
    sampled_classes: [0, range_max).

    Also note that the data distribution should follow the log_uniform.
    e.g., the classes should be in decreasing order of frequencey when in text generation.

    Params:
        - num_sampled: int. The number to be sampled.
        - range_max: int. The number of total classes.
        - replacement: bool. If false, sampled candidates are unique.

    Examples:
        >>> log_uniform(2, 10)
        >>> # tensor([7, 2])
    """
    classes = arange(0, range_max)
    probs = log((classes + 2) / (classes + 1)) / math.log(range_max + 1)
    return probs, multinomial(probs, num_sampled, replacement=replacement)


def main():
    bs, k = 2, 4
    num_classes = 8

    logits = randn(bs, num_classes)
    labels = tensor([2, 4])
    probs, noise_classes = log_uniform(bs * k, num_classes)
    logits_pos = logits.take_along_dim(labels[:, None], dim=1)
    log_pn_pos = probs[labels]
    log_pn_neg = probs[noise_classes]
    logits_k = repeat(logits, '(b 1) h -> (b k) h', k=k)
    logits_neg = logits_k.take_along_dim(noise_classes.reshape(bs * k, -1), dim=1)

    loss = nce_loss(logits_pos, logits_neg, log_pn_pos, log_pn_neg, k)
    print('nce loss: %f' %loss)

if __name__ == '__main__':
    main()
```

至于实验，先鸽一下，留在后面与 info-nce，negative-sampling 等做对比

## References

1. http://proceedings.mlr.press/v9/gutmann10a.html
2. http://arxiv.org/abs/1206.6426
3. https://leimao.github.io/article/Noise-Contrastive-Estimation/
4. https://www.tensorflow.org/api_docs/python/tf/random/log_uniform_candidate_sampler

[^1]: Proxy Problem，指用新的任务或指标来完成对原本任务的建模
[^2]: 现代 NLP 基本都是 token，这里是为了表达简便
