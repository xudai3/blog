---
title: "Svn服务器搭建及Hook配置"
date: 2020-12-03T11:07:23+08:00
lastmod: 2020-12-03T11:07:23+08:00
slug: "Svn服务器搭建及Hook配置"
categories:
  - svn
tags:
  - svn
toc: true
draft: true
---

# svn服务器搭建及Hook配置

## svn服务器搭建

### 1.安装SVN

在服务器上安装好SVN

```shell
$ sudo apt install subversion
```

### 2.创建服务端仓库

```shell
# 创建存放SVN仓库的目录
$ mkdir -p /data/svn
$ cd /data/svn
# 创建SVN仓库
$ svnadmin create svn-repo
```

### 3.配置

```shell
$ cd /data/svn/svn-repo/conf
```

#### svnserve.conf

```shell
$ vim /data/svn/svn-repo/conf/svnserve.conf
# 下面三行默认是被注释掉的，需要取消掉注释
# 权限 none:不允许读和写 read:只允许读 write:允许读和写
# anonymous用户的权限
anon-access = none
# 授权用户的权限
auth-access = write
# 登陆密码的配置文件，passwd是指conf/passwd文件
password-db = passwd
```

#### authz

```shell
$ vim /data/svn/svn-repo/conf/authz
# 配置仓库名和用户的权限
[svn-repo:/]
xudai = rw
* = r
```

#### passwd

```shell
$ vim /data/svn/svn-repo/conf/passwd
# 配置用户名和密码
[users]
xudai = xxxx
```

### 4.启动svn服务

```shell
# 例如创建的svn仓库地址在/data/svn/svn-repo/，启动的时候root选项一定要用上一级的地址/data/svn/
$ svnserve -d -r /data/svn
```

### 5.连接

地址：`svn://ip/svn-repo`

## Hook配置

hooks目录下有各种hook的模版文件(hook-name.tmpl)，复制模板去掉后缀或者自己创建一个跟所需hook同名的文件，可以直接写shell脚本执行，也可以通过shell脚本再执行其他的如python脚本，记得加上可执行权限

```shell
$ cd /data/svn/svn-repo/hooks
$ ls
pre-commit.tmpl
post-commit.tmpl
...
```

### pre-commit

提交之前执行，执行脚本时会传递两个参数，REPOS(仓库路径) ，TXN(事务名)

可以用来做log规范检查，log不规范不允许提交

```shell
$ cp pre-commit.tmpl pre-commit
$ vim pre-commit
```

```shell
# pre-commit
REPOS="$1"
TXN="$2"

python "$REPOS"/hooks/comment_check.py "$REPOS" "$TXN"
```

```python
# comment_check.py
import sys
import os

def main(argv):
    (repos, txn) = argv

    changelist = os.popen("/usr/bin/svnlook changed '%s' -t '%s'" % (repos,txn)).readlines()
    changelist_str = "".join(changelist)
    message = "".join(os.popen("/usr/bin/svnlook log '%s' -t '%s'" % (repos, txn)).readlines()).strip()
    if len(message) < 10:
        sys.stderr.write("请输入10字以上描述详细改动的log")
        sys.exit(1)
    sys.exit(0)

if __name__ == '__main__':
    main(sys.argv[1:])
```

### post-commit

提交成功之后执行，执行时会传递三个参数，REPOS(仓库路径)，REV(版本号)，TXN_NAME(事务名)

可以用来做提交通知

```shell
$ cp pre-commit.tmpl pre-commit
$ vim pre-commit
```

```shell
# post-commit
REPOS="$1"
REV="$2"
TXN_NAME="$3"

python "$REPOS"/hooks/commit_notify.py "$REPOS" "$REV" "$TXN_NAME"
```

```python
# commit_notify.py
import os
import sys
import requests

url = "your-url"

def main(argv):
    (repos_path, rev, txn_name) = argv
    svninfo = os.popen("/usr/bin/svnlook info '%s'" % repos_path).readlines()
    submitter = svninfo[0]
    submit_date = svninfo[1]
    current_version = svninfo[2]
    submit_comment = svninfo[3]
    timestr = " ".join(submit_date.split(" ")[:2])
    changelist = os.popen("/usr/bin/svnlook changed '%s'" % repos_path).readlines()
    changelist_str = "".join(changelist)
    data = {
        'type': "post-commit",
        'submitter': submitter,
        'change_list': changelist_str,
        'time': timestr,
        'comment': submit_comment,
    }
    requests.post(url, json=data)

if __name__ == '__main__':
    main(sys.argv[1:])
```

