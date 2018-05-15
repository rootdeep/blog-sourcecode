---
title: "发布Gitbook至GitHub个人主页"
draft: false
date: "2018-05-15T19:33:57+08:00"
categories: "github"
tags: ["gitbook, github"]
---

这里不讲如何生成gitbook的html 静态网页，跳过这一步，从如何发布静态网页到GitHub的个人主页(http://{username}.github.io)下说起。即本文的主题是通过域名```http://{username}.github.io/{gitbook_repository}``` 实现访问自己的gitbook 内容。

**准备工作**:
    1. 已经在github 创建了一个名称为{username}.github.io的项目，这个名称格式的项目将是你的个人主页的入口，地址为：http://{username}.github.io.
    2. 已经生成了gitbook 的静态网页。通常在gitbook 的源文件下运行命令```gitbook build``` ,生成的静态网页默认保存位置在```./_book```目录下。
    3. 需要明白```Github Pages``` 概念。具体地简单阅读此链接便可明白：https://help.github.com/articles/configuring-a-publishing-source-for-github-pages/

**方法原理：**

在GitHub上新建一个项目，该项目将会具有两个分支
   - master分支： 用来保存Gitbook的源文件
   - gh-pages 分支：用来保存生成的静态网页
   - 设置gh-pages 分支为 GitHub Pages site

具体步骤：
1. 在GitHub 新建一个仓库，名字就是访问域名中的{gitbook_repository}字段，本例中使用example-book 作为仓库名进行说明。

2. 将该仓库复制到本地（git clone example-book)
   - 为该项目添加.gitignore 文件，将_book 目录添加到忽略文件中去，以便在将源码上传到master 分支时，忽略_book 下的静态网页文件。

3. 为该仓库新建gh-pages 分支(git checkout -b gh-pages)

4. 建立gh-pages 的远程库 git push origin gh-pages

5. 清空分支gh-pages 的内容 (git rm ->git add ->git commit ->git push)

6. 在一个新的目录复制远程仓库的gh-pages 分支并保存为 book-build

   ```git clone -b gh-pages git@github.com:{USERNAME}/example-book.git book-build```

7. 在本地切回master分之后, 在主分支上编写gitbook 的源码

8. 使用```gitbook serve``` 命令后，可以使用localhost:4000 查看生成的book

9. 确认生成book的内容无误后：
   - 将源码推送到远程master分支以便保存
   - 将_book 目录下的内容复制到book-build 目录下，并推送至远程gh-pages  
     分支

10. 确认使用gh-pages分支生成该仓库的主页，具体如下：
  进入项目-->settings -->下拉至Github Pages

  ![github-setttings](/github-setttings.png)

  ![github-setttings](/github-pages.png)

  ​

最后，使用http://{username}.github.io/example-book 域名访问Gitbook

![gitbook](/gitbook.png)



   
