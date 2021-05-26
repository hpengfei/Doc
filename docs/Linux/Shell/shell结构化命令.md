# 结构化命令

## if-then

```shell
if command
then
  commands
fi
或者
if command; then
  commands
fi
```

if 语句会运行 if 后面的那个命令。如果该命令的退出状态码是 0 ，位于then 部分的命令就会被执行，如果改命令的退出状态码是其他值，then 部分的命令就不会被执行，bash shell 会继续执行脚本中的下一个命令。

```shell
#!/bin/bash

testuser=monitor
if grep $testuser /etc/passwd
then
   echo $testuser exist.
   ls -a /home/$testuser
fi
```

## if-then-else

```shell
if command
then
  commands
else
  commands
fi
```

当 if 语句中的命令返回退出状态码 0 时，then 部分中的部分会被执行，这跟普通的 if-then 语句一样。当 if 语句中的命令返回非零退出状态码时，bash shell 会执行 else 部分中的命令。

```shell
#!/bin/bash

testuser=monitor
if grep $testuser /etc/passwd
then
   echo $testuser exist.
   ls -a /home/$testuser
else
   echo $testuser does not exist.
fi
# ./test.sh 
monitor does not exist.
```

## 嵌套 if

```shell
if command
then
  commands
elif command
then 
  more commands
else
  commands
fi
```

elif 语句行提供了另一个要测试的命令，这类似于原始的 if 语句行。如果elif 后命令的退出状态码是 0，则bash 会执行第二个 then 语句部分的命令。如果elif 后命令的退出状态码是非 0，则执行 else 语句部分的命令。

在 elif 语句中，紧跟其后的 else 语句属于 elif 代码块。它们并不属于之前的 if-then 代码块。

## test 命令

test 命令提供了在 if-then 语句中测试不同条件的途径。如果 test 命令中列出的条件成立，test 命令就会退出并返回退出状态码 0。这样 if-then 语句就与其他编程语言中的 if-then 语句以类似的方式工作。如果条件不成立，test 命令就会退出并返回非零的退出状态码，这使得 if-then  语句不会在被执行。

```shell
test condition
```

在bash shell 中提供了另一种条件测试方法，无需在 if-then 语句声明 test 命令。

```shell
if [ condition ]
then
   commands
fi
```

### 数值比较

| 比较      | 描述                      |
| --------- | ------------------------- |
| n1 -eq n2 | 检查 n1 是否与 n2 相等    |
| n1 -ge n2 | 检查 n1 是否大于或等于 n2 |
| n1 -gt n2 | 检查 n1 是否大于 n2       |
| n1 -le n2 | 检查 n1 是小于或等于 n2   |
| n1 -lt n2 | 检查 n1 是否小于 n2       |
| n1 -ne n2 | 检查 n1 是否不等于 n2     |

```shell
#!/bin/bash
value1=10
value2=11
if [ $value1 -gt 5 ]
then
	echo "The test value $value1 is greater than 5."
fi

if [ $value1 -eq $value2 ]
then
	echo "The values are equal."
else
	echo "The values are different."
fi
```

```
The test value 10 is greater than 5.
The values are different.
```

说明：上面的比较常用来比较整数，对浮点数比较会有问题。

### 字符串比较

| 比较         | 描述                       |
| ------------ | -------------------------- |
| str1 = str2  | 检查 str1 是否和 str2 相同 |
| str1 != str2 | 检查 str1 是否和 str2 不同 |
| str1 < str2  | 检查 str1 是否比 str2 小   |
| str1 > str2  | 检查 str1 是否比 str2 大   |
| -n str1      | 检查 str1 的长度是否非0    |
| -z str1      | 检查 str1 的长度是否为0    |

问题点：

* 大于号和小于号必须转义，否则shell会把它们当作重定向符号，把字符串值当作文件名。
* 大于和小于顺序和sort命令所采用的不同。

```shell
#!/bin/bash
testuser=monitor
if [ $USER != $testuser ]
then
	echo "This is not $testuser."
else
	echo "Welcome $testuser."
fi
val1=Testing
val2=testing
if [ $val1 \> $val2 ]
then
	echo "$val1 is greater than $val2."
else
	echo "$val1 is less than $val2."
fi
if [ -n $var1 ]
then
	echo "The string '$val1' is not empty."
else
	echo "The string '$val1' is empty."
fi
if [ -z $var111]
then
	echo "The string '$var111' is empty."
else
	echo "The string '$var111' is not empty."
fi
```

```
This is not monitor.
Testing is less than testing.
The string 'Testing' is not empty.
The string '' is empty.
```

说明：空和未定义的变量会对shell脚本测试产生不可预知的问题，可以在使用变量前通过 -z 和 -n 判断变量是否含有值。

### 文件比较

| 比较            | 描述                                       |
| --------------- | ------------------------------------------ |
| -d file         | 检查 file 是否存在并是一个目录             |
| -e file         | 检查 file 是否存在                         |
| -f file         | 检查 file 是否存在并是一个文件             |
| -r file         | 检查 file 是否存在并可读                   |
| -s file         | 检查 file 是否存在并非空                   |
| -w file         | 检查 file 是否存在并可写                   |
| -x file         | 检查 file 是否存在并可执行                 |
| -O file         | 检查 file 是否存在并属于当前用户所有       |
| -G file         | 检查 file 是否存在并且默认组与当前用户相同 |
| file1 -nt file2 | 检查 file1 是否比 file2 新                 |
| file1 -ot file2 | 检查 file1 是否比 file2 旧                 |

### 复合条件测试

```shell
[ condition1 ] && [ condition2 ]
[ condition1 ] || [ condition2 ]
```

```shell
#!/bin/bash
if [ -d $HOME ] && [ -w $HOME/testing ]
then
	echo "The file exists and you can write to it."
else
	echo "I cannot write to the file."
fi
```

```
I cannot write to the file.
```

### 高级数字表达式的双括号

| 符号  | 描述     |
| ----- | -------- |
| val++ | 后增     |
| val-- | 后减     |
| ++val | 先增     |
| --val | 先减     |
| !     | 逻辑求反 |
| ~     | 位求反   |
| **    | 幂运算   |
| <<    | 左位移   |
| \>>   | 右位移   |
| &     | 位布尔和 |
| \|    | 位布尔或 |
| &&    | 逻辑和   |
| \|\|  | 逻辑或   |

```shell
#!/bin/bash
val1=10
if (( $val1 ** 2 > 90 ))
then
	(( val2 = $val1 ** 2 ))
	echo "The square of $val1 is $val2."
fi
```

```
The square of 10 is 100.
```

说明：可以在 if 语句中用双括号命令，也可以在脚本中的普通命令里使用来赋值。

### 高级字符串处理的双方括号

```shell
[[ expression ]]
```

双方括号里的 expression 使用了 test 命令中采用的标准字符串比较。但它提供了 test 命令未提供的另一种特性--模式匹配。

```shell
#!/bin/bash
if [[ $USER == r* ]]; then
	echo "Hello, $USER"
else
	echo "Sorry, I do not know you."
fi
```

```
Hello, root
```

## case 命令

```shell
case variable in
pattern1 | pattern2) commands;;
pattern3) commands;;
*)  commands;;
esac
```

case 命令会将指定的变量与不同的模式进行比较。如果变量和模式是匹配的，那么shell会执行为改模式的指定命令。可以通过竖线操作符在一行中分隔出多个模式。星号会捕获所有与已知模式不匹配的值。

## for 命令

### 读取列表中的复杂值

for 循环假定每个值都是用空格分割的。如果带有特殊符合可以使用 \ 做转义，也可以使用 “” 来处理。

```shell
for var in list
do
  commands
done
```

```shell
#!/bin/bash
for test in I don\'t know if "this'll" work "test word"
do
	echo "word: $test"
done
```

```
word: I
word: don't
word: know
word: if
word: this'll
word: work
word: test word
```

### 从变量读取列表

```shell
#!/bin/bash
list="CentOS Redhat Ubuntu"
list=$list" openSUSE"
for linux in $list
do
	echo "Linux: $linux"
done
```

说明：上面 list 变量第二次符值作用：是向变量中存储的已有文本字符串尾部添加文本的一种方法。

### 从命令读取值

```shell
#!/bin/bash
file="states.txt"
IFS=$'\n'
for state in $(cat $file)
do
	echo "Visit beautiful $state."
done
```

```
Visit beautiful Beijing.
Visit beautiful Shanghai.
Visit beautiful Henan zhengzhou.
Visit beautiful Henan xinyang.
Visit beautiful Henan luoyang.
```

技巧：

* 在写脚本时我们可以使用 `IFS.OLD=$IFS;  IFS=$'\n'`  这两条命令先修改 IFS变量并使用，使用完毕之后在用 `IFS=$IFS.OLD` 恢复 IFS 变量。
* `IFS=:` 这种用冒号作为分隔符，如果想指定多个分隔符例如：`IFS=$'\n':;"`  这个定义了 换行符、冒号、分号和双引号作为字段的分隔符。

### 通配符读取目录

```shell
#!/bin/bash
for file in /root/* /root/.b*
do
	if [ -d "$file" ]
	then
		echo "$file is a directory."
	elif [ -f "$file" ]
	then
		echo "$file is a file."
	fi
done
```

说明： $file 加双引号是为了避免目录或文件包含空格字符。

### C 风格的迭代计算

```shell
#!/bin/bash
for (( i = 1,j = 10 ; i <= 10; i++,j-- )); do
	echo "$i - $j"
done
```

## while 语句

```shell
while test command
do
  other commands
done
```

```shell
#!/bin/bash
var1=2
while echo $var1 
      [ $var1 -ge 0 ]
do
        echo "This is inside the loop"
        var1=$[ $var1 - 1 ]
done
```

```
2
This is inside the loop
1
This is inside the loop
0
This is inside the loop
-1
```

## until 命令

```shell
until test commands
do
  other commands
done
```

```shell
#!/bin/bash
var1=100
until echo "$var1"
       [ $var1 -eq 0 ]
do
	echo "Inside the loop: $var1"
	var1=$[ $var1 -25 ]
done
```

```
100
Inside the loop: 100
75
Inside the loop: 75
50
Inside the loop: 50
25
Inside the loop: 25
0
```

## 嵌套循环数字

```shell
#!/bin/bash
var1=3
while [ $var1 -ge 0 ]
do
	echo "Outer loop: $var1"
	for (( i = 1; i < 3; i++ ))
	do
		var2=$[ $i * $var1 ]
		echo "  Inner loop: $var1 * $i = $var2"
	done
	var1=$[ $var1 - 1 ]
done
```

```
Outer loop: 3
  Inner loop: 3 * 1 = 3
  Inner loop: 3 * 2 = 6
Outer loop: 2
  Inner loop: 2 * 1 = 2
  Inner loop: 2 * 2 = 4
Outer loop: 1
  Inner loop: 1 * 1 = 1
  Inner loop: 1 * 2 = 2
Outer loop: 0
  Inner loop: 0 * 1 = 0
  Inner loop: 0 * 2 = 0
```

## 嵌套循环处理文件数据

```shell
#!/bin/bash
IFS_OLD=$IFS
IFS=$'\n'
for entry in $(cat /etc/passwd)
do
	echo "Values in $entry -"
	IFS=:
	for value in $entry
	do
		echo "    $value"
	done
done
```

## 控制循环

### break

跳出内部循环

```shell
#!/bin/bash
for (( i = 1; i < 3; i++ ))
do
	echo "Outer loop: $i"
	for (( j = 1; j < 100; j++ ))
	do
		if [ $j -eq 5 ]
		then
			break
		fi
		echo "    Inner loop: $j"
	done
done
```

```
Outer loop: 1
    Inner loop: 1
    Inner loop: 2
    Inner loop: 3
    Inner loop: 4
Outer loop: 2
    Inner loop: 1
    Inner loop: 2
    Inner loop: 3
    Inner loop: 4
```

跳出外部循环

```shell
#!/bin/bash
for (( i = 1; i < 4; i++ ))
do
        echo "Outer loop: $i"
        for (( j = 1; j < 100; j++ ))
        do
                if [ $j -gt 4 ]
                then
                        break 2
                fi
                echo "    Inner loop: $j"
        done
done
```

```
Outer loop: 1
    Inner loop: 1
    Inner loop: 2
    Inner loop: 3
    Inner loop: 4
```

### continue

```shell
#!/bin/bash
for (( i = 1; i < 10; i++ ))
do
	if [ $i -gt 3 ] && [ $i -lt 8 ]
	then
		continue
	fi
	echo "Iteration number: $i"
done
```

```
Iteration number: 1
Iteration number: 2
Iteration number: 3
Iteration number: 8
Iteration number: 9
```

注意：当shell 执行continue命令时，它会跳过剩余的命令。如果在其中某个条件里对测试条件变量进行增值，问题就会出现。

```shell
#!/bin/bash
var1=0
while echo "while iteration: $var1"
      [ $var1 -lt 15 ]
do
	if [ $var1 -gt 3 ] && [ $var1 -lt 8 ]
	then
		continue
	fi
	echo "    Inside interation number: $var1"
	var1=$[ $var1 + 1 ]
done
```

```shell
#!/bin/bash
for (( a = 1; a <=5; a++ ))
do
	echo "Iteration $a:"
	for (( b = 1; b < 3; b++ ))
	do
		if [ $a -ge 2 ] && [ $a -le 4 ]
		then
			continue 2
		fi
		var3=$[ $a * $b ]
		echo "    The result of $a * $b is $var3"
	done
done
```

```
Iteration 1:
    The result of 1 * 1 is 1
    The result of 1 * 2 is 2
Iteration 2:
Iteration 3:
Iteration 4:
Iteration 5:
    The result of 5 * 1 is 5
    The result of 5 * 2 is 10
```

## 处理循环的输出

可以将输出重定向到指定文件：

```shell
#!/bin/bash
for file in /root/*
do
        if [ -d "$file" ]
        then
                echo "$file is a directory"
        elif [ -f "$file" ]
        then
                echo "$file is a file"
        fi
done > output.txt
```

可以将输出结果通过管道连接另外一个命令：

```shell
#!/bin/bash
for state in Beijing Shanghai "Henan zhengzhou" "Henan xinyang" "Henan luoyang"
do
	echo "$state is the net place to go"
done | sort
```