---
title: "Fabric升级到fabric2"
date: 2020-11-23T17:13:18+08:00
lastmod: 2020-11-23T17:13:18+08:00
slug: "Fabric升级到fabric2"
categories:
  - python
tags:
  - python
  - fabric
toc: true
draft: true
---

# fabric升级到fabric2

fabric2相比于fabric改动很多，并且不能向下兼容

## 安装

可以用pip安装fabric的2.X版本，也可以安装fabric2，都是一样的，用fabric2为了区分之前的版本，fabric2改动特别大

## fabric=>fabric2

### 改动

完整改动还是得看官网文档[Upgrading from 1.x](http://www.fabfile.org/upgrading.html#)，这里只列举部分常用的

#### env

移除了env，使用fabric.Config来进行配置

env.use_ssh_config不再需要，fabric2默认就会使用ssh_config

#### sudo密码

之前用env.password来配置sudo的密码，现在可以用

##### 方法1 使用全局的connection，并配置config

```python
sudo_pass = 'xxxxxx'
fab_user = 'xudai'
config = Config(overrides={'sudo': {'password': sudo_pass}})
c = Connection(host='xd', config=config)

@task
def sample(ctx):
  c.sudo('whoami')
```

##### 方法2 执行时修改config

```python
@task
def sample(ctx):
    print(ctx.config)
    sudo(ctx)
	  print(ctx.config)

def sudo(ctx):
    ctx.config.sudo.password = sudo_pass
```

##### 方法3 给sudo传参

```python
@task
 def sample(c):
    c.sudo('whoami', password=os.environ.get('SUDO_PASSWORD'))
```

##### 方法4 --prompt-for-sudo-password

执行时加上这个参数，会让你先在命令行输入一次sudo password

##### 方法5 配置config文件

#### run/local/sudo

```python
# fabric 1.X
from fabric.api import (
    local,
    run,
    sudo,
)

local('cmd')
run('cmd')
sudo('cmd')

# fabric 2.X
from fabric.connection import Connection

c = Connection(host='xd')
c.local('cmd')
c.run('cmd')
c.sudo('cmd')
```

#### fabric.contrib

fabric.contrib也移除了，现在用patchwork，可以用pip安装

##### files.exists

需要传connection参数

```python
# fabric 1.X
from fabric.contrib import files

def init_dir():
    if not files.exists(DIR_FEISHUBOT):
        create_dir(DIR_FEISHUBOT)
        
# fabric 2.X
from patchwork import files

def init_dir(ctx):
    if not files.exists(ctx, DIR_FEISHUBOT):
        create_dir(ctx, DIR_FEISHUBOT)
```

##### rsync

参数名字有点不同，并且需要传connection参数

```python
# fabric 1.X
from fabric.contrib.project import rsync_project

def deploy_bin():
    rsync_project(remote_dir=REMOTE_BIN,
                  local_dir=LOCAL_BIN,
                  exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
                  ssh_opts='-i ~/.ssh/id_rsa')
    
# fabric 2.X
from patchwork.transfers import rsync

def deploy_bin(ctx):
    rsync(c,
          target=REMOTE_BIN,
          source=LOCAL_BIN,
          exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
          ssh_opts='-J jump')
```

如果是像需要通过跳板机才能连接的host，connection.host获取到的这个host是不能直连的host地址，所以直接rsync连接不了，一般都配置了`~/.ssh.config`，在rsync的ssh_opts加上`-J jump`就行

```
Host jump
    HostName jump_host
    User xudai
    ForwardAgent yes
Host host_nickname
    HostName host_name
    User ubuntu
    ProxyJump jump
```


#### task

之前fabric只需要fabfile定义一个函数就可以被识别成fab任务，现在fabric2必须要给函数加上task装饰器才会被识别成fab任务，并且必须有一个connection参数，通常用c/ctx就行

```python
from fabric import task

@task
def sample(ctx):
  print(ctx.config)
```

如果task函数名字带有_，使用的时候要用-，用`fab --list`可以看到

```python
@task
def test_sample(ctx):
	print(ctx.config)
```

```shell
$ fab test-sample
```

#### fabric2必须指定shell

如果用的不是bash，例如zsh，执行的指令前面需要加上`/bin/zsh -c`指定shell，否则即使当前用的zsh，执行的时候还是会用bash，一些zshrc配置的东西就找不到

```python
# fabric 1.X
COMMAND_BUILD_FEISHUBOT = 'GOOS=linux GOARCH=amd64 go build -o bin/feishubot -v main.go'
# fabric 2.X
COMMAND_BUILD_FEISHUBOT = '/bin/zsh -c "GOOS=linux GOARCH=amd64 /usr/local/go/bin/go build -o bin/feishubot -v main.go"'
```

#### 命令行参数

之前传参用:，现在用GNU/POSIX-style long and short flags

```shell
# fabric 1.X
$ fab deploy:arg1,arg2
# fabric 2.X
# def deploy(ctx, prefix, postfix)
$ fab deploy --prefix=arg1 --postfix=arg2
# 可以用首字母p就行，而且即使有多个参数都是p开头的，会按顺序读取
$ fab deploy --p=arg1 --p=arg2
```

#### 命令行指定服务器

```shell
# fabric 1.X
# 之前通过定义一个set_svr(svr)的函数接收服务器名然后设置env.host
$ fab set_svr:xj deploy
# fabric 2.X
# 现在可以直接用-H来指定ssh_config配置的服务器
$ fab -H xj deploy
```

### fabric 1.X的例子

```python
#!/usr/bin/env/python
# -*-coding:utf-8 -*-
from os import getenv
from fabric.api import (
    local,
    env,
    run,
    sudo,
)
from fabric.contrib.project import rsync_project
from fabric.contrib import files

env.use_ssh_config = True

LOCAL_BIN = 'bin/'
LOCAL_CONF = 'conf/feishubot.yaml'
LOCAL_SUPERVISOR_CONF = 'conf/feishubot.conf'

REMOTE_BIN = '/data/qa/feishubot/bin'
REMOTE_CONF = '/data/qa/feishubot/conf/feishubot.yaml'
TEMP_SUPERVISOR_CONF = '/data/qa/feishubot/feishubot.conf'
REMOTE_SUPERVISOR_CONF = '/etc/supervisor/conf.d/'

COMMAND_BUILD_FEISHUBOT = 'GOOS=linux GOARCH=amd64 go build -o bin/feishubot -v main.go'

DIR_FEISHUBOT = '/data/qa/feishubot'
DIR_BIN = '/data/qa/feishubot/bin'
DIR_CONF = '/data/qa/feishubot/conf'
DIR_LOG = '/data/qa/log/feishubot'

env.hosts = 'xj'
env.gateway = getenv("USER") + '@jump_host'
env.password = 'xxxxxx'  # 配置password执行sudo就不用输入密码了


def create_dir(path):
    run('mkdir %s' % path)


def init_dir():
    if not files.exists(DIR_FEISHUBOT):
        create_dir(DIR_FEISHUBOT)
    if not files.exists(DIR_BIN):
        create_dir(DIR_BIN)
    if not files.exists(DIR_CONF):
        create_dir(DIR_CONF)
    if not files.exists(DIR_LOG):
        create_dir(DIR_LOG)


def deploy():
    deploy_bin()
    deploy_conf()


def deploy_bin():
    rsync_project(remote_dir=REMOTE_BIN,
                  local_dir=LOCAL_BIN,
                  exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
                  ssh_opts='-i ~/.ssh/id_rsa')


def deploy_conf():
    rsync_project(remote_dir=REMOTE_CONF,
                  local_dir=LOCAL_CONF,
                  exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
                  ssh_opts='-i ~/.ssh/id_rsa')


def deploy_supervisor():
    rsync_project(remote_dir=TEMP_SUPERVISOR_CONF,
                  local_dir=LOCAL_SUPERVISOR_CONF,
                  exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
                  ssh_opts='-i ~/.ssh/id_rsa')
    sudo('mv %s %s' % (TEMP_SUPERVISOR_CONF, REMOTE_SUPERVISOR_CONF))


def restart():
    sudo('supervisorctl restart feishubot')


def build():
    local(COMMAND_BUILD_FEISHUBOT)


def update_supervisor():
    sudo('supervisorctl update')


def init():
    build_feishubot()
    init_dir()
    deploy()
    deploy_supervisor()
    update_supervisor()


def feishu():
    build()
    deploy()
    restart()
```

### fabric 2.X的例子

```python
#!/usr/bin/env/python
# -*-coding:utf-8 -*-

from fabric import task, Config
from fabric.connection import Connection
from patchwork import files
from patchwork.transfers import rsync

sudo_pass = 'xxxxxx'
fab_user = 'xudai'
config = Config(overrides={'sudo': {'password': sudo_pass}})
c = Connection(host='xd', config=config)

LOCAL_BIN = 'bin/'
LOCAL_CONF = 'conf/feishubot.yaml'
LOCAL_SUPERVISOR_CONF = 'conf/feishubot.conf'

REMOTE_BIN = '/usr/share/feishubot/bin'
REMOTE_CONF = '/usr/share/feishubot/conf/feishubot.yaml'
TEMP_SUPERVISOR_CONF = '/usr/share/feishubot/feishubot.conf'
REMOTE_SUPERVISOR_CONF = '/etc/supervisor/conf.d/'

COMMAND_BUILD_FEISHUBOT = '/bin/zsh -c "GOOS=linux GOARCH=amd64 /usr/local/go/bin/go build -o bin/feishubot -v main.go"'

DIR_FEISHUBOT = '/usr/share/feishubot'
DIR_BIN = '/usr/share/feishubot/bin'
DIR_CONF = '/usr/share/feishubot/conf'
DIR_LOG = '/var/log/feishubot'


@task
def create_dir(ctx, path):
    # need root permission
    c.sudo('mkdir %s' % path)
    # The rsync command should be done by the same user as the folder owner's one
    c.sudo('chown -R %s %s' % (fab_user, path))


@task
def init_dir(ctx):
    if not files.exists(c, DIR_FEISHUBOT):
        create_dir(ctx, DIR_FEISHUBOT)
    if not files.exists(c, DIR_BIN):
        create_dir(ctx, DIR_BIN)
    if not files.exists(c, DIR_CONF):
        create_dir(ctx, DIR_CONF)
    if not files.exists(c, DIR_LOG):
        create_dir(ctx, DIR_LOG)


@task
def deploy(ctx):
    deploy_bin(ctx)
    deploy_conf(ctx)


@task
def deploy_bin(ctx):
    rsync(c,
          target=REMOTE_BIN,
          source=LOCAL_BIN,
          exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
          ssh_opts='-J jump')


@task
def deploy_conf(ctx):
    rsync(c,
          target=REMOTE_CONF,
          source=LOCAL_CONF,
          exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
          rsync_opts='-O',
          ssh_opts='-i ~/.ssh/id_rsa')


@task
def deploy_supervisor(ctx):
    rsync(c,
          target=TEMP_SUPERVISOR_CONF,
          source=LOCAL_SUPERVISOR_CONF,
          exclude=['.git', '*.pyc', '__pycache__', '.vscode', '.idea', '*.xls', '.DS_Store'],
          ssh_opts='-J jump')
    c.sudo('mv %s %s' % (TEMP_SUPERVISOR_CONF, REMOTE_SUPERVISOR_CONF))


@task
def restart(ctx):
    c.sudo('supervisorctl restart feishubot')

    
@task
def build(ctx):
    c.local(COMMAND_BUILD_FEISHUBOT)


@task
def update_supervisor(ctx):
    c.sudo('supervisorctl update')


@task
def init(ctx):
    build(ctx)
    init_dir(ctx)
    deploy(ctx)
    deploy_supervisor(ctx)
    update_supervisor(ctx)


@task
def feishu(ctx):
    build(ctx)
    deploy(ctx)
    restart(ctx)
```

