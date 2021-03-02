# 前言

写完一篇文章发布到各平台，往往需要把文章中的图片重新上传到对应平台以及排版，整个过程比较耗时耗力。为减少这些不必要的重复操作，本文使用如下工具，实现一键上传图片与排版，一劳永逸。

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/image-20210116203519463.png)

通过 **Typora** 编辑文档，配置使用 **PicGo** 将图片上传至 Github ，将写完的文章（**Markdown** 格式）复制到 **Mdnice** 网址，选择主题即可一键排版。用这个流程写文章，基本上可以做到写完即可发布，不用再单独排版，也不用在微信公众号后台重新上传图片，下文有动图展示。

# Markdown

> 双手不离开键盘即可排版的语言

markdown 用法自行谷歌，本文不做过多阐述

# Typora

> 所见即所得的markdown编辑器

直达 https://www.typora.io/#download，自行下载安装，动图感受下 Typora 的魅力，正文编辑完文字，左侧大纲和展示效果立即更新，做到了真正的所见即所得的效果，并且还支持一键复制粘贴图片（需配置图片上传，下文有教程）

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/typora.gif)



# 图床配置

> 一键上传图片至 Github 仓库，通过 jsDelivr 免费CDN访问图片

## 1. 新建 Github 仓库

创建一个仓库，用来存放图片，比如我创建了imgcdn这个仓库

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/imgcdn.png)

## 2. Github 生成 Token

授权仓库的操作权限，通过API实现自动化上传

打开 https://github.com/settings/tokens/new，填写 `Token` 描述，勾选 repo、write、read 然后点击 `Generate token` 生成一个 `Token` 并保存

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/github_token.png)

## 3. PicGo 图床搭建

打开 https://github.com/Molunerfinn/picgo/releases，下载安装 PicGo ，安装好后开始配置 Github 图床

```
仓库名: 上面创建的存储图片的仓库
分支名: master
Token: 粘贴之前叫你保存的 Token
自定义域名: https://cdn.jsdelivr.net/gh/用户名/仓库名 

(注：jsDelivr 是一个免费、开源的加速CDN公共服务，可以访问 github 仓库中的资源)
```

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/picgo_settings.png)

## 4. Typora 图片上传配置

点击 Typora -> 偏好设置 -> 图片，按照下图标红的地方配置，点击验证图片上传选项检查配置是否无误，配置无误后使用 Typora 编辑文档时，复制粘贴图片时就会自动上传到 github

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/image-20210113230229310.png)

# Mdnice

> markdown 排版利器

使用 https://www.mdnice.com/ 一键排版，并复制到公众号后台，不需要重新上传图片及调整，即可直接发布

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/mdnice.gif)



# 总结

全文流程如下，首次使用需要安装与配置软件，后续写文章只需要第 3, 4, 5 步即可

![](https://cdn.jsdelivr.net/gh/caijinlin/imgcdn/image-20210116201029781.png)

