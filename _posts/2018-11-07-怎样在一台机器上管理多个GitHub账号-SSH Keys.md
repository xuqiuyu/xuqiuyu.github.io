---
layout:     post
title:      怎样在一台机器上管理多个GitHub账号 SSH Keys
subtitle:   GitHub SSH Keys 多账号
date:       2018-11-07
author:     Breeze
header-img: img/multipleGitHubAccounts.png
catalog: true
tags:
    - GitHub MutipleAccounts
---

## 前言

随着gitflow工作流的广泛使用，越来越多的开发者可能同时以多种不同的身份参与到不同Git项目中。最常见的一种情况是：作为公司员工，我会有一个公司邮箱注册git账号，同时我也会有一个个人的git账号，当我们在同一台机器上完成不同project的commit时，需要用不同的git账号push到对应的远程分支，但是我们如何管理不同的project对应不同的account呢 ？这篇文章也许能帮你解决这个困扰。


## Mac 和 Windows 原理相同

>本文以Mac为例

### 生成 SSH Keys

在生成 SSH keys 之前，我们可以检查一下系统中是否已经存在 SSH keys: ls -al ~/.ssh 这个命令会列出所有共私有keys对。
如果 ~/.ssh/id_rsa 存在，我们可以直接使用它，否则我们可以通过命令： ssh-keygen -t rsa 手动生成一个key到默认的 ~/.ssh/id_rsa中。
若该交互命令要求指定一个key的存储位置，直接回车默认即可。一个私钥key和公钥key ~/.ssh/id_rsa.pub将会被生成在 ~/.ssh/.目录中。
然后，我们为个人账号使用默认的key pair。工作账号，还需要再生成不同的SSH keys。下面的命令会生成SSH keys并讲公司邮箱email@work_mail.com 作为公钥key tag保存到 ~/.ssh/id_rsa_work_user1.pub
```sh
 $ ssh-keygen -t rsa -C "email@work_mail.com" -f "id_rsa_work_user1"
```
现在我们已经生成好的两对不同的 SSH Keys w
```
 ~/.ssh/id_rsa
 ~/.ssh/id_rsa_work_user1
```

### 分别将SSH key添加到不同的GitHub账号
我们已经生成好了SSH public keys，接下来要让我们的GitHub信任我们的keys。
复制public key: pbcopy < ~/.ssh/id_rsa.pub，然后用对应的账号登陆到GitHub网站进行以下配置：
1. Go to Settings
2. Select SSH and GPG keys from the menu to the left.
3. Click on New SSH key, provide a suitable title, and paste the key in the box below
4. Click Add key — and you’re done!
工作账号设置同上。


### 使用 ssh-agent 注册新创建的 SSH Keys
要使用ssh keys,我们首先得在机器上注册它。使用 eval "$(ssh-agent -s) 命令确保 ssh-agent 处于运行状态。
添加 ssh keys 到 ssh-agent: 
```
ssh-add ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa_work_user1

```
要求 ssh-agent 对不同的SSH Hosts使用各自的ssh keys.

接下来是至关重要的两步：创建并使用 SSH 配置文件并确保同一时间只有一个SSH key活跃在ssh-agent中。

### 创建 SSH 配置文件
这里其实我们是在通过配置文件对不同的 Hosts 运用不同的规则。
找到 SSH config file 位置 ~/.ssh/config。如果存在则编辑，否则创建它。
```sh
$ cd ~/.ssh/
$ touch config           // Creates the file if not exists
$ code config            // Opens the file in VS code, use any editor

```
给个参考的demo config:
```sh
# Personal account, - 
the default config
Host github.com
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_rsa
# Work account-1
Host github.com-work_user1    
   HostName github.com
   User git
   IdentityFile ~/.ssh/id_rsa_work_user1

```

“work_user1” is the GitHub user id for the work account.
“github.com-work_user1” is a notation used to differentiate the multiple Git accounts. You can also use “work_user1.github.com” notation as well. Make sure you’re consistent with what hostname notation you use. This is relevant when you clone a repository or when you set the remote origin for a local repository

The above configuration asks ssh-agent to:

Use id_rsa as the key for any Git URL that uses @github.com
Use the id_rsa_work_user1 key for any Git URL that uses @github.com-work_user1

### 确保同时只存在一个活跃的 SSH key 在 ssh-agent 中
ssh-add -l 命令会列出所有添加到 ssh-agent 中的 SSH keys, 删除所有已添加的keys, 然后将你要使用的 SSH keys 添加进去。
以 personal account 为例：
```sh
$ ssh-add -D            //removes all ssh entries from the ssh-agent
$ ssh-add ~/.ssh/id_rsa                 // Adds the relevant ssh key
```
现在 ssh-agent 只有一个ssh key 对应到 GitHub account。

### 给本地 repo 设置 git remote url
对我们克隆或者创建的本地 repo , 确保配置好对应远程 repo 的 user name 和 user email.
在本地 repo 中通过 git config user.name 和 git config  user.email 列出配置的name和email.
如果没有，需要配置：
```
git config user.name "User 1"   // Updates git config user name
git config user.email "user1@workMail.com"

```

### 当我们克隆跟一个远程 repo 时，要区分我们要克隆到哪个账号对应的本地 repo 
#### 个人账号克隆repo:
```sh
git clone git@github.com:personal_account_name/repo_name.git
or
git clone repo address (https://github.com/user_name/repo_name.git)

```

#### 工作账号克隆repo，则需要做一点调整：
```sh
git clone git@github.com-work_user1:work_user1/repo_name.git
```
该调整对应我们在SSH config中做的配置：在 @ 和 ：之间的字符串(github.com-work_user1) 需要同我们在config中设置的字符串一致。

### 对本地已经存在的 repo 配置方法
查看对应的远程 repo 信息，git remote -v
检查 URL 是否与我们要使用的 GitHub host 一致，否则更新 remote origin URL.
```sh
git remote set-url origin git@github.com-worker_user1:worker_user1/repo_name.git
```
Ensure the string between @ and : matches the Host we have given in the SSH config.

如果你在本地新建一个 new repo:
- 通过 git init 完成 Git 初始化
- 远程创建 new repo 并在本地将其设置为 remote url 推送的目标 repo
```sh
git remote add origin git@github.com-work_user1:work_user1/repo_name.git 
```
- 同理，要求在 @ 和 ：之间的字符串(github.com-work_user1) 需要同我们在config中设置的字符串一致。
- 推送本地初始化 commit 到远程 repo
```
git add .
git commit -m "Initial commit"
git push -u origin master

```

### 结语
到这里，我们所有的工作都完成了。后面只需要在 git clone 时对应账号的 repo 地址即可轻松完成不同账号的远程推送。

### 参考

- [How to manage multiple GitHub accounts on a single machine with SSH keys](https://medium.freecodecamp.org/manage-multiple-github-accounts-the-ssh-way-2dadc30ccaca)
