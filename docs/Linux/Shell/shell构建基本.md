# 构建基本脚本

## 变量

```shell
#!/bin/bash

value1=10
value2=$value1
echo The resulting value is $value1.
# ./test.sh 
The resulting value is 10.
```

变量引用还可以通过 `${variable}` 形式引用变量。

## 命令替换

* 反引号 `
* $() 格式

```shell
#!/bin/bash

yesterday=`date +%F -d -1day`
today=$(date +%F)
echo yesterday: $yesterday today: ${today}.
# ./test.sh  
yesterday: 2021-02-28 today: 2021-03-01.
```

## 重定向

### 输出重定向

* 覆盖重定向 `>`
* 追加重定向 `>>`

```shell
ls /root &>>/dev/null
```

上面是将正确输出和错误追加到 `/dev/null` 中，也可以追加到指定文件。

### 输入重定向

```shell
cat >>a.txt<<EOF
> a
> b
> EOF
```

## 管道

```
command1 | command2
```

管道的执行顺序不是串起上面两个命令依次执行。Linux 系统实际上会同时运行这两个命令，在系统内部将它们连接起来。在第一个命令产生输出的同时，输出会被立即送给第二个命令。数据传输不会用到任何中间文件或者缓冲区。

## tee 命令

tee 命令相当于管道的一个T型接头。它将从STDIN过来的数据同时发往两处。一处是STDOUT，另一处是tee命令行所指定的文件名。

```shell
# who |tee testfile
root     pts/1        2021-03-07 14:56 (192.168.120.1)
# cat testfile 
root     pts/1        2021-03-07 14:56 (192.168.120.1)
```

注意：默认情况下tee命令会在每次使用时覆盖输出文件内容，如果想要追加必须用 -a 选项。

```shell
# who |tee -a  testfile
root     pts/1        2021-03-07 14:56 (192.168.120.1)
# who |tee -a  testfile
root     pts/1        2021-03-07 14:56 (192.168.120.1)
# cat testfile 
root     pts/1        2021-03-07 14:56 (192.168.120.1)
root     pts/1        2021-03-07 14:56 (192.168.120.1)
```

## 执行数学运算

### expr 命令

expr 命令能够识别少数的数字和字符串操作符，目前主要永下面一种方法做数字运算。

### 使用方括号

在 bash 中再将一个数学运算结果赋给某个变量时，可以用 `$[ operation ]` 将数学表达式围起来。

```shell
#!/bin/bash

var1=100
var2=45
var3=$[$var1 / $var2]
echo The final result is $var3.
# ./test.sh 
The final result is 2.
```

### 浮点数计算

```shell
variable=$(echo "optins; expression" | bc)
```

options 允许你设置的变量，如果你需要不止一个变量，可以用分号将其分开。expression 参数定义了通过 bc 执行的数学表达式。

```shell
#!/bin/bash

var1=20
var2=3.14159
var3=$(echo "scale=10; $var1 * $var1" |bc)
var4=$(echo "scale=10; $var3 / $var2" |bc)
echo var3: $var3 var4: $var4.
# ./test.sh   
var3: 400 var4: 127.3240620195.
```

上面例子中 scale 变量设置了十位小数。

```shell
#!/bin/bash

var1=10.46
var2=43.76
var3=33.2
var4=71
var5=$(bc <<EOF
scale=4
a1=($var1 * $var2)
b1=($var3 * $var4)
a1 + b1
EOF
)
echo var5: $var5.
# ./test.sh 
var5: 2814.9296.
```

## 退出脚本

### 查看退出状态码

 ```shell
echo $?
 ```

| 状态码 | 描述                          |
| ------ | ----------------------------- |
| 0      | 命令成功结束                  |
| 1      | 一般性未知错误                |
| 2      | 不适合的shell命令             |
| 126    | 命令不可执行                  |
| 127    | 没找到命令                    |
| 128    | 无效的退出参数                |
| 128+x  | 与Linux 信号 x 相关的严重错误 |
| 130    | 通过 Ctrl + c 终止的命令      |
| 255    | 正常范围之外的退出状态码      |

### exit

默认情况下，shell 脚本会以脚本中最后一个命令的退出状态码退出。可以用 exit 命令返回自己指定的退出状态码。

# 处理用户输入

## 命令行参数

### 读取参数

`$0` 为程序名，`$1` 为第一个参数，以此类推，`$9` 第九个参数，`${10}` 第十个参数，后面的参数需要加上大括号。

```shell
#!/bin/bash
factorial=1
for (( number = 1; number <= $1; number++ ))
do
        factorial=$[ $factorial * $number ]
done
echo "The factorial of $1 is $factorial"
```

```
# ./test.sh 5
The factorial of 5 is 120
```

由于读取参数是按空格分割的，如果参数包含空格必须用引号。

```shell
#!/bin/bash
echo Hello $1, glad to meet you.
# ./test.sh "Rich Blum"
Hello Rich Blum, glad to meet you.
```

### 读取脚本名

我们在执行脚本时，一般会用 `bash script` 、`./script` 或 `/path/to/script` 几种方式执行，这种执行方式会导致 `$0` 取到的脚本名会有些区别。要解决这总问题可以用 `basename` 命令提取脚本名。

```shell
#!/bin/bash
name=$(basename $0)
if [ $name = "addem" ]
then
        total=$[ $1 + $2 ]
elif [ $name = "multem" ]
then
        total=$[ $1 * $2 ]
fi
echo The calculated value is $total.
]# ll addem  multem 
-rwxr-xr-x 1 root root 173 Mar  7 13:10 addem
lrwxrwxrwx 1 root root   7 Mar  7 13:10 multem -> test.sh
# ./addem 2 5
The calculated value is 7.
# ./multem 2 5
The calculated value is 10.
```

### 测试参数

```shell
#!/bin/bash
if [ -n "$1" ]
then
	echo Hello $1, glad to meet you.
else
	echo "Sorry, you did not identify yourself."
fi
```

## 特殊参数变量

### 参数统计

`$#`  变量含有脚本运行时携带的命令行参数的个数。

```shell
#!/bin/bash
name=$(basename $0)
if [ $# -ne 2 ]
then
	echo "Usage: $name a b"
else
	total=$[ $1 + $2 ]
	echo "The total is $total."
fi
```

```
# ./test.sh 
Usage: test.sh a b
# ./test.sh  2 8
The total is 10.
```

如果只想要最后一个参数可以使用 `${!#}` 变量，如果输入参数为空则变量返回脚本名。

```shell
#!/bin/bash
echo The last parameter is ${!#}
# ./test.sh a b c
The last parameter is c
# ./test.sh 
The last parameter is ./test.sh
```

### 抓取所有的数据

* `$*` 变量会将命令行上提供的所有参数当作一个单词保存。这个单词包含了命令行中出现的每一个参数值。基本上 `$*` 变量会将这些参数视为一个整体，而不是多个个体。
* `$@` 变量会将命令行上提供的所有参数当作同一字符串的多个独立的单词。这样你就能够遍历所有的参数，得到每个参数。这通常通过 for 命令完成。

```shell
#!/bin/bash
count=1
for param in "$*"
do
	echo "\$* Parameter #$count = $param"
	count=$[ $count + 1 ]
done
count=1
for param in "$@"
do
	echo "\$@ Parameter #$count = $param"
	count=$[ $count + 1 ]
done
```

```
# ./test.sh rich barbara katie jessica
$* Parameter #1 = rich barbara katie jessica
$@ Parameter #1 = rich
$@ Parameter #2 = barbara
$@ Parameter #3 = katie
$@ Parameter #4 = jessica
```

## 移动变量 shift

使用shift 命令时，默认情况下它会将每个参数（$1、$2...）变量向左移动一个位置。`$0` 程序名不会改变。

```shell
#!/bin/bash
count=1
while [ -n "$1" ]
do
        echo "Parameter #$count = $1"
        count=$[ $count + 1 ]
        shift
done
```

```
# ./test.sh a b c
Parameter #1 = a
Parameter #2 = b
Parameter #3 = c
```

也可以一次移动多位，例子如下：

```shell
#!/bin/bash
echo "The original parameters: $*"
shift 2
echo "Here's the new first parameters: $1"
# ./test.sh a b c d
The original parameters: a b c d
Here's the new first parameters: c
```

## 处理选项

### 查找选项

处理简单选项

```shell
#!/bin/bash
while [ -n "$1" ]
do
        case "$1" in
        -a) echo "Found the -a option." ;;
        -b) echo "Found the -b option." ;;
        *) echo "$1 is not an option." ;;
    esac
    shift
done
```

```
# ./test.sh -a -b -c
Found the -a option.
Found the -b option.
-c is not an option.
```

分离参数和选项

如果想在 shell 脚本中同时使用选项和参数，则可以通过特殊字符将两者分开，这个字符就是双破折线 `--`， 在双破折线之后脚本可以将剩下的命令行参数当作参数而不是选项来处理。

```shell
#!/bin/bash
while [ -n "$1" ]
do
	case "$1" in
        -a) echo "Found the -a option." ;;
        -b) echo "Found the -b option." ;;
        --) shift
            break ;;
        *) echo "$1 is not an option." ;;
    esac
    shift
done
count=1
for param in $@
do
	echo "Parameter #$count: $param"
	count=$[ $count + 1 ]
done
```

```
# ./test.sh -a -b -c  -- 1 2 
Found the -a option.
Found the -b option.
-c is not an option.
Parameter #1: 1
Parameter #2: 2
```

处理带值得的选项

```shell
#!/bin/bash
while [ -n "$1" ]
do
	case "$1" in
        -a) echo "Found the -a option." ;;
        -b) param=$2
            echo "Found the -b option, with parameter value $param"
            shift ;;
        --) shift
            break ;;
        *) echo "$1 is not an option." ;;
    esac
    shift
done
count=1
for param in $@
do
	echo "Parameter #$count: $param"
	count=$[ $count + 1 ]
done
```

```
# ./test.sh -c -a -b 33333
-c is not an option.
Found the -a option.
Found the -b option, with parameter value 33333
```

### 使用 getopt 命令

命令格式

```
getops optstring parameters
```

```shell
# getopt ab:cd -a -b 3232 -cd 11 22 33
 -a -b 3232 -c -d -- 11 22 33
```

optstring 定义了四个有效选项字母：a、b、c和d。冒号（:）被放在字母b后，因为b选项需要一个参数值。当getopt 命令运行时，它会检查提供的参数列表（-a -b 3232 -cd 11 22 33)，并基于提供的optstring进行解析。注意，它会自动将-cd选项分成两个单独的选项，并插入双破折线来分隔行中的额外参数。

如果指定了一个不在optstring中的选项，默认情况下，getopt命令会产生一条错误信息。

```shell
# getopt ab:cd -a -b 3232 -cde 11 22 33 
getopt: invalid option -- 'e'
 -a -b 3232 -c -d -- 11 22 33
```

如果想忽略这条错误消息，可以在命令后加 -q 选项。

```shell
# getopt -q ab:cd -a -b 3232 -cde 11 22 33   
 -a -b '3232' -c -d -- '11' '22' '33'
```

脚本中使用getopt

```shell
#!/bin/bash
set -- $(getopt -q ab:cd "$@")
while [ -n "$1" ]
do
	case "$1" in
        -a) echo "Found the -a option." ;;
        -b) param=$2
            echo "Found the -b option, with parameter value $param"
            shift ;;
        -c) echo "Found the -c option." ;;
        --) shift
            break ;;
        *) echo "$1 is not an option." ;;
    esac
    shift
done
count=1
for param in $@
do
	echo "Parameter #$count: $param"
	count=$[ $count + 1 ]
done
```

```
# ./test.sh -ac -b 2333 22 33
Found the -a option.
Found the -c option.
Found the -b option, with parameter value '2333'
Parameter #1: '22'
Parameter #2: '33'
```

```
# ./test.sh -ac -b 2333 "22 33"
Found the -a option.
Found the -c option.
Found the -b option, with parameter value '2333'
Parameter #1: '22
Parameter #2: 33'
```

注意：getopt 命令并不擅长处理带空格和引号的参数值。它会将空格当作参数分隔符，而不是根据双引号将二者当作一个参数。解决这种问题可以用 getops 命令。

### getopts 命令

getopts 只处理命令行上检测到的一个参数，处理完所有的参数后，它会退出并返回一个大于0的退出状态码。

命令格式

```shell
getopts optstring variable
```

opstring 值类似于 getopt 命令中的那个。有效的选项字母都会列在optstring中，如果选项字母要求有个参数值，就加一个冒号。要去掉错误消息的话，可以在optstring之前加一个冒号。getopts命令将当前参数保存在命令行中定义的variable中。

getopts 命令会用到两个环境变量。如果选项需要跟一个参数值，OPTARG环境变量就会保存这个值。OPTIND环境变量保存了参数列表中getopts正在处理的参数位置。

```shell
#!/bin/bash
while getopts :ab:c opt
do
	case "$opt" in
        a) echo "Found the -a option" ;;
        b) echo "Found the -b option, with value $OPTARG" ;;
        c) echo "Found the -c option" ;;
        *) echo "Unknown option: $opt:" ;;
    esac
done
shift $[ $OPTIND - 1 ]
count=1
for param in "$@"
do
	echo "Parameter $count: $param"
	count=$[ $count + 1 ]
done
```

```
# ./test.sh -ac -b "22 33"  -d  test1 test2 test3
Found the -a option
Found the -c option
Found the -b option, with value 22 33
Unknown option: ?:
Parameter 1: test1
Parameter 2: test2
Parameter 3: test3
```

## 将选项标准化

| 选项 | 描述                             |
| ---- | -------------------------------- |
| -a   | 显示所有对象                     |
| -c   | 生成一个计数                     |
| -d   | 指定一个目录                     |
| -e   | 扩展一个对象                     |
| -f   | 指定读入数据的文件               |
| -h   | 显示命令的帮助信息               |
| -i   | 忽略文本大小写                   |
| -l   | 产生输出的长格式版本             |
| -n   | 使用非交互式模式（批处理）       |
| -o   | 将所有输出重定向到指定的输出文件 |
| -q   | 以安静模式运行                   |
| -r   | 递归地处理目录和文件             |
| -s   | 以安静模式运行                   |
| -v   | 生成详细输出                     |
| -x   | 排除某个对象                     |
| -y   | 对所有问题回答yes                |

## 获得用户输入

### 基本地读取

```shell
#!/bin/bash
read -p "Please enter your age: " age
days=$[ $age * 365 ]
echo "That makes you over $days days old!"

read -p "Enter your name: " first last
echo "Checking data for $last,$first..."
```

```
# ./test.sh 
Please enter your age: 30
That makes you over 10950 days old!
Enter your name: Rich Blum
Checking data for Blum,Rich...
```

也可以在read命令行中不指定变量。如果这样，read命令会将它收到的任何数据都放进特殊环境变量 REPLY 中。

### 超时

```shell
#!/bin/bash
read -n1 -p "Do you want to continue [Y/N]? " answer
case $answer in
    Y | y) echo
           echo "fine, continue on..." ;;
    N | n) echo
           echo "OK, goodbye"
           exit ;;
esac
echo "This is the end of the script."
```

```
# ./test.sh 
Do you want to continue [Y/N]? y
fine, continue on...
This is the end of the script.
```

如果使用计时方式格式如下：

```
read -t 5 -p "please enter: " name  # 时间单位是秒
```

### 隐藏方式读取

```shell
#!/bin/bash
read -s -p "Enter your password: " pass
echo
echo "Is your password really $pass"
```

```
# ./test.sh 
Enter your password: 
Is your password really abc
```

### 从文件中读取

```shell
#!/bin/bash
count=1
cat states.txt | while read line
do
	echo "Line $count: $line"
	count=$[ $count + 1 ]
done
echo "Finished processing the file"
```



