---
layout: post
title: 用 ansible 管理集群
tags:
    - ansible
    - cluster
    - devops
comments: true
---

最近接手了一个 Hadoop 集群的维护。这个集群以前规模不大，维护的时候都是手工一台台 ssh 过去操作，唯一比较“自动化”的部分就是在一台机器上编辑好配置后执行 scp 把配置同步到其它机器上。这次接手后，做了些安全加固，并且后面准备扩充集群规模。这种手工方式总是容易出错（就像忘了在其中一台机器上执行某条命令什么的），于是开始寻求运维自动化工具。在对比过几个比较流行的工具后，选择了使用 ansible.

## Ansible 的特点

Ansible 是 Red Hat 的项目，它的开源版本功能就非常强大了。我选择使用 ansible 主要是看中了它的以下优点。

1. 不需要在受控主机安装额外程序。

    Ansible 只需要受控主机有 ssh, sftp 和 python.

2. Declarative programing.

    Ansible 的操作是 declarative 的，用 ansible 控制集群的时候，给出的指令一般是描述期望的最终状态。例如，启动服务是 `state=started`. 如果在受控主机上，服务已经启动了，那么 ansible 不会重复执行启动命令，并且会在返回结果上告诉你 `changed: false`, 状态没有发生变化。

    同样，在传输文件、用户管理等操作中，ansible 也不会传输一样的文件，不会重复建立相同的用户，或加入相同的组。这一点很重要，因为这意味着这些操作可以重复地执行，而不用担心出错。比如 Hadoop 集群建立 hdfs 用户，并加入 hadoop 组，如果某一天我新增加了一批节点，我可以直接对所有 datanode 执行这个操作，不需要担心在原有的节点上执行建立 hdfs 用户的命令会出错。

3. 兼容多种环境。

    还是以管理服务为例，同样的 `enabled=yes` 在 upstart 机器上会调用 `update-rc.d`, 在 systemd 上会调用 `systemctl`. Systemd 的配置文件修改后要执行 `systemctl daemon-reload` 才会生效，ansible 也会自己根据需要去执行这个命令，而不需要用户关心这些细节。


## Ansible 的使用

### 安装

Ansible 是个 python 程序，直接 `pip install ansible`。

为了方便控制，应该使用 ssh 的 key autentication. 如果 ssh 私钥设置了口令密钥，应该在 ssh-agent 环境中运行 ansible. 这些都是 ssh 的基础操作，就不详细讲了。

### Inventory

Ansible 需要知道你的集群有哪些机器，这些信息保存在 inventory 中。这是一个简单的 inventroy.

```
[namenode]
192.168.2.100   ansible_user=root

[secondarynamenode]
192.168.2.101   ansible_user=root

[datanode]
192.168.2.101   ansible_user=root
192.168.2.102   ansible_user=root
192.168.2.103   ansible_user=root
192.168.2.104   ansible_user=root
192.168.2.105   ansible_user=root
```

### Ad-hoc 命令

Ansible 的 ad-hoc 就是直接在命令行上指定要执行的任务。与之相反的是通过 playbook 文件指定要执行的一系列任务。Ad-hoc 命令的常见用法如下。

```
ansible -i <inventory file> <host or group> -m <module> -a "<parameters>"
```

下面是一些例子。

* 部署代码和配置

{% highlight sh %}
ansible -i inventory all -m synchronize -a "src=opt/hadoop dest=opt/"
ansible -i inventory all -m file -a "path=opt/hadoop owner=root group=root recurse=yes"
ansible -i inventory namenode -m copy -a "src=service/hdfs-namenode.service dest=/etc/systemd/system/"
ansible -i inventory secondarynamenode -m copy -a "src=service/hdfs-secondarynamenode.service dest=/etc/systemd/system/"
ansible -i inventory datanode -m copy -a "src=service/hdfs-datanode.service dest=/etc/systemd/system/"
{% endhighlight %}

* 启动服务

{% highlight sh %}
ansible -i inventory datanode -m service "name=hdfs-datanode state=started"
ansible -i inventory namenode -m service "name=hdfs-namenode state=started"
ansible -i inventory secondarynamenode -m service "name=hdfs-secondarynamenode state=started"
{% endhighlight %}
