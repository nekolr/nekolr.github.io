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

有两种运行代码的方式，第一种就是通过 pipenv 命令来运行：

```
pipenv run python main.py
```

> 如果使用 PyCharm 开发环境，可以在设置中搜索 pipenv 并设置它的路径，这个路径的填写与上述路径相同