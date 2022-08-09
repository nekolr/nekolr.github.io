---
title: Python 虚拟环境
date: 2019/3/8 9:10:0
tags: [Python]
categories: [Python]
---
在初学 Python 时，我使用的是官方推荐的 Pipenv，一开始用起来还好，但是每次创建和修改 lock 文件时都会卡住很长时间，在切换了其他镜像后问题依旧，最后忍无可忍直接放弃了 Pipenv，改用原始的 pip + virtualenv 的方式。  

<!--more-->

# 为什么要使用虚拟环境
我们在使用 pip 安装第三方包时，通常都会安装到 Python 的 `site-packages` 目录中。比如我本地的是 Windows 平台，安装目录为 `C:\Users\nekolr\AppData\Roaming\Python\Python37\site-packages`。我们可以通过 `py -m site --user-site` 命令查看 pip 的安装目录，也可以通过 `pip list` 命令列出所有通过 pip 安装的包，然后再通过 `pip show 包名` 来查看每个包具体的安装路径。  

如果我们要同时开发多个应用，所有的应用都会使用同一个 Python 环境，那么使用 pip 命令安装的包都会在同一个目录中。这会存在两个问题：一个问题是当应用很多时该目录会膨胀到很大，不利于 IDE 进行索引；另一个问题是由于使用 pip 安装的包不会带有版本后缀，因此只会有一个版本，如果不同的应用需要使用不同版本的包，这就无法实现了。  

因此我们需要每个应用都能够有一套自己独立的 Python 运行环境，这也就是 virtualenv 的由来。  

# 环境变量
为了能够全局使用 python 和 pip 命令，以及后续我们安装的第三方包提供的命令，我们需要设置环境变量。  

设置 `PYTHON_HOME` 为 Python 的安装目录，同时在 `PATH` 中加入 `%PYTHON_HOME%` 和 `%PYTHON_HOME%\Scripts`，这样我们就可以在全局使用 python 和 pip 等命令了。接下来再添加第三方包的命令的搜索目录，为此我创建了 `PIPINSTALL` 变量，然后使用 `py -m site --user-site` 找到 pip 的第三方包安装目录，把最后的 site-packages 替换为 Scripts，这个替换后的路径作为 `PIPINSTALL` 变量的值，然后将该变量加入到 `PATH` 变量的值中。  

# 安装
安装 virtualenv 比较简单，直接通过 pip 命令安装。  

```python
pip install virtualenv
```

pip 默认会从 <https://pypi.org/simple> 处下载第三方包，如果下载速度较慢，可以切换到其他源下载。  

```python
# 清华源
pip install virtualenv -i https://pypi.tuna.tsinghua.edu.cn/simple/
# 阿里云源
pip install virtualenv -i https://mirrors.aliyun.com/pypi/simple/
```

# 使用
假如我们有一个新项目需要一套独立的 Python 运行环境，我们可以通过以下步骤操作。  

首先，我们要创建项目，然后通过 cd 切换到项目所在目录。  

然后使用 `virtualenv --no-site-packages venv` 命令来创建一个独立的 Python 运行环境，这里的 venv 是我自己指定的虚拟环境的名称，`no-site-packages` 参数意为不会将 site-packages 下的包复制到新环境中，因此我们就能够得到一个干净的新环境。  

这个新环境默认会在当前应用的目录下创建一个与创建虚拟环境时指定的虚拟环境名称相同的目录，比如我上面创建虚拟环境时指定的名称为 venv，因此在当前应用下会出现一个新的名为 venv 的文件夹。然后我们可以通过该目录下的 Scripts 目录中的命令来进入虚拟环境。  

```bash
$ venv\Scripts\activate
```

进入虚拟环境后，在命令提示符前面会多出一个 (venv) 前缀。在 venv 环境下，用 pip 安装的包都被安装到 venv 这个环境下，系统 Python 环境不会受到任何影响。如果想退出虚拟环境，可以使用 `deactivate` 命令。  

```bash
$ venv\Scripts\deactivate
```

# 参考
> [virtualenv](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432712108300322c61f256c74803b43bfd65c6f8d0d0000)