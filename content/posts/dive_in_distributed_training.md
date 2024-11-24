---
title: Diving in distributed training in PyTorch
date: 2022-11-20 10:20:00 +0800
tags: [pytorch, 训练]
math: true
canonicalURL: "https://canonical.url/to/page"
---
鉴于网上此类教程有不少模糊不清，对原理不得其法，代码也难跑通，故而花了几天细究了一下相关原理和实现，欢迎批评指正！代码开源在此：

{{< github name="DL-Tools" link="https://github.com/sherlcok314159/dl-tools" description="Cache effective tools for deep learning. 用以存放深度学习有用的代码片段" language="Python" >}}

**在开始前，我需要特别致谢一下一位挚友，他送了我双显卡的服务器来赞助我做个人研究，否则多卡的相关实验就得付费在云平台上跑了，感谢好朋友一路以来的支持，这份恩情值得一辈子铭记！**

## Why Parallel

我们在两种情况下进行并行化训练[^1]：

1. **模型一张卡放不下**：我们需要将模型不同的结构放置到不同的 GPU 上运行，这种情况叫*ModelParallel(MP)*
2. **一张卡的 batch size(bs)过小**：有些时候数据的最大长度调的比较高（e.g., 512），可用的 bs 就很小，较小的 bs 会导致收敛不稳定，因而将数据分发到多个 GPU 上进行并行训练，这种情况叫*DataParallel(DP)*。当然，DP 肯定还可以加速训练，常见于大模型的训练中

这里只讲一下 DP 在 pytorch 中的原理和相关实现，即 DataParallel 和 DistributedParallel

## Data Parallel

### 实现原理

实现就是循环往复一个过程：数据分发，模型复制，各自前向传播，汇聚输出，计算损失，梯度回传，梯度汇聚更新，可以参见下图[^2]：

{{< figure src="https://s2.loli.net/2022/11/20/cGkHbLx8S7jlpOd.png" caption="[Data Parallel 过程](https://www.cnblogs.com/ljwgis/p/15471530.html)" align="center" width=520px height=440px >}}

pytorch 中部分关键源码[^3]截取如下：

{{< collapse summary="Data Parallel 源码" >}}
```python
def data_parallel(
	module,
	input,
	device_ids,
	output_device=None
):
    if not device_ids:
        return module(input)

    if output_device is None:
        output_device = device_ids[0]

    # 复制模型
    replicas = nn.parallel.replicate(module, device_ids)
    # 拆分数据
    inputs = nn.parallel.scatter(input, device_ids)
    replicas = replicas[:len(inputs)]
    # 各自前向传播
    outputs = nn.parallel.parallel_apply(replicas, inputs)
    # 汇聚输出
    return nn.parallel.gather(outputs, output_device)
```
{{< /collapse >}}

### 代码使用

{{< notice notice-note >}}
因为运行时会将数据平均拆分到 GPU 上，所以我们准备数据的时候， batch size = per_gpu_batch_size \* n_gpus
{{< /notice >}}

同时，需要注意主 GPU 需要进行汇聚等操作，因而需要比单卡运行时*多留出一些空间*

```python
import torch.nn as nn
# device_ids 默认所有可使用的设备
# output_device 默认cuda:0
net = nn.DataParallel(model, device_ids=[0, 1, 2],
                      output_device=None, dim=0)
# input_var can be on any device, including CPU
output = net(input_var)  
```

接下来看个更详细的例子[^4]，需要注意的是被 DP 包裹之后涉及到模型相关的，需要调用 DP.module，比如*加载模型*

{{< collapse summary="DP 例子" >}}
```python
class Model(nn.Module):
    # Our model
    def __init__(self, input_size, output_size):
        super(Model, self).__init__()
        # for convenience
        self.fc = nn.Linear(input_size, output_size)

    def forward(self, input):
        output = self.fc(input)
        print("\tIn Model: input size", input.size(),
              "output size", output.size())
        return output

bs, input_size, output_size = 6, 8, 10
# define inputs
inputs = torch.randn((bs, input_size)).cuda()
model = Model(input_size, output_size)
if torch.cuda.device_count() > 1:
  print("Let's use", torch.cuda.device_count(), "GPUs!")
  # dim = 0 [6, xxx] -> [2, ...], [2, ...], [2, ...] on 3 GPUs
  model = nn.DataParallel(model)
# 先 DataParallel，再 cuda
model = model.cuda()
outputs = model(inputs)
print("Outside: input size", inputs.size(),
	  "output_size", outputs.size())
# assume 2 GPUS are available
# Let's use 2 GPUs!
#    In Model: input size torch.Size([3, 8]) output size torch.Size([3, 10])
#    In Model: input size torch.Size([3, 8]) output size torch.Size([3, 10])
# Outside: input size torch.Size([6, 8]) output_size torch.Size([6, 10])

# save the model
torch.save(model.module.state_dict(), PATH)
# load again
model.module.load_state_dict(torch.load(PATH))
# do anything you want
```
{{< /collapse >}}

如果经常使用 huggingface，这里有两个误区需要小心：

```python
# data parallel object has no save_pretrained
model = xxx.from_pretrained(PATH)
model = nn.DataParallel(model).cuda()
model.save_pretrained(NEW_PATH) # error
# 因为 model 被 DP wrap 了，得先取出模型 #
model.module.save_pretrained(NEW_PATH)
```

```python
# HF实现貌似是返回 N 个 loss （N 为 GPU 数量）
# 然后对 N 个 loss 取 mean
outputs = model(**inputs)
loss, logits = outputs.loss, outputs.logits
loss = loss.mean()
loss.backward()

# 返回的 logits 是汇聚后的
# HF 实现和我们手动算 loss 有细微差异
# 手动算略好于 HF
loss2 = loss_fct(logits, labels)
assert loss != loss2
# True
```

### 显存不均匀

了解前面的原理后，就会明白为什么会显存不均匀。因为 GPU0 比其他 GPU 多了汇聚的工作，得留一些显存，而其他 GPU 显然是不需要的。那么，解决方案就是让其他 GPU 的 batch size 开大点，GPU0 维持原状，即不按照默认实现的*平分数据*

首先我们继承原来的 DataParallel（此处参考[^5]），这里我们给定第一个 GPU 的 bs 就可以，这个是实际的 bs 而不是乘上梯度后的。假如你想要总的 bs 为 64，梯度累积为 2，一共 2 张 GPU，而一张最多只能 18，那么保险一点 GPU0 设置为 14，GPU1 是 18，也就是说你 DataLoader 每个 batch 大小是 32，`gpu0_bsz=14`

```python
class BalancedDataParallel(DataParallel):
    def __init__(self, gpu0_bsz, *args, **kwargs):
        self.gpu0_bsz = gpu0_bsz
        super().__init__(*args, **kwargs)
```

核心代码就在于我们重新分配 chunk_sizes，实现思路就是将总的减去第一个 GPU 的再除以剩下的设备，源码的话有些死板，用的时候不妨参考我的[^6]

{{< collapse summary="修改后的 scatter 代码" >}}
```python
def scatter(self, inputs, kwargs, device_ids):
    # 不同于源码，获取 batch size 更加灵活
    # 支持只有 kwargs 的情况，如 model(**inputs)
    if len(inputs) > 0:
        bsz = inputs[0].size(self.dim)
    elif kwargs:
        bsz = list(kwargs.values())[0].size(self.dim)
    else:
        raise ValueError("You must pass inputs to the model!")

    num_dev = len(self.device_ids)
    gpu0_bsz = self.gpu0_bsz
    # 除第一块之外每块GPU的bsz
    bsz_unit = (bsz - gpu0_bsz) // (num_dev - 1)
    if gpu0_bsz < bsz_unit:
        # adapt the chunk sizes
        chunk_sizes = [gpu0_bsz] + [bsz_unit] * (num_dev - 1)
        delta = bsz - sum(chunk_sizes)
        # 补足偏移量
        # 会有显存溢出的风险，因而最好给定的 bsz 是可以整除的
        # e.g., 总的 =52 => bsz_0=16, bsz_1=bsz_2=18
        # 总的 =53 => bsz_0=16, bsz_1=19, bsz_2=18
        for i in range(delta):
            chunk_sizes[i + 1] += 1
        if gpu0_bsz == 0:
            chunk_sizes = chunk_sizes[1:]
    else:
        return super().scatter(inputs, kwargs, device_ids)

    return scatter_kwargs(inputs, kwargs, device_ids, chunk_sizes, dim=self.dim)
```
{{< /collapse >}}

### 优缺点

- 优点：便于操作，理解简单
- 缺点：GPU 分配不均匀；每次更新完都得销毁**线程**（运行程序后会有一个进程，一个进程可以有很多个线程）重新复制模型，因而速度慢

## Distributed Data Parallel

### 实现原理

1. 与 DataParallel 不同的是，Distributed Data Parallel 会开设多个进程而非线程，进程数 = GPU 数，每个进程都可以独立进行训练，也就是说代码的所有部分都会被每个进程同步调用，如果你某个地方 print 张量，你会发现 device 的差异
2. sampler 会将数据按照进程数切分，**确保不同进程的数据不同**
3. 每个进程独立进行前向训练
4. 每个进程利用 Ring All-Reduce 进行通信，将梯度信息进行聚合
5. 每个进程同步更新模型参数，进行新一轮训练

#### 按进程切分

如何确保数据不同呢？不妨看看 DistributedSampler 的源码

{{< collapse summary="DistributedSampler 的源码" >}}
```python
# 判断数据集长度是否可以整除 GPU 数
# 如果不能，选择舍弃还是补全，进而决定总数
# If the dataset length is evenly divisible by # of replicas
# then there is no need to drop any data, since the dataset
# will be split equally.
if (self.drop_last and len(self.dataset) % self.num_replicas != 0):
    # num_replicas = num_gpus
    self.num_samples = math.ceil((len(self.dataset) -
        self.num_replicas) /self.num_replicas)
else:
    self.num_samples = math.ceil(len(self.dataset) /
        self.num_replicas)
self.total_size = self.num_samples * self.num_replicas

# 根据是否 shuffle 来创建 indices
if self.shuffle:
    # deterministically shuffle based on epoch and seed
    g = torch.Generator()
    g.manual_seed(self.seed + self.epoch)
    indices = torch.randperm(len(self.dataset), generator=g).tolist()
else:
    indices = list(range(len(self.dataset)))
if not self.drop_last:
    # add extra samples to make it evenly divisible
    padding_size = self.total_size - len(indices)
    if padding_size <= len(indices):
        # 不够就按 indices 顺序加
        # e.g., indices 为 [0, 1, 2, 3 ...]，而 padding_size 为4
        # 加好之后的 indices[..., 0, 1, 2, 3]
        indices += indices[:padding_size]
    else:
        indices += (indices * math.ceil(padding_size / len(indices)))[:padding_size]
else:
    # remove tail of data to make it evenly divisible.
    indices = indices[:self.total_size]
assert len(indices) == self.total_size
# subsample
# rank 代表进程 id
indices = indices[self.rank:self.total_size:self.num_replicas]
return iter(indices)
```
{{< /collapse >}}

#### Ring All-Reduce

那么什么是**Ring All-Reduce**呢？又为啥可以降低通信成本呢？

首先将每块 GPU 上的梯度拆分成四个部分，比如$g_0 = [a_0; b_0; c_0; d_0]$，如下图（此部分原理致谢下王老师，讲的很清晰[^7]）：

{{< figure src="https://s2.loli.net/2022/11/20/q72OKSHhmuXYWvN.png" caption="[Ring All-Reduce 1](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

所有 GPU 的传播都是**同步**进行的，传播的规律有两条：

1. 只与自己*下一个位置*的 GPU 进行通信，比如 0 > 1，3 > 0
2. 四个部分，哪块 GPU 上占的多，就由该块 GPU 往它下一个传，初始从主节点传播，即 GPU0，你可以想象跟接力一样，a 传 b，b 负责传给 c

第一次传播如下：

{{< figure src="https://s2.loli.net/2022/11/20/MNWF2dtB7wOsIoK.png" caption="[Ring All-Reduce 2](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

那么结果就是：

{{< figure src="https://s2.loli.net/2022/11/20/4mfqWSMjO3IokxH.png" caption="[Ring All-Reduce 3](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

那么，按照谁多谁往下传的原则，此时应该是 GPU1 往 GPU2 传 a0 和 a1，GPU2 往 GPU3 传 b1 和 b2，以此类推

{{< figure src="https://s2.loli.net/2022/11/20/4mfqWSMjO3IokxH.png" caption="[Ring All-Reduce 4](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

接下来再传播就会有 GPU3 a 的部分全有，GPU0 上 b 的部分全有等，就再往下传

{{< figure src="https://s2.loli.net/2022/11/20/v3jzp4PIQSYERZy.png" caption="[Ring All-Reduce 5](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

再来几遍便可以使得每块 GPU 上都获得了来自其他 GPU 的梯度啦

{{< figure src="https://s2.loli.net/2022/11/20/OA2Ikvxt59YGiVH.png" caption="[Ring All-Reduce 6](https://www.youtube.com/watch?v=rj-hjS5L8Bw)" align="center" width=450px height=250px >}}

### 代码使用

#### 基础概念

第一个是后端的选择，即数据传输协议，从下表可以看出[^8]，当使用 CPU 时可以选择`gloo`而 GPU 则可以是`nccl`

|  **Backend**   | **gloo** |     | **mpi** |     | **nccl** |     |
| :------------: | :------: | :-: | :-----: | :-: | :------: | :-: |
|     Device     |   CPU    | GPU |   CPU   | GPU |   CPU    | GPU |
|      send      |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|      recv      |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|   broadcast    |    ✓     |  ✓  |    ✓    |  ?  |    ✘     |  ✓  |
|   all_reduce   |    ✓     |  ✓  |    ✓    |  ?  |    ✘     |  ✓  |
|     reduce     |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|   all_gather   |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|     gather     |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|    scatter     |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✘  |
| reduce_scatter |    ✘     |  ✘  |    ✘    |  ✘  |    ✘     |  ✓  |
|   all_to_all   |    ✘     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |
|    barrier     |    ✓     |  ✘  |    ✓    |  ?  |    ✘     |  ✓  |

接下来是一些参数的解释[^9]：

|    Arg     |                              Meaning                               |
| :--------: | :----------------------------------------------------------------: |
|   group    | 一次发起的所有进程构成一个 group，除非想更精细通信，创建 new_group |
| world_size |               一个 group 中进程数目，即为 GPU 的数量               |
|    rank    |      进程 id，主节点 rank=0，其他的在 0 和 world_size-1 之间       |
| local_rank |                      进程在本地节点/机器的 id                      |

举个例子，假如你有两台服务器（又被称为 node），每台服务器有 4 张 GPU，那么，world_size 即为 8，rank=[0, 1, 2, 3, 4, 5, 6, 7], 每个服务器上的进程的 local_rank 为[0, 1, 2, 3]

然后是**初始化方法**的选择，有`TCP`和`共享文件`两种，一般指定 rank=0 为 master 节点

TCP 显而易见是通过网络进行传输，需要指定主节点的 ip（可以为主节点实际 IP，或者是 localhost）和空闲的端口

```python
import torch.distributed as dist

dist.init_process_group(backend, init_method='tcp://ip:port',
                        rank=rank, world_size=world_size)
```

共享文件的话需要手动删除上次启动时残留的文件，加上官方有一堆警告，还是建议使用 TCP

```python
dist.init_process_group(backend, init_method='file://Path',
                        rank=rank, world_size=world_size)
```

#### launch 方法

##### 初始化

这里先讲用 launch 的方法，关于 torch.multiprocessing 留到后面讲

在启动后，rank 和 world_size 都会自动被 DDP 写入环境中，可以提前准备好参数类，如`argparse`这种

```python
args.rank = int(os.environ['RANK'])
args.world_size = int(os.environ['WORLD_SIZE'])
args.local_rank = int(os.environ['LOCAL_RANK'])
```

首先，在使用`distributed`包的任何其他函数之前，按照 tcp 方法进行初始化，需要注意的是需要手动指定一共可用的设备`CUDA_VISIBLE_DEVICES`

{{< collapse summary="DDP launch 源码" >}}
```python
def dist_setup_launch(args):
    # tell DDP available devices [NECESSARY]
    os.environ['CUDA_VISIBLE_DEVICES'] = args.devices
    args.rank = int(os.environ['RANK'])
    args.world_size = int(os.environ['WORLD_SIZE'])
    args.local_rank = int(os.environ['LOCAL_RANK'])

    dist.init_process_group(args.backend,
                            args.init_method,
                            rank=args.rank,
                            world_size=args.world_size)
    # this is optional, otherwise you may need to specify the
    # device when you move something e.g., model.cuda(1)
    # or model.to(args.rank)
    # Setting device makes things easy: model.cuda()
    torch.cuda.set_device(args.rank)
    print('The Current Rank is %d | The Total Ranks are %d'
          %(args.rank, args.world_size))
```
{{< /collapse >}}

##### DistributedSampler

接下来创建 DistributedSampler，是否 pin_memory，根据你本机的内存决定。pin_memory 的意思是提前在内存中申请一部分专门存放 Tensor。假如说你内存比较小，就会跟虚拟内存，即硬盘进行交换，这样转义到 GPU 上会比内存直接到 GPU 耗时。

因而，如果你的内存比较大，可以设置为 True；然而，如果开了导致卡顿的情况，建议关闭

{{< collapse summary="加载 DataLoader" >}}
```python
from torch.utils.data import DataLoader, DistributedSampler

train_sampler = DistributedSampler(train_dataset, seed=args.seed)
train_dataloader = DataLoader(train_dataset,
                              pin_memory=True,
                              shuffle=(train_sampler is None),
                              batch_size=args.per_gpu_train_bs,
                              num_workers=args.num_workers,
                              sampler=train_sampler)

eval_sampler = DistributedSampler(eval_dataset, seed=args.seed)
eval_dataloader = DataLoader(eval_dataset,
                             pin_memory=True,
                             batch_size=args.per_gpu_eval_bs,
                             num_workers=args.num_workers,
                             sampler=eval_sampler)
```
{{< /collapse >}}

##### 加载模型

然后加载模型，跟 DataParallel 不同的是需要提前放置到 cuda 上，还记得上面关于设置 cuda_device 的语句嘛，因为设置好之后每个进程只能看见一个 GPU，所以直接`model.cuda()`，不需要指定 device

同时，我们必须给 DDP 提示目前是哪个 rank

```python
from torch.nn.parallel import DistributedDataParallel as DDP
model = model.cuda()
# tell DDP which rank
model = DDP(model, find_unused_parameters=True, device_ids=[rank])
```

注意，当模型带有 Batch Norm 时：

```python
if args.syncBN:
    nn.SyncBatchNorm.convert_sync_batchnorm(model).cuda()
```

##### 训练相关

每个 epoch 开始训练的时候，记得用 sampler 的 set_epoch，使得每个 epoch 打乱顺序是不一致的

关于梯度回传和参数更新，跟正常情况无异

```python
for epoch in range(epochs):
    # record epochs
    train_dataloader.sampler.set_epoch(epoch)
    outputs = model(inputs)
    loss = loss_fct(outputs, labels)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

这里有一点需要小心，这个 loss 是各个进程的 loss 之和，如果想要存储每个 step 平均损失，可以进行 all_reduce 操作，进行平均，不妨看官方的小例子来理解下：

```python
>>> # All tensors below are of torch.int64 type.
>>> # We have 2 process groups, 2 ranks.
>>> tensor = torch.arange(2, dtype=torch.int64) + 1 + 2 * rank
>>> tensor
tensor([1, 2]) # Rank 0
tensor([3, 4]) # Rank 1
>>> dist.all_reduce(tensor, op=ReduceOp.SUM)
>>> tensor
tensor([4, 6]) # Rank 0
tensor([4, 6]) # Rank 1
```

```python
@torch.no_grad()
def reduce_value(value, average=True):
    world_size = get_world_size()
    if world_size < 2:  # 单 GPU 的情况
        return value
    dist.all_reduce(value)
    if average:
	    value /= world_size
    return value
```

看到这，肯定有小伙伴要问，那这样我们是不是得先求平均损失再回传梯度啊，不用，因为，当我们回传 loss 后，DDP 会自动对所有*梯度进行平均*[^10]，也就是说回传后我们更新的梯度和 DP 或者单卡同样 batch 训练都是一致的

```python
loss = loss_fct(...)
loss.backward()
# 注意在 backward 后面
loss = reduce_value(loss, world_size)
mean_loss = (step * mean_loss + loss.item()) / (step + 1)
```

还有个注意点就是学习率的变化，这个是和 batch size 息息相关的，如果 batch 扩充了几倍，也就是说 step 比之前少了很多，还采用同一个学习率，肯定会出问题的，这里，我们进行线性增大[^11]

```python
N = world_size
lr = args.lr * N
```

肯定有人说，诶，你线性增大肯定不能保证梯度的 variance 一致了，正确的应该是正比于$\sqrt{N}$，关于这个的讨论不妨参考[^12]

##### evaluate 相关

接下来，细心的同学肯定好奇了，如果验证集也切分了，metric 怎么计算呢？此时就需要咱们把每个进程得到的预测情况集合起来，t 就是一个我们需要 gather 的张量，最后将每个进程中的 t 按照第一维度拼接，先看官方小例子来理解 all_gather

```python
>>> # All tensors below are of torch.int64 dtype.
>>> # We have 2 process groups, 2 ranks.
>>> tensor_list = [torch.zeros(2, dtype=torch.int64) for _ in range(2)]
>>> tensor_list
[tensor([0, 0]), tensor([0, 0])] # Rank 0 and 1
>>> tensor = torch.arange(2, dtype=torch.int64) + 1 + 2 * rank
>>> tensor
tensor([1, 2]) # Rank 0
tensor([3, 4]) # Rank 1
>>> dist.all_gather(tensor_list, tensor)
>>> tensor_list
[tensor([1, 2]), tensor([3, 4])] # Rank 0
[tensor([1, 2]), tensor([3, 4])] # Rank 1
```

```python
def sync_across_gpus(t, world_size):
    gather_t_tensor = [torch.zeros_like(t) for _ in
                       range(world_size)]
    dist.all_gather(gather_t_tensor, t)
    return torch.cat(gather_t_tensor, dim=0)
```

可以简单参考我前面提供的源码的 evaluate 部分，我们首先将预测和标签比对，把结果为 bool 的张量存储下来，最终 gather 求和取平均。

这里还有个有趣的地方，tensor 默认的类型可能是 int，bool 型的 res 拼接后自动转为 0 和 1 了，另外 bool 型的张量是不支持 gather 的

```python
def eval(...):
    results = torch.tensor([]).cuda()
    for step, (inputs, labels) in enumerate(dataloader):
        outputs = model(inputs)
        res = (outputs.argmax(-1) == labels)
        results = torch.cat([results, res], dim=0)

    results = sync_across_gpus(results, world_size)
    mean_acc = (results.sum() / len(results)).item()
    return mean_acc
```

##### 模型保存与加载

模型保存，参考部分官方教程[^13]，我们只需要在主进程保存模型即可，注意，这里是被 DDP 包裹后的，DDP 并没有 state_dict，这里 barrier 的目的是为了让其他进程等待主进程保存模型，以防不同步

```python
def save_checkpoint(rank, model, path):
    if is_main_process(rank):
    	# All processes should see same parameters as they all
        # start from same random parameters and gradients are
        # synchronized in backward passes.
        # Therefore, saving it in one process is sufficient.
        torch.save(model.module.state_dict(), path)

    # Use a barrier() to keep process 1 waiting for process 0
    dist.barrier()
```

加载的时候别忘了 map_location，我们一开始会保存模型至主进程，这样就会导致 cuda:0 显存被占据，我们需要将模型 remap 到其他设备

```python
def load_checkpoint(rank, model, path):
    # remap the model from cuda:0 to other devices
    map_location = {'cuda:%d' % 0: 'cuda:%d' % rank}
    model.module.load_state_dict(
        torch.load(path, map_location=map_location)
    )
```

##### 进程销毁

运行结束后记得销毁进程：

```python
def cleanup():
    dist.destroy_process_group()

cleanup()
```

##### 如何启动

在终端输入下列命令【单机多卡】

```bash
python -m torch.distributed.launch --nproc_per_node=NUM_GPUS
          main.py (--arg1 --arg2 --arg3 and all other
          arguments of your training script)
```

目前 torch 1.10 以后更推荐用 run

```python
torch.distributed.launch -> torch.distributed.run / torchrun
```

多机多卡是这样的：

```bash
# 第一个节点启动
python -m torch.distributed.launch \
    --nproc_per_node=NUM_GPUS \
    --nnodes=2 \
    --node_rank=0 \
    --master_addr="192.168.1.1" \
    --master_port=1234 main.py

# 第二个节点启动
python -m torch.distributed.launch \
    --nproc_per_node=NUM_GPUS \
    --nnodes=2 \
    --node_rank=1 \
    --master_addr="192.168.1.1" \
    --master_port=1234 main.py
```

#### mp 方法

第二个方法就是利用 torch 的多线程包

```python
import torch.multiprocessing as mp
# rank mp 会自动填入
def main(rank, arg1, ...):
    pass

if __name__ == '__main__':
    mp.spawn(main, nprocs=TOTAL_GPUS, args=(arg1, ...))
```

这里输入参数时务必注意要给定 iterable 形式，比如你的 main 方法除了 rank 还接受一个 int 类型参数，你 spawn 方法时需要以元组形式给定：

```python
if __name__ == '__main__':
    # Note that is (1, ) not (1)
    mp.spawn(main, nprocs=TOTAL_GPUS, args=(1, ))
```

同时因为此种方法是自动给定 rank，我们不需要在初始化时告诉可用的设备有哪些，否则多个进程无法同时启动

```python
def dist_setup_mp(args):
    # there is no need to set CUDA_VISIBLE_DEVICES
    # os.environ['CUDA_VISIBLE_DEVICES'] = args.devices
```

这种运行的时候就跟正常的 python 文件一致：`python main.py`

### 优缺点

- **优点**： 相比于 DP 而言，不需要反复创建和销毁线程；Ring All-Reduce 算法提高通信效率；模型同步方便
- **缺点**：操作起来可能有些复杂，一般可满足需求的可先试试看 DataParallel

### Possible Bugs

{{< notice notice-warning >}}
Producer process has been terminated before all shared CUDA tensors released. See Note Sharing CUDA tensors
{{< /notice >}}

这个应该是用 mp 方法时在 DataLoader 里设置 num_workers 大于导致，应该是 torch 的 multiprocessing 会每一个 num_worker 复制一个模型[^14]

{{< notice notice-warning >}}
RuntimeError: cannot pin ‘torch.cuda.ByteTensor’ only dense CPU tensors can be pinned
{{< /notice >}}

还记得我们前面对于`pin_memory`的解释，是在内存中专门开辟一块空间，专门用于与 GPU 的通信，那么没到 GPU 之前就是 CPU，也就是 CPU tensors 才能被 pin

出现这个 bug 的原因可能是你在 DataLoader 内部处理时已经将张量放到了 GPU 上[^15]，比如自定义`collate_func`

[^1]: https://blog.csdn.net/qq_37541097/article/details/109736159
[^2]: https://www.cnblogs.com/ljwgis/p/15471530.html
[^3]: https://pytorch.org/tutorials/beginner/former_torchies/parallelism_tutorial.html?highlight=dataparallel
[^4]: https://pytorch.org/tutorials/beginner/blitz/data_parallel_tutorial.html
[^5]: https://github.com/kimiyoung/transformer-xl
[^6]: https://github.com/sherlcok314159/dl-tools/blob/main/balanced_data_parallel/README.md
[^7]: https://www.youtube.com/watch?v=rj-hjS5L8Bw
[^8]: https://pytorch.org/docs/stable/distributed.html#backends
[^9]: https://stackoverflow.com/questions/58271635/in-distributed-computing-what-are-world-size-and-rank
[^10]: https://discuss.pytorch.org/t/average-loss-in-dp-and-ddp/93306/4
[^11]: https://arxiv.org/abs/1706.02677
[^12]: https://github.com/Lightning-AI/lightning/discussions/3706
[^13]: https://pytorch.org/tutorials/intermediate/ddp_tutorial.html
[^14]: https://discuss.pytorch.org/t/w-cudaipctypes-cpp-22-producer-process-has-been-terminated-before-all-shared-cuda-tensors-released-see-note-sharing-cuda-tensors/124445/1
[^15]: https://discuss.pytorch.org/t/what-are-dense-cpu-tensor/55703
