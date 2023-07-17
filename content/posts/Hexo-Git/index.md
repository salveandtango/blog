---
title: 使用 GIT 来更新你的博客
date: 2021-11-1 10:38:33
categories: ["technical"]
---

## 前言
博客之前是使用 code-server 来进行编辑并更新的，不过可能是因为我的服务器太过辣鸡，进入 code-server 服务之后总是卡在加载拓展的进程中，一等就是十几分钟半个小时，基本没办法使用。于是便转为使用 Git 来更新博客。
## 具体操作
我的博客是使用 [Hexo](https://hexo.io/zh-cn/index.html) 来搭建的，博客文章和图片资源都储存于`_posts`文件夹下，从这个信息出发，我们可以考虑将该文件夹下的所有内容加入到 git 仓库，这样就可以拷贝至本地并进行更新。大致的操作步骤如下：

1. 搭建 git 服务，创建远端仓库
2. 同步远端仓库和 Hexo 博客的 `_posts`文件夹
3. 更新博客

### Git Hooks

那么首先我们需要在服务器上搭建一个 git 服务，具体教程可以看之前的一篇 [文章](https://unboiledwater.top/2021/10/29/Linux-Git)。值得注意的一点是，**在远端仓库中是看不到上传的文件的**，所以必须在本地 push 之后，远端仓库通过调用 git hooks 将文件 check out 出来。我们进入远端仓库的目录，可以看到有一个`hooks`文件夹，目录如下：

``` bash
.
├── applypatch-msg.sample
├── commit-msg.sample
├── fsmonitor-watchman.sample
├── post-update.sample
├── pre-applypatch.sample
├── pre-commit.sample
├── pre-merge-commit.sample
├── prepare-commit-msg.sample
├── pre-push.sample
├── pre-rebase.sample
├── pre-receive.sample
└── update.sample
```

这些文件代表了 git 在执行了不同操作后会调用的脚本，例如`applypatch-msg.sample`这个文件就是当`git am`命令调用时被启用，详情可以查看 git 官网的 [说明](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)，介绍了各个脚本的作用。我们这里需要用到的是`post-receive`这个脚本，**如果没有可以新建一个**。我们来看一下官网对这个脚本作用的介绍——

> `post-receive` 挂钩在整个过程完结以后运行，可以用来更新其他系统服务或者通知用户。 它接受与 `pre-receive` 相同的标准输入数据。 它的用途包括给某个邮件列表发信，通知持续集成（continuous integration）的服务器， 或者更新问题追踪系统（ticket-tracking system） —— 甚至可以通过分析提交信息来决定某个问题（ticket）是否应该被开启，修改或者关闭。 该脚本无法终止推送进程，不过客户端在它结束运行之前将保持连接状态， 所以如果你想做其他操作需谨慎使用它，因为它将耗费你很长的一段时间。

### 脚本编写

在明确了脚本的作用之后我们就可以知道，远端仓库在客户端 push 结束之后可以通过编写 post-receive 脚本来将仓库内容拉取到指定文件。命令很简单，假设有文件`/home/git/check_out`，那么下面的命令就可以将文件。

``` bash
git --work-tree = /home/git/check_out -fd

git --work-tree = /home/git/check_out checkout --force
```

再通过拷贝命令与 Hexo 博客文件下的`_posts`文件同步、执行启动/构建项目的命令，就可以更新和发布博客了。而为了方便在本地编写文章时图片也能正确预览，我将`.md`文件放入了图片文件的下级文件夹，在本地，文件路径是这样的

``` bash
├── Picture-floder-A
├── Picture-floder-B
├── Picture-floder-C
├── md
│   ├── a.md
│   ├── b.md
│   ├── c.md
```

最终`post-receive`的脚本文件是这样的，完成之后，在本地编写博客后直接 push 即可。

``` bash
git --work-tree = /home/git/check_out -fd

git --work-tree = /home/git/check_out checkout --force

cd /home/git/check_out

rm -rf /home/git/blog/source/_posts/*

# 因为 hexo 图片路径的问题，需要将 .md 文件和图片文件夹放在同一级中
# 分两次将图片文件夹和 md 文件夹中的 .md 文件拷贝至 _post
# 拷贝并排除文件夹的语法为：
# ls x1/ | grep -v x2 | xargs -i cp -r x1/{} x3/    //x1为源路径， x2为欲排除的文件/目录，x3为目标路径

# 拷贝图片文件夹
cp -r `ls ~/check_out | grep -v md | xargs` /home/git/blog/source/_posts/

# 拷贝 md 文件夹下的所有 .md 文件
cd /home/git/check_out/md/
cp -r * /home/git/blog/source/_posts/

# 构建
cd /home/git/blog
hexo clean && hexo g

# 拷贝静态文件至 nginx 目录下 （这里可能需要额外设置用户权限，不再赘述）
\cp -r /home/git/blog/public/* /usr/share/nginx/html
```



