---
title: GitHub 多账号的 SSH 配置
tags: 
- 技术
- 容器
- Docker
date: 2017-01-01 08:00:00
external: https://learnku.com/articles/4411/github-multi-account-ssh-configuration
---


##  背景
有这样的情况，现在有 2 个 github 账号。现在都想使用 ssh key 的方式拉取代码。但是第二个账号无法使用，怎么办?

## 实现
### 1. 生成 SSH key
1.1 生成第一个 key

	$ ssh-keygen -t rsa -C "user1@email.com"

然后，回车确认即可。最后会生成

	id_rsa
	id_rsa.pub

2个文件。

1.2 生成第二个 key

	$ ssh-keygen -t rsa -C "user2@email.com"

注意，这里需要给 key 设置名称。要不然会覆盖第一个key，例如:

	 ~/.ssh/id_rsa_user2@email.com

至此，我们会得到4个文件。


### 2. 添加密钥到 SSH agent 中

	$ ssh-agent bash
	$ ssh-add ~/.ssh/id_rsa_user2@email.com

### 3. 修改 SSH 的 config 配置

	$ touch config    

增加:

	HOST user2.github
	hostname github.com
	IdentityFile ~/.ssh/id_rsa_user2@email.com
	User user2@email.com

### 4. 配置公钥

把 id_rsa.pub  和 id_rsa_user2@email.com.pub 文件的内容添加到对应账号的公钥配置中。

### 5. 拉取项目
例如，user2 要克隆这个项目:

	git@github.com:EvaEngine/Dockerfiles.git

则执行下面的命令即可:

	$ git clone git@user2.github:EvaEngine/Dockerfiles.git

注意，地址中的 host 替换为了 config 中配置的名称。

搞定。
