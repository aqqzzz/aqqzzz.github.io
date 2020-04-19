---
title: hexo d报错Spawn failed
date: 2020-04-19 18:19:26
tags: hexo
categories: hexo
---

### hexo部署流程

```
# 清理部署文件
hexo clean

# 生成静态文件
hexo g

# 本地起服务查看博客
hexo s

# 部署到远程服务器
hexo d
```

### hexo d 报错

今天在 hexo d 的时候突然报错

![image.png](http://ww1.sinaimg.cn/large/8ac7964fly1gdz9c1ngqmj20vg0ii7wh.jpg)

在网上查了一下，大部分提供的解决方案是在博客根目录中的 _config.yml 文件中，将 deploy 部分修改为以下：

```javascript
deploy:
  type: git
  repo: git@github.com:aqqzzz/aqqzzz.github.io.git
  branch: master
```



网上的大部分情况，在这里配置之后应该就可以了，但我修改之后再执行 hexo d 时，还是会报错，报错信息如下图

![image.png](http://ww1.sinaimg.cn/large/8ac7964fly1gdz9gebhmjj20vi0dokhx.jpg)



还在报 Spawn failed 的错误，但是发现上面还有连接失败的错误，无法连接到git仓库

因此使用  `ssh -v git@github.com` 来查看 git 的ssh连接，如果连接成功，会返回用户名，我在调用时则返回了 permission denied 的错误，发现可能是因为ssh密钥连接的问题，

```
ls -al ~/.ssh
```

其中 id_rsa.pub 则为存储公钥的文件，查看发现当前存储的密钥还是之前实习的时候的邮箱生成的密钥，可能当时在远程的时候配置了公司的ssh



接下来则具体参考如下链接中的流程进行配置：https://help.github.com/cn/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent

具体为：生成新ssh密钥——将ssh密钥添加到 ssh-agent



### 结语

在这次生成新ssh密钥时，把旧的id_rsa.pub文件中的公钥文件覆盖了，如果之后需要在别的项目中使用别的git账号，需要重新走流程生成新的 SSH key

Mac 配置多个git账号的方式：https://www.jianshu.com/p/698f82e72415