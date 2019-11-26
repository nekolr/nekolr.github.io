---
title: 使用 pipenv
date: 2019/11/26 17:35:0
tags: [Python]
categories: [Python]
---

首先需要安装 Python 环境，同时将安装目录和安装目录下的 Scripts 目录添加到 PATH 环境变量中。

<!--more-->

# 安装 pipenv
接下来开始安装 pipenv，使用以下命令将 pipenv 安装到用户基础目录中。

```bash
pip install pipenv --user -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

# 设置环境变量
接着需要使用 `py -m site --user-site` 命令获取用户基础目录，假设获取到的路径为：

```
C:\Users\nekolr\AppData\Roaming\Python\Python38\site-packages
```

将 `site-packages` 去掉并加上 `Scripts` 组成以下路径：

```
C:\Users\nekolr\AppData\Roaming\Python\Python38\Scripts
```

这个就是 pipenv 安装的目录，将这个路径添加到 PATH 环境变量中，这样我们就可以在命令行中使用 pipenv 命令了。

# 安装依赖
接下来就可以新建一个 Python 项目，然后进入到项目目录下，使用 `pipenv install` 命令来新建一个虚拟环境。pipenv 会在项目目录中创建一个 Pipfile 和一个 Pipfile.lock 文件，用于跟踪项目中安装的依赖，在项目提交时，可将 Pipfile 文件和 Pipfile.lock 文件一并提交。

假如我们需要安装一个依赖库 requests，则使用以下命令：

```bash
pipenv install requests
```

# 运行
编写以下代码：

```python
# -*- coding: utf-8 -*-

import requests

response = requests.get('https://httpbin.org/ip')
print('Your IP is {0}'.format(response.json()['origin']))

```

有两种方式来运行代码，一种是通过 `pipenv run python main.py` 命令来运行，另一种是启动虚拟环境的 shell 来运行：

```bash
# 启动虚拟环境的 shell
pipenv shell
# 执行
python3 main.py
```

> 如果使用 PyCharm 作为开发工具，可以在 settings 中搜索 pipenv 并设置它的安装路径，这样在新建项目时 IDE 就会直接帮我们新建并初始化虚拟环境了，非常方便。

# pipenv 常用命令
```bash
pipenv --where                 # 列出本地工程路径
pipenv --venv                  # 列出虚拟环境路径
pipenv --py                    # 列出虚拟环境的 Python 可执行文件
pipenv install                 # 创建虚拟环境
pipenv isntall [moduel]        # 安装包
pipenv install [moduel] --dev  # 安装包到开发环境
pipenv uninstall[module]       # 卸载包
pipenv uninstall --all         # 卸载所有包
pipenv graph                   # 查看包依赖
pipenv lock                    # 生成 lockfile
pipenv run python [pyfile]     # 运行 python 文件
pipenv --rm                    # 删除虚拟环境
```