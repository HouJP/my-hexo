---
title: SSH Notes
date: 2018-04-23 17:42:52
tags: [ssh]
---

### 配置跳板机

```shell
# 通过命令行
ssh username@目标机器IP -p 22 -o ProxyCommand='ssh -p 22 username@跳板机IP -W %h:%p'
```

<!-- more -->

```Shell
# 通过ssh config
vim ~/.ssh/config

Host tiaoban				# 跳板机别名
    HostName 192.168.1.1	# 跳板机IP/HostName
    Port 22      			# 跳板机端口
    User username_tiaoban	# 跳板机用户

Host nginx      			# 目的主机别名
    HostName 192.168.1.2  	# 目的主机IP/HostName
    Port 22   				# 目的主机端口
    User username   		# 目的主机用户
    ProxyCommand ssh username_tiaoban@tiaoban -W %h:%p

Host 10.10.0.*      		# 批量配置目的主机
    Port 22   				# 目的主机端口
    User username   		# 目的主机用户
    ProxyCommand ssh username_tiaoban@tiaoban -W %h:%p

# 通过nginx登录
ssh nginx

# 通过批量配置登录
ssh username@10.10.0.{}		# {} 需要根据需要填充
```

### 免密登录

如果想在A服务器上不需要密码登录到B服务器，那么需要将A服务器的公钥放到B服务器上，具体如下。

1. 在A服务器上生成公钥。

   ```Shell
   # located at A server
   ssh-keygen -t rsa
   cat ~/.ssh/id_rsa.pub
   ```

2. 将A服务器的公钥拷贝到B服务器的`authorized_keys`文件中。

   ```Shell
   # located at B server
   vim ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

   ​