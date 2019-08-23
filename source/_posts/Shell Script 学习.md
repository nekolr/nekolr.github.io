---
title: Shell Script 学习
date: 2019/1/23 13:48:0
tags: [Shell]
categories: [Shell]
---
简单学习过 Shell 后，接下来继续学习 Shell 脚本。  

<!--more-->  
# 良好风格
- 在脚本的第一行，要宣告该脚本使用的 shell 名称，比如 `#!/bin/bash`。
- 说明部分，要说明脚本的功能、版本、作者、联系方式、版权声明等。
- 脚本运行时需要用到的环境变量要预先定义和配置。
- 脚本正文中，比较特殊的命令使用绝对路径的方式下达。
- 定义回传值，比如成功为 0，失败为非零整数。

# 注释
单行注释很简单，使用 `#` 开头就代表本行注释。  

```bash
# 我是注释
```

## 多行注释
使用 `:<<` 开头，后边可以跟其他的字符，最后再以对应的字符结尾。  

```bash
:<<EOF
我是多行注释
I am multi-line comment
EOF

:<<!
我是多行注释
I am multi-line comment
!
```

# 条件判断
一般在编程语言中，我们通常都能够使用 if else 语句进行条件判断，bash 也不例外。但是除了条件判断语句，bash 还有一些条件判断的命令。  

## test 命令
test 命令可以检测系统中的文件以及相关的属性。常用的参数如下：  

| 标志 | 含义 |
|-|-|
| -e | 文件是否存在 |
| -f | 判断是否存在且为文件 |
| -d | 判断是否存在且为目录 |
| -L | 判断文件是否存在且是链接文件（软链接或硬链接） |
| -r | 判断文件是否存在且有读的权限 |
| -w | 判断文件是否存在且有写的权限 |
| -x | 判断文件是否存在且有执行的权限 |
| -nt | `test file1 -nt file2`，判断 file1 是否比 file2 新（newer than） |
| -ot | 判断 file1 是否比 file2 旧（older than） |
| -ef | 判断两个文件是否为同一个文件，主要用于判断硬链接。 |
| -eq | 判断两个整数是否相等（equals） |
| -ne | 判断两个整数是否不相等（not equals） |
| -gt | `test num1 -gt num2`，判断 num1 是否大于 num2（greater than） |
| -lt | 判断 num1 是否小于 num2（less than） |
| -ge | 判断 num1 是否大于等于 num2（greater than or equals） |
| -le | 判断 num1 是否小于等于 num2（less than or equals） |
| -z | 判断字符串是否为空 |
| -n | 判断字符串是否不为空 |
| = | `test str1 = str2`，判断 str1 是否等于 str2 |
| != | `test str1 != str2`，判断 str1 是否不等于 str2 |
| -a | 相当于 and，只有都成立时才会返回 true，比如 `test -r file1 -a -x file2` |
| -o | 相当于 or，任何一个成立时就会返回 true |
| ! | 取反，比如 `test ! -x file1`，当 file1 不具有执行的权限时返回 true |

需要注意的是，test 命令返回的值并不会直接输出到屏幕，因此需要自己控制输出，比如：  

```bash
$ test -r file1 && echo true || echo false
```

## 判断符号
除了 test 命令，我们还可以使用判断符号 `[]`，也就是中括号来进行数据的判断。需要注意的是，因为中括号用在很多地方，所以在 bash 中使用中括号作为判断符号时，**中括号的两端需要有空白字节来分隔，并且在中括号中的变量一定要用双引号包含，同时常量也要使用单引号或者是双引号包含**。  

```bash
$ [ -z "$HOME" ]; echo $?
$ read -p "Please input (Y/N)" yn
$ [ "$yn" == "Y" -o "$yn" == "y" ] && echo "OK, continue." && exit 0
```

> 在 bash 中使用一个等号与两个等号的意义相同，不过在程序中一般写法是，一个等号代表赋值，两个等号则代表条件判断。建议在条件判断时，使用双等号。  

## 条件语句
```bash
read -p "Please input (Y/N): " yn
if [ "$yn" == "Y" ] || [ "$yn" == "y" ]; then
    echo "OK, continue."
    exit 0
elif [ "$yn" == "N" ] || [ "$yn" == "n" ]; then
    echo "Oh, interrupt!"
    exit 0
else
    echo "I don't know what your choice is"
fi
```

```bash
case $1 in
    "open")
        echo "This switch is open"
        ;;
    "close")
        echo "This switch is close"
        ;;
    "")
        echo "You must input parameters"
        ;;
esac
```

# 函数
```bash
function printit() {
    echo "Your choice is $1"
}

case $1 in
    "one")
        # 其中 1 是调用函数时传入的参数
        printit 1
        ;;
    "two")
        printit 2
        ;;
    "three")
        printit 3
        ;;
esac
```

# 循环
```bash
while [ "$yn" != "Y" ] && [ "$yn" != "y" ]
do
	read -p "Please input Y/y to stop this program: " yn
done
echo "OK! you input the correct answer."
```

```bash
until [ "$yn" == "Y" ] || [ "$yn" == "y" ]
do
	read -p "Please input Y/y to stop this program: " yn
done
echo "OK! you input the correct answer."
```

while 循环与 until 循环的区别是，while 循环在条件满足时才会执行循环体；而 until 循环是在条件不满足时才会执行循环体。  

```bash
for animal in dog cat mouse rabbit
do
	echo "There are ${animal}s.... "
done
```

```bash
read -p "Please input a number, I will count for 1+2+...+your_input: " num

sum=0
for (( i=1; i<=$num; i=i+1 ))
do
	sum=$(($sum+$i))
done
echo "The result of '1+2+3+...+$num' is ==> $sum"
```

# 语法检查
使用 `sh -n xxx.sh` 来检查脚本是否存在语法错误，如果没有则不会显示任何信息。同时，可以使用 `sh -x xxx.sh` 将脚本的执行过程全部列出，这样能够方便发现和调试错误。  

# 运行 Shell 脚本
在运行脚本之前，需要先确认脚本是否具有执行的权限。如果没有，则需要赋予执行的权限。  

```
# 以下命令需要 cd 到脚本所在目录执行

# 赋予执行的权限
chmod +x ./xxx.sh

# 执行脚本
./xxx.sh
```

一定要写成 `./xxx.sh`，运行其它二进制的程序也一样。如果直接写 `xxx.sh`，系统会去 PATH 里寻找，而一般 PATH 里只有 `/bin`，`/sbin`，`/usr/bin`，`/usr/sbin` 等，当前目录通常不在 PATH 里，所以写成 `xxx.sh` 是找不到命令的，要用 `./xxx.sh` 告诉系统就在当前目录找。  

还有一种运行脚本的方法是直接运行解释器，传入需要运行的脚本文件名：`/bin/sh xxx.sh`。以及使用 source 命令运行脚本，比如 `source ./xxx.sh`，这种方式与前两种的区别是，使用该命令，脚本会在父程序中执行；而前两种会在子程序中运行。这也是为什么使用 source 可以使配置立即生效的原因。  

# 参考
> 《鸟哥的Linux私房菜》
