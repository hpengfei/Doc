# 函数

## 基本的脚本函数

### 创建函数

```shell
function name {
	commands
}
```

```shell
name() {
    commands
}
```

### 使用函数

为了养成良好的编写习惯，函数的使用最好养成如下几个习惯：

* 先定义后使用
* 函数名必须是唯一的，请勿重复使用函数名
* 函数名做到见名知意

```shell
#!/bin/bash
function func1 {
	echo "This is an example of a function."
}
count=1
while [ $count -le 2 ]
do
	func1
	count=$[ $count + 1 ]
done
echo "This is the end of the loop."
func1
echo "Now this is the end of the script."
```

```
# ./test.sh 
This is an example of a function.
This is an example of a function.
This is the end of the loop.
This is an example of a function.
Now this is the end of the script.
```

## 返回值

### 默认退出状态码

默认情况下，函数的退出状态码是函数中最后一条命令返回的退出状态码。

```shell
#!/bin/bash
func1() {
	echo "trying to display a non-existent file"
	ls -l badfile
}
func2() {
	ls -l badfile
	echo "trying to display a non-existent file"
}
echo "test the function: "
func1
echo "The func1 exit status is: $?"
func2
echo "The func2 exit status is: $?"
```

```
# ./test.sh 
test the function: 
trying to display a non-existent file
ls: cannot access badfile: No such file or directory
The func1 exit status is: 2
ls: cannot access badfile: No such file or directory
trying to display a non-existent file
The func2 exit status is: 0
```

### 使用 return 命令

bash shell 中使用return命令来退出函数并返回特定的退出状态码。return命令允许指定一个整数值来定义函数的退出状态码，从而提供了一种简单的途径来编程设定函数退出状态码。

* 函数一结束就取返回值
* 退出状态码必须是0~255

### 使用函数输出

```shell
#!/bin/bash
function db1 {
	read -p "Enter a value: " value
	echo $[ $value * 2 ]
}
result=$(db1)
echo "The new value is $result"
```

```
# ./test.sh 
Enter a value: 50
The new value is 100
```

## 在函数中使用变量

### 向函数传递参数

```shell
#!/bin/bash
function addem {
	if [ $# -eq 0 ] || [ $# -gt 2 ]
	then
		echo "-1"
	elif [ $# -eq 1 ]
	then
		echo $[ $1 + $1 ]
	else
		echo $[ $1 + $2 ]
	fi
}
echo -n "Adding 10 and 15: "
value=$(addem 10 15)
echo $value
echo -n "Let's try adding just one number: "
value=$(addem 10)
echo $value
echo -n "Now trying adding no numbers: "
value=$(addem)
echo $value
echo -n "Finally, try adding tree numbers:"
value=$(addem 10 15 20)
echo $value
```

```
# ./test.sh 
Adding 10 and 15: 25
Let's try adding just one number: 20
Now trying adding no numbers: -1
Finally, try adding tree numbers:-1
```

```shell
#!/bin/bash
function func {
	echo $[ $1 * $2 ]
}
if [ $# -eq 2 ]
then
	value=$(func $1 $2)
	echo "The result is $value"
else
	echo "Usage: script a b"
fi
```

```
# ./test.sh 10 15
The result is 150
```

### 在函数中处理变量

#### 全局变量

全局变量是在shell脚本中任何地方都有效的变量。如果你在脚本的主体部分定义了一个全局变量，那么可以在函数内读取它的值。类似的如果你在函数内定义了一个全局变量，可以在脚本的主体部分读取它的值。

```shell
#!/bin/bash
function db1 {
	value=$[ $value * 2 ]
}
read -p "Enter a value: " value
db1
echo "The new value is: $value"
```

```
# ./test.sh 
Enter a value: 10
The new value is: 20
```

如果变量在函数内被赋予了新值，那么在脚本中引用该变量时，新值也依然有效。但是这种用法很危险容易产生不可预知的问题。

```shell
#!/bin/bash
function func {
	temp=$[ $value + 5 ]
	result=$[ $temp * 2 ]
}
temp=4
value=6
func
echo "The result is $result"
if [ $temp -gt $value ]
then
	echo "temp is larger"
else
	echo "temp is smaller"
fi
```

```
# ./test.sh 
The result is 22
temp is larger
```

由于函数中用到了`$temp` 变量，它的值在脚本使用时收到了影响，产生了意想不到的后果。

#### 局部变量

无需在函数中使用全局变量，函数内部使用的任何变量都可以被声明成局部变量。要实现这一点，只要在变量声明的前面加上local关键字就可以了。

```shell
#!/bin/bash
function func {
	local temp=$[ $value + 5 ]
	result=$[ $temp * 2 ]
}
temp=4
value=6
func
echo "The result is $result"
if [ $temp -gt $value ]
then
	echo "temp is larger"
else
	echo "temp is smaller"
fi
```

```
# ./test.sh 
The result is 22
temp is smaller
```

## 数组变量和函数

如果将数组变量当作函数参数，函数只会取数组变量的第一个值。

```shell
#!/bin/bash
function testit {
	echo "The parameters are: $@"
	thisarray=$1
	echo "The received array is ${thisarray[*]}"
}
myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
testit $myarray
```

```
# ./test.sh 
The original array is: 1 2 3 4 5
The parameters are: 1
The received array is 1
```

addarray函数会遍历所有的数组元素，将它们累加在一起。你可以在myarray数组变量中放置任意多的值，addarray函数会将它们都加起来。

```shell
#!/bin/bash
function addarray {
	local sum=0
	local newarray
	newarray=($(echo "$@"))
	for value in ${newarray[*]}
	do
		sum=$[ $sum + $value ]
	done
	echo $sum
}
myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
arg1=$(echo ${myarray[*]})
result=$(addarray $arg1)
echo "The result is $result"
```

```
# ./test.sh 
The original array is: 1 2 3 4 5
The result is 15
```

### 从函数返回数组

```shell
#!/bin/bash
function arraydblr {
	local origarray
	local newarray
	local elements
	local i
	origarray=($(echo "$@"))
	newarray=($(echo "$@"))
	elements=$[ $# -1 ]
	for (( i=0; i<=$elements; i++ )) {
		newarray[$i]=$[ ${origarray[$i]} * 2 ]
	}
	echo ${newarray[*]}
}
myarray=(1 2 3 4 5)
echo "The original array is: ${myarray[*]}"
arg1=$(echo ${myarray[*]})
result=($(arraydblr $arg1))
echo "The new array is: ${result[*]}"
```

```
# ./test.sh 
The original array is: 1 2 3 4 5
The new array is: 2 4 6 8 10
```

该脚本用$arg1 变量将数组值传给arraydblr函数。arraydblr函数将该数组重组到新的数组变量中，生成该输出数组变量的一个副本。然后对数据元素进行遍历，将每个元素值翻倍，并将结果存入函数中该数组变量的副本。

arraydblr函数使用echo语句来输出每个数组元素的值。脚本用arraydblr函数的输出来重新生成一个新的数组变量。

## 函数递归

```shell
#!/bin/bash
function factorial {
	if [ $1 -eq 1 ]
	then
		echo 1
	else
		local temp=$[ $1 -1 ]
		local result=$(factorial $temp)
		echo $[ $result * $1 ]
	fi
}
read -p "Enter value: " value
result=$(factorial $value)
echo "The factorial of $value is: $result"
```

```
# ./test.sh 
Enter value: 5
The factorial of 5 is: 120
```

