---
title: 新的主题
date: 2022-06-19 11:10:00 +0800
lastmod: 2024-11-20 18:00:00 +0800
tags: [网站，折腾]
showToc: true
math: true
comments: true
ShowCopyRight: true
showLastMod: true
canonicalURL: "https://canonical.url/to/page"
UseHugoToc: true
---
该网站所用所有源码均在此仓库，欢迎使用：

{{< github name="MyPaperMod" link="https://github.com/sherlcok314159/MyPaperMod" description="This is the demo of my improved PaperMod theme. You can visit the introduction: https://yunpengtai.top/posts/hello-world/" language="HTML" >}}

{{< quote >}}
这是基于 Hugo 系列主题第一篇文章，因为之前是在 Jekyll 上进行渲染，故而 Hello World 也有更新
{{< /quote >}}

### 为啥变动

那么为啥从 Jekyll 变到 Hugo 呢？原因其实有几点：

1. 之前用的主题看着不好看，感觉很拥挤，之前想改没空，想要个简洁干净，专注于内容的主题
2. Jekyll 在服务器端的渲染速度实在是不快（本地却很快，不懂）
3. Hugo 的设计更符合我的直觉，而且方便自定义好玩的功能，比如 shortcode 功能

主要是被另一位博主 [Li’s Blog](https://lilianweng.github.io/) 用的主题吸引，就立马换成了新的主题 [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，然而这个主题虽然简洁，但是少了一些我认为必须得有的功能，比如渲染公式，侧边目录等。于是任着喜欢「折腾」的性子，从零开始学 Hugo 的语法，然后四处借鉴学习，花了两天时间添加了一些功能，当然，这篇文章主要介绍相关的 feature，不会涉及到具体的实现

没有一个主题能完全满足个人的需求，还是从底层原理出发，这样才能「自由自在」地进行更改

### 支持 features

基于的主题 PaperMod 的相关 features 便不在继续介绍，介绍一些我加入的特性

1. 公式渲染：这个主题一开始是没有公式渲染功能的，我通过引入 [MathJax](https://www.mathjax.org/) 来完成这一点，行内公式如：`$a+b=c$`，行间公式效果如下：
`$$
e^{\pi i} + 1 = 0
$$`
然而，Hugo 引擎本身是有些问题，有些时候多行公式没办法渲染成公式，此时需要先将公式用代码 block 框起来，等网页渲染好之后再去掉代码 block，开始渲染公式
{{< sidenote >}}
借鉴了 [yihui](https://github.com/yihui/hugo-lithium/blob/master/static/js/math-code.js) 的实现
{{< /sidenote >}}
2. 评论功能：网上关于评论系统的实现大致分为三种：GitHub 为基础，Disqus 这种官方托管的，以及自我进行托管的。网站之前是用了第三种的 [Waline](https://waline.js.org/)，后来换到了 [Artalk](https://artalk.js.org/)，因为后者可以在评论中使用数学公式以及提供邮箱提醒。因为个人比较喜欢玩表情，还在 Artalk 中加入了一些好玩的表情。
3. 侧边目录：目录在原主题中是固定放在文章顶部，这样不利于读者对于长文整体的把握，故而将目录移动到了侧边
{{< sidenote >}}
借鉴了 [sulvblog](https://github.com/xyming108/sulv-hugo-papermod/blob/main/assets/css/extended/toc.css)
{{< /sidenote >}}
，并且修改了一些 css 设置，自动高亮当前目录并且给它添加下划线，这样可以清楚地发现此时的章节。
4. MarginNote
{{< sidenote >}}
借鉴了 [kennethfriedman](https://kennethfriedman.org/thoughts/2019/marginal-notes/) 和 [scripter](https://scripter.co/sidenotes-using-only-css/)
{{< /sidenote >}}
：以往 markdown 都是仅支持脚注，这样不利于文章的阅读，查看相关引用信息需要移至页面底部再返回，而 MarginNote 则给了作者更多自由的空间和方便读者阅读
而且移动至相关的标注会自动高亮对应的 MarginNote，更加方便读者阅读；同时过长会自动转行，方便进行较长的标注
{{< sidenote >}}
Sidenotes give more life and variety to the page and are the easiest of all to find and read, If carefully designed, they need not enlarge either the page or the cost of printing it.
{{< /sidenote >}}
5. 代码渲染：舍弃了 PaperMod 内置的 an-old-hope.min，重新选择了 [atom-one-dark/light](https://github.com/highlightjs/highlight.js/blob/main/src/styles/atom-one-light.css) 系列
{{< collapse summary=示例代码 >}}
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
```
{{< /collapse >}}
6. 更新时间：原先主题根本没有更新时间的设置，这样不利于读者查看文章最新的时间，文章的开头便有更新的时间，可以查看
7. 著作权声明：现在网上各种抄袭成风，需要对自己的文章进行声明，添加至文章末尾，例如本文用的是 CC BY-NC-SA 4.0
{{< sidenote >}}
Attribution-NonCommercial-ShareAlike 4.0 International
{{< /sidenote >}}
8. 字体：原来的字体看着有点不舒服，这里将中文换成了[霞鹜文楷](https://github.com/chawyehsu/lxgw-wenkai-webfont)，观感很不错，英文的话是在几个字体中选一个：-apple-system, BlinkMacSystemFont, segoe ui, Roboto, Oxygen, Ubuntu, Cantarell 这样看起来也很舒服，选取 Lorem Ipsum 进行测试
{{< sidenote >}}
Just dummy text
{{< /sidenote >}}
{{< quote >}}
Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry’s standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book.
{{< /quote >}}
9. 中英文空白：汉学家称中文和英文之间的空白字元为「盘古之白」，大抵是因为劈开了全形字和半形字之间的混沌。在书写时适当地留白会提升观感，这个是通过[盘古的 js](https://cdn.bootcss.com/pangu/4.0.7/pangu.min.js)来实现的，手敲空格噼里啪啦的，倒不是很方便
10. notice：通过 shortcode 添加了各种的提示框
{{< sidenote >}}
借鉴了 [Hugo-notice](https://github.com/martignoni/hugo-notice)
{{< /sidenote >}}
，看起来很不错，比如：
{{< notice notice-info >}}
一生疏狂尽余欢，半剖肝胆入剑寒。 剑至高危如蜀道，生逢穷途行路难。
{{< /notice >}}
{{< notice notice-warning >}}
一生疏狂尽余欢，半剖肝胆入剑寒。 剑至高危如蜀道，生逢穷途行路难。
{{< /notice >}}
{{< notice notice-tip >}}
一生疏狂尽余欢，半剖肝胆入剑寒。 剑至高危如蜀道，生逢穷途行路难。
{{< /notice >}}
{{< notice notice-note >}}
一生疏狂尽余欢，半剖肝胆入剑寒。 剑至高危如蜀道，生逢穷途行路难。
{{< /notice >}}
11. 添加了友链功能
{{< sidenote >}}
借鉴了 [sulvblog](https://github.com/xyming108/sulv-hugo-papermod/blob/main/layouts/shortcodes/friend.html)
{{< /sidenote >}}
，这样方便添加别人的博客，加强朋友之间的联系，可查看主页的 [Friends](/friends)
12. 对于 markdown 引用的改进，原来的引用比较丑，下面是改进后的引用：
{{< sidenote >}}
借鉴了 [Guan Qirui](https://guanqr.com/tech/website/hugo-shortcodes-customization/#quote)
{{< /sidenote >}}
{{< quote >}}
Basically, I’m not interested in doing research and I never have been… I’m interested in understanding, which is quite a different thing. And often to understand something you have to work it out yourself because no one else has done it. — David Blackwell
{{< /quote >}}
13. 添加了 GitHub 仓库的小卡片，比较美观，用了本网站的源码仓库作示例：
{{< github name="MyPaperMod" link="https://github.com/sherlcok314159/MyPaperMod" description="This is the demo of my improved PaperMod theme. You can visit the introduction: https://yunpengtai.top/posts/hello-world/" language="HTML" >}}
14. 点击图片进行放大，对于图片大，缩放多的情况，点击放大是很有必要的，不妨试试看：
{{< figure src="https://s2.loli.net/2022/06/19/ztF8gu1RnYJW4KV.jpg" caption="[Black Holes: Monsters in Space](https://www.nasa.gov/image-article/black-holes-monsters-space/)" align="center" width=600px height=300px >}}


## 致谢

尽管上文对所用工具均已写明出处，但是这些精妙绝伦的工具值得再单独列出来致谢，同时也方便读者自己 DIY 时进行选择性添加：

1. [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，这是一开始基于的主题，感谢作者的开源
2. 评论系统的支持，感谢 [Artalk](https://artalk.js.org/)，做的很干净
3. 关于对代码主题的更改，主要借鉴了 [HightLight.js](https://cdnjs.com/libraries/highlight.js) 官方的 demo
4. 公式的渲染方面 [MathJax](https://www.mathjax.org/) 做的很 nice
5. 侧边目录和一些 shortcode 的实现借鉴了 [Sulv’s Blog](https://www.sulvblog.cn/) 以及 [Guan Qirui](https://guanqr.com/)
6. 字体还得感谢 [霞骛文楷](https://github.com/chawyehsu/lxgw-wenkai-webfont)，比那些收费的丑字体好太多了
7. 盘古之白是通过 [pangu.js](https://cdn.bootcss.com/pangu/4.0.7/pangu.min.js) 实现的，点赞
8. MarginNote 的实现借鉴了 [kennethfriedman](https://kennethfriedman.org/thoughts/2019/marginal-notes/) 和 [scripter](https://scripter.co/sidenotes-using-only-css/)
9. 图片放大主要用了 [fancybox](http://fancybox.net/) 的功能，用起来也比较方便