---
title: Tips for Training Neural Networks
date: 2022-07-30 11:10:00 +0800
tags: [训练, 神经网络]
math: true
showCopyRight: false
canonicalURL: "https://canonical.url/to/page"
---
Recently, I have read a [blog](http://karpathy.github.io/2019/04/25/recipe/) about training neural networks (simplified as NN in the rest part of this post) and it is really amazing. I am going to add my own `experience` in this post along with `summarizing` that blog's interesting part.

Nowadays, it seems like that training NN is extremely easy for there are plenty of free `frameworks` which are simple to use (e.g. PyTorch, Numpy, Tensorflow). Well, training NN is easy when you are `copying` others' work (e.g. reproducing a BERT) because everything is there for you. However, when designing a NN or facing a new task, you are most probably trapped somewhere. 

And this blog is meant to guide you to handle new problems.

Let's first begin with some basic rules. Hope you guys enjoy it!

1. <b>Rush into training neural networks leads to suffering.</b> Training NN is not like writing the common code. For instance, if you plug in a int while it needs a string, errors just come out. However, writing the code about NN can not be so easy for it won't show you bugs `automatically` (only if you make big mistakes). 
2. <b>Sweating the details always pays off.</b> Someone may say the details are `infinite` and can stop you from marching. Note that the details mentioned here are all necessary instead of some `trivialities`. And sweating the details can reduce your pain. 
3. <b>Observation leads to intuition. </b> Sadly, if you just keep thinking about something, inspiration will never come to you. For instance, if you want to upgrade the algorithm, you had better observe the data where the algorithm fails instead of just thinking about the algorithm.
4. <b>Trusting your intuition instead of your implementation. </b> Sometimes when you come up with a new idea, the implementation of it may go wrong to some degree. When the result is `opposite` to your `assumption`, always check your code first before doubting your idea.
5. <b>Quicker, then better.</b> When you are trying to test your hypothesis, use the most `efficient` way to verify it as `fast` as possible.

Then let us go through concrete parts.

### Familiar with Data

At this stage, you need to do some basic analysis and data mining. Assume we have a `classification` dataset in NLP. There are several aspects to think about.
- <b>The Distribution.</b> To begin with, you need to know the `label` distribution especially and `visualize` it. For instance, if you observe `long-tailed distribution` (e.g. the number of the instances for good emotion is 900 while for the bad emotion is 10), then some methods such as data augmentation or cost-sensitive loss functions can play their part. For your interest, you can refer to this up-to-date [survey](https://arxiv.org/pdf/2110.04596.pdf). Similarly, you may also need to know the distribution of the `length` of sequence. Thus you can set the appropriate maximum sequence length for your task. Moreover, you can also pass the original data through the `feature extractor` (such as BERT) to gain their representation. Then you can `cluster` them.
- <b>Greed is good.</b> I strongly suggest that you `look through` the whole dataset ambitiously just like the `greedy algorithm`. And I promise you are bound to find surprise. You can have a whole understanding of the `domain` of this dataset. And you can choose appropriate `pre-trained` models according to the domain. Also, remember to understand the `labels`. Once you understand the labels, you can see if the annotation is `contradictory`. And you can select certain samples to see the annotation and estimate how noisy the whole dataset is. You may also need to think for the `machine`. Are the sequences easy to understand? If they are easy to understand, then we do not need to apply very complicated models to tackle this problem. Is the `local` information more important than the `global`? Your understanding about the dataset can help you figure out some basic modeling problems and offer you intuition about `rule-based` methods.
- <b>Simple quantification.</b> You may need to know the `size` of the dataset. If the size is `small`, we can use the simple models such as textCNN or FastText instead of BERT-based models for the complicated models need more data to model the inductive bias. Also, you can write simple code for detecting the `duplicate` instances and instances which are `corrupted` (e.g. lack of the label).
- <b>Stand on the shoulder of the model.</b> When your model is trained on the dataset, you can see how it performs on the `dev/eval` set. You need to pay attention to those `mis-predicted` instances (i.e. bad cases) and think about why the prediction is wrong. Is the label wrong or the way of modeling weak to capture these information.
- <b>Filter / Sort / Clean</b>. You can decide whether to filter or clean the dataset based on your thorough observation.

### End-to-End Pipeline

When you finish observing the dataset, you need to build the simple pipeline to ensure everything goes well.
- <b>Fix the random seed.</b> When carrying out the experiments, you had better fix the seed to reduce the influences of `randomness` on the experiments.
- <b>As simple as possible.</b> When building the pipeline, you do not have to use very complicated modeling methods, etc. We are just testing. Thus make everything as simple as possible. For instance, you can use the simple classifier such as SVM and MLP (Multi-Layer Perceptron). 
- <b>Record the accuracy and loss.</b> Training accuracy and loss are very useful for you to figure out which `difference` is beneficial. Also, we do not need complicated tools (e.g. Tensorboard and Wandb) to do so. You can use a list to store things you want and visualize it by matplotlib or write it down in a `txt` (Sometimes, the data on the terminal can disappear for certain reasons).   
- <b>Track the progress.</b> In python, you can simply use the `tqdm` to track the progress. And you can also add the immediate accuracy and loss on the progress bar. Believe me, this can reduce your anxiety. 
- <b>Verify the init loss.</b> For the multi-label classification problem, its loss should equal `$-\log (1/ \text{num classes})$` (with a softmax). For instance, if you need to make the true prediction among 4 labels, the init loss should equal $1.386$. 
- <b>Good Initialization.</b> For regression problems, if the `average` of your data is 6, then you can initialize the bias as 6 which can lead to `fast` and stable convergence. One more example, if you want to initialize the weights and you do not want the weights to be influenced by the `output shape`, then you may prefer Lecun Normal to Glorot Normal (all initialize with `$\text{N(0, scale)}$`). Also, `normal` initialization is better than uniform initialization by experience. Last but not least, when facing an `imblanced` dataset with ratio 1:10 (positive cases V.S. negative cases), set the bias on the `logits` so that the model can learn the bias with the first few iterations.
	```python
	# fan_in, fan_out represent the input and output shape
	scale = 1.
	# lecun normal
	scale_lecun /= max(1., fan_in)
	scale_lecun = sqrt(scale_lecun)
	
	# glorot normal
	scale_glorot /= max(1., fan_in + fan_out)
	scale_glorot = sqrt(scale_glorot)
	```
- <b>Human Baseline.</b> If the dataset is very particular and there are few related evaluation methods, you had better set the `human baseline` in sampled instances. Compared to the human baseline, you can have an idea that where your model has gone.
- <b>Input-Independent Baseline.</b> You can set the input all `zeros` and see the performance. And it should be `worse` than the performance of plugging in your data. 
- <b>Overfit a small batch.</b> The model should `overfit` a batch of few instances (e.g. 10 or 20). Theoretically speaking, you should achieve the least loss. If the model fails to do so, then you can go and find the foxy bug.
- <b>Visualize the input before going into the NN.</b> Take Google's [code](https://github.com/google-research/bert) as example, it shows the input tensors when performing classification problems by BERT. This habit has saved me many times when coming up with a brand-new task because the preparation of data can be hard to some degree. 
- <b>Visualize the predictions dynamically over the course of training.</b> By doing so, we can have a direct picture about where the model has gone and how it performs. 
- <b>Try Back-Propogation yourself.</b> `Gradients` can give you information about what the model depends on to make such predictions. 
- <b>Generalize a special case.</b> You should not write the general functions at the beginning because your thoughts can be easy to change, thus these general functions are fragile. You can generalize a special case when you are sure that it won't change a lot.

### Overfitting
Since we have built a pipeline and tested it, it's time for us to make the model overfit the whole dataset.
- <b>Picking the right model.</b> The selection of models is related to the size of the dataset and complexity of the task. If your dataset is small, you can choose relatively big models to overfit.
- <b>Borrow experience from the giants.</b> Sometimes you are unfamiliar with the task, you may have no idea about which `hyper-parameter` to choose (e.g. learning rate). Then you can search some related papers and see the `appendix` for training details.
- <b>Carry out many controlled experiments.</b> Deep Learning is a `experimental subject`. Sometimes observation fails to give you idea about what exactly it will perform. For instance, if you want to know which learning rate is most suitable for this task, try more options to select the best. Remember change a variable `once a time` to reduce the influence of mixture. 
- <b>Turn to tricks.</b> Tricks are infinite. For instance, you can apply adversarial training like FGM or PGD to improve the model's performance. Also, if permitted, you can use `random` searching for the best parameters. 

### Regularize
- <b>More data is better.</b> The most effective way to regularize the model is collecting more real-world data for training. After all, we are using the `small` set of data to `approximate` the distribution of the real-world. 
- <b>Data Augmentation.</b> If you `lack` data, you can apply data augmentation to increase your data. Although this method seems very easy, it does `demand` your thorough understanding of your data and task. And `creative` methods can always pay off. For instance, in NLP, you can use back-translation to augment.
- <b>Cross Validation.</b> You can split the data several times. And use separate data to train some models. Then we can ensemble them to gain the final prediction. 


### Others
- <b>Always remember to record your results in a good order.</b> For instance, you must record all the parameters and the model's performances at the dev/eval set. You had better record the `motivation` for you trying out this experiment.
- <b>Always back up your code and data.</b> When you are trying some new methods, do not just try it on the original code. The same for the data. You need to back up the original pipeline and data for bad things happening.