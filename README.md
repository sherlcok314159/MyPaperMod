# MyPaperMod 

```md
hugo: v0.117.0-b2f0696cad918fb61420a6aff173eb36662b406e linux/amd64 BuildDate=2023-08-07T12:49:48Z VendorInfo=gohugoio
PaperMod: v8.0
```

This is the demo of my improved PaperMod theme. You can visit the introduction: https://yunpengtai.top/posts/hello-world/ 

该主题基于PaperMod进行改进，下面是详细配置步骤： 

0. 安装好hugo
1. 首先将本仓库clone到本地，然后cd至该目录
2. 输入git submodule update --init，表示拉取themes/PaperMod/下的子模块，里面放的是原始主题PaperMod，版本为
3. 配置好config.yml
4. 在根目录输入hugo server，之后便可在本地端口预览效果了

若想提交至GitHub仓库，则创建`yourname.github.io`，然后输入`hugo`命令生成public文件夹，将public文件夹下的所有内容提交至刚刚创建的仓库，等待片刻即可显示

有问题或者有想PR的feature可以提~