---
title: Going Deeper into Back-Propagation
date: 2022-09-07 18:10:00 +0800
tags: [back-propagation]
math: true
canonicalURL: "https://canonical.url/to/page"
---
### 1. Gradient descent optimization

Gradient-based methods make use of the gradient information to adjust the parameters. Among them, gradient descent can be the simplest. Gradient descent makes the parameters to walk a small step in the direction of the negative gradient.

`$$
\boldsymbol{w}^{\tau + 1} = \boldsymbol{w}^{\tau} - \eta \nabla_{\boldsymbol{w}^{\tau}} E \tag{1.1}
$$`

where `$\eta, \tau, E$` label learning rate (`$\eta > 0$`), the iteration step and the loss function. Wait! But why is the negative gradient?

### 2. Why negative gradient

{{< notice notice-note >}}
The function increases the most sharply by following the direction of the gradient.
{{< /notice >}}

The below is an example. The three-dimensional plane is `$z = F(x, y)$`. The black point is on the plane. You can try to move the point to see how the arrow changes. Interestingly, the arrow always points to the direction which leads to the **biggest** increase of the function value.  Note that when you move one step, the gradient just changes. Thus if you still want to increase the function value in the most sharp way, another computation is needed.

{{< notice notice-tip >}}
The starting point of the arrow is the mapping of the black point to the $xoy$ plane. The arrow is parallel to the gradient.
{{< /notice >}}

<div class='iframe_container'>
<iframe scrolling="no" title="Directional derivatives and Gradient" src="https://www.geogebra.org/material/iframe/id/wd5mrudh/width/800/height/600/border/888888/rc/false/ai/false/sdz/false/smb/false/stb/false/stbh/false/ld/false/sri/false/at/auto" width="600px" height="460px" > </iframe>
</div>

Let us use another graph to better understand what the mapping means. The left graph is contour plot while the right is the plane. The red point is just the mapping of the black point to `$xoy$` plane. The blue arrow is just the direction of the gradient. And you can move the point to feel about it.

<div class='iframe_container'>
<iframe scrolling="no" title="Directional derivatives and Gradient" src="https://www.geogebra.org/material/iframe/id/vpt37qtt/width/788/height/520/border/888888/sfsb/true/smb/false/stb/false/stbh/false/ai/false/asb/false/sri/false/rc/false/ld/false/sdz/true/ctl/false" width="600px" height="460px"> </iframe>
</div>

That is the **intuitive** way to feel about the gradient. Furthermore, we can just try to prove it.
{{< sidenote >}}
from the question in [stackexchange](https://math.stackexchange.com/questions/223252/why-is-gradient-the-direction-of-steepest-ascent)
{{< /sidenote >}}

Consider a Taylor Expansion:

`$$
\begin{aligned}
F(\boldsymbol{r_0} + \boldsymbol{r}) &= F(\boldsymbol{r_0}) + \nabla_{r_0}F \cdot \boldsymbol{r} \\
&= F(\boldsymbol{r_0}) + \|\nabla_{r_0}F\|\cdot \|\boldsymbol{r}\| \cdot \cos \theta
\end{aligned}\tag{2.1}
$$`

When you decide to move a small step, the two magnitudes are certain. If `$\theta=0$`, you can maximize the function value (i.e. **in the direction of gradient**).

Thus if we want to minimize our loss function, we need to go in the opposite direction of the gradient. That is why we need a negative gradient. Also, note that Taylor Expansion only applies to small `$\Delta x$` which requires `$\eta$` to be small (e.g. `$2 \times 10^{-5}, 5 \times 10^{-5}$`).

But how to compute the gradients needs a powerful technique: back-propagation.

### 3. Definition of back-propagation

{{< notice notice-tip >}}
Back-propagation allows information from the cost to then flow backwards through the network, in order to compute the gradients used to adjust the parameters.
{{< /notice >}}

Back-propagation can be new to the novices, but it does exist in the life widely. For instance, the loss can be your teacher's attitude towards you. If you fail in one examination, your teacher can be disappointed with you. Then, he can tell your parents about your failure. Your parents then ask you to work harder to win the examination.

Your parents can be seen as **hidden units** in the neural network, and you are the parameter of the network. Your teacher's bad attitude towards your failure can ask you to make adjustments: working harder. Similarly, the loss can require the parameters to make adjustments via gradients.

### 4. Chain Rule

Suppose `$z = f(y), y = g(x) \implies z = (f \circ g)(x)$`, how to calculate the derivative of `$z$` with respect to `$x$`? The chain rule of calculus is used to compute the derivatives of functions formed by composing other functions whose derivatives are known.

`$$
\frac{dz}{dx} = \frac{dz}{dy} \frac{dy}{dx} \tag{4.1}
$$`

### 5. Case Study

{{< figure src="https://s2.loli.net/2022/09/07/iTFg9de8RyfJavm.png" caption="Fully-connected layer" align="center" width=300px height=150px >}}

Let's first see an important example. In fully connected layers, one input neuron sends information (i.e., multiplied by weights) to every output neuron. Denote `$w_{ji}$` as the weight from `$x_i$` to `$y_j$`. Then for every output neuron (e.g., `$y_j$`), it accepts the information sent by every input neuron:

`$$
y_{j}= \sum\limits_{i} w_{ji} x_{i} \tag{5.1}
$$`

Then the partial derivative of $y_j$ with respect to $x_i$:

`$$
\frac{\partial y_j}{\partial x_{i}}= w_{ji} \tag{5.2}
$$`

Let's see another example which is represented by the computational graph below.
{{< sidenote >}}
Bishop-Pattern-Recognition-and-Machine-Learning-2006
{{< /sidenote >}}

{{< figure src="https://s2.loli.net/2022/09/07/QOJYtIw6BNmsc2i.png" caption="Another example" align="center" width=300px height=200px >}}

And we can perform a forward propagation according to the computational graph.

`$$
\begin{align}
h_{j} &= \sum\limits_{i} w_{ji}^{(1)} x_{i} \tag{5.3} \\
z_{j} &= f(h_{j})     \tag{5.4}      \\
y_{k} &= \sum\limits_{j}w_{kj}^{(2)} z_{j} \tag{5.5}
\end{align}
$$`

where

`$$
f(h) = \tanh(h) = \frac{e^h - e^{-h}}{e^h + e^{-h}} \tag{5.6}
$$`

A useful feature of this activation is that its derivative can be expressed in a particularly simple form:

`$$
f'(h) = 1 - f(h)^2 \tag{5.7}
$$`

The error function can be mean squared errors:

`$$
E(\boldsymbol{w}) = \frac{1}{2} \sum\limits_{k}(y_{k}- \hat{y}_k)^2 \tag{5.8}
$$`

If we want to update the parameters, we need first to compute the partial derivative of `$E(\boldsymbol{w})$` with respect to them.

`$$
\frac{\partial E(\mathbf{w})}{\partial w_{kj}^{(2)}} = \frac{\partial E(\mathbf{w})}{\partial y_{k}} \frac{\partial y_k}{\partial w_{kj}^{(2)}} = (y_{k}- \hat{y}_k)z_j \tag{5.9}
$$`

`$$
\begin{align}
\frac{\partial E(\boldsymbol{w})}{\partial w_{ji}^{(1)}} &= \frac{\partial E(\boldsymbol{w})}{\partial h_{j}}\frac{\partial h_j}{\partial w_{ji}^{(1)}} = \left(\frac{\partial E(\boldsymbol{w})}{\partial z_{j}} \frac{\partial z_j}{\partial h_j}\right)x_{i} \tag{5.10}\\
\end{align}
$$`

`$$
\frac{\partial E(\boldsymbol{w})}{\partial z_j} = \sum\limits_k\frac{\partial E(\boldsymbol{w})}{\partial y_k}\frac{\partial y_k}{\partial z_j}= \sum\limits_k (y_k- \hat{y}_k) w_{kj}^{(2)} \tag{5.11}
$$`

`$\text{Remark.}$` `$z_j$` can send information to all the output neurons (e.g., `$y_k$`), thus we need to sum over all the derivatives with respect to `$z_j$`.

Substituting `$\text{(4.11)}$` into `$\text{(4.10)}$` we obtain

`$$
\frac{\partial E(\boldsymbol{w})}{\partial w_{ji}^{(1)}} = (1 - z_j^2)x_{i} \sum\limits_{k} (y_{k} - \hat{y}_{k}) w_{kj}^{(2)} \tag{5.12}
$$`

### 6. Interpretation

Recall the Taylor approximation of the two variables function:

`$$
f(x, y) = f(x_0, y_0) + f_x (x- x_0) + f_y(y-y_0) \tag{6.1}
$$`

`$\text{Remark.}$` `$(x, y)$` needs to be close to `$(x_0, y_0)$`, otherwise the approximation can fail.

We can transform `$\text{(5.1)}$` into `$\text{(5.3)}$`:

`$$
\begin{align}
f(x,y) - f(x_{0},y_0) &= f_x (x- x_0) + f_y(y-y_0) \tag{6.2}\\\
\implies \Delta f &= f_x \Delta x  + f_y \Delta y\tag{6.3}
\end{align}
$$`
If we apply `$\text{(5.3)}$` in the example above, we can obtain

`$$
\Delta E(\boldsymbol{w}) = \nabla_{\boldsymbol{w}}E(\boldsymbol{w}) \Delta \boldsymbol{w} \tag{6.4}
$$`

From another perspective, a small change in the parameters will propagate into a small change in object function by getting multiplied by the gradient.

{{< notice notice-tip >}}
To summarize, back-propagation allows information to flow backwards through the network. This information can tell the model a small change in one particular parameter can result in what change in the object function. And gradient descent can use this information to adjust the parameters for optimizing the object function.
{{< /notice >}}