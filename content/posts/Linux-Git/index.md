---
title: 手动搭建一个 GIT 服务器
date: 2021-07-01 12:06:55
categories: ["technical"]
---

##  前言
想在自己的服务器上搭建一个 Git 仓库，查了一些资料，历经了千辛万苦终于搭建成功了。虽然步骤多，但是其实很简单，只不过如果有一个地方出错就很难进行排查。
<!--more-->
## 具体操作
### GIT 服务器搭建

我们需要在服务器上创建一个新的 git 用户负责仓库的管理

``` bash
# 新增用户
adduser git

# 设置密码
passwd git

# 新建一个文件夹作为远程仓库
mkdir /home/git/warehouse

# 将文件夹权限赋给 git 用户
chown -R git:git /home/git
```



接下来可以切换成 git 用户进行后面的操作了（注：没有安装 git 的话可以看 [这里](https://www.runoob.com/git/git-install-setup.html)，建议使用最新版本）

``` bash
# 切换至 git 账号
su git

cd /home/git/warehouse

# 使用 git 命令生成远端仓库
git init --bare
```



这个 `--bare`参数，代表该仓库为一个”裸仓库“，即这个仓库中不包含”工作区“，只保存 git 历史提交的版本信息，也不允许用户在该仓库中执行任何的 git 操作。裸仓库一般作为远端仓库使用，其仓库文件树如下：

``` bash
.
├── branches
├── config
├── description
├── HEAD
├── hooks
│   ├── applypatch-msg.sample
│   ├── commit-msg.sample
│   ├── fsmonitor-watchman.sample
│   ├── post-update.sample
│   ├── pre-applypatch.sample
│   ├── pre-commit.sample
│   ├── pre-merge-commit.sample
│   ├── prepare-commit-msg.sample
│   ├── pre-push.sample
│   ├── pre-rebase.sample
│   ├── pre-receive.sample
│   └── update.sample
├── info
│   └── exclude
├── objects
│   ├── info
│   └── pack
└── refs
    ├── heads
    └── tags
```



到此为止远端仓库就可以正常使用了，例如在本地执行`git clone git@xx.xx.xxx.xx:/home/git/warehouse`即可开始克隆

这段代码中第二个 git 指的是 Linux 中的 git 用户，`@`后面则跟的是服务器的 IP 地址，以及仓库地址。

### SSH免密登录

在仓库搭建成功之后，发现每次拉取和推送其实都需要输入 git 用户的密码，这样其实会很麻烦，于是开始研究使用 SSH 密钥免密登录的方法。

在配置文件 `/etc/ssh/sshd_config`中，有一栏是`AuthorizedKeysFile	.ssh.authorized_keys`

从这一项我们可以得知，在客户端发起 **SSH** 连接请求之后，系统会去该用户目录下的`.ssh/authorized_keys`文件中查询对应的密钥。



回到 git 用户，我们可以在`home/git`目录下，输入`ll -a`查看是否有`.ssh`文件，如果没有的话，可以新建并生成`authorized_key`文件

``` bash
cd /home/git

# 创建 .ssh 文件
mkdir .ssh

cd .ssh

# 创建新的 authorized_key 文件
touch authorized_key
```



接着修改`.ssh`和`.authorized_key`的权限，这一步**非常重要**，文件权限的过大或者过小，都可能会导致 SSH 连接失败

``` bash
chmod 700 /home/git/.ssh

chmod 600 /home/git/.ssh/authorized_key
```

这个时候就可以往`authorized_key`文件中添加公钥，一行一个。



密钥的匹配有很多种解决方案，例如在本地生成一个密钥对，公钥放在服务器上。而我的本地主机已经有一个正在使用的密钥了，所以我的做法是在云服务器上生成密钥，将公钥放入`authorized_key`中，私钥储存在本地

``` bash
cd /home/git/.ssh

# 生成密钥
ssh-keygen -t rsa -b 2048 -f gitKey -C "linux git ssh key"

# 将公钥写入 authorized_key 中
cat id_rsa.pub >> authorized_key
```



接着将生成的私钥也就是`id_rsa`下载至本地，记得重命名一下，避免和本机原有私钥名冲突，例如将它重命名为`linux_git_rsa`

之后将它放在`C:\Users\(你的电脑用户名)\.ssh`下。接着我们需要让这个新的私钥加入到 SSH 连接匹配密钥队列中，在本地的`.ssh`文件中新建一个`config`文件并编辑

``` bash
Host *
	# 原有的密钥文件
	IdentityFile ~/.ssh/id_rsa
	
	# 服务器的密钥文件
    IdentityFile ~/.ssh/linux_git_rsa
```



完成之后再进行 git 操作，就不需要输入密码了。如果 SSH 连接不成功，请**仔细检查**以下内容：

1. 密钥是否匹配
2. `.ssh`和`authorized_key`的文件权限是否正确
3. `/etc/ssh/sshd_config`中，是否正确设置了允许用户密钥登陆，也就是`PubkeyAuthentication	yes`
4. `/etc/ssh/sshd_config`中，是否正确设置了密钥的存放地址，也就是`AuthorizedKeysFile	.ssh.authorized_keys`
5. 密钥是否正确存入`authorized_key`文件中
6. 本地是否配置了`config`文件并正确设置了私钥的位置

以上都是我的痛苦配置过程中总结出的经验，当时配置 SSH 连接一直不成功，总是需要密码登录。

在搜索引擎上泡了一天，最后才发现是在`AuthorizedKeysFile	.ssh.authorized_keys`这一项中配置错了位置。

