# collections 数据类型容器

`collections` 模块包含内置类型 `list`、`dict` 和 `tuple` 之上的数据类型。

## ChainMap 多字典查询

 `ChainMap` 类管理一系列字典，并按照给出的顺序搜索字典，以查找与见关联的值。`ChainMap` 是一个很好的 "Context" 容器，因为它可以被是为一个堆栈，随着堆栈的增长，更改会发生变化，而随着堆栈的缩小，这些更改将再次被丢弃。

### 访问值

`ChainMap` 支持与用于访问现有值的常规字典相同的 API。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}

m = collections.ChainMap(a, b)

print('Individual Values')
print('a = {}'.format(m['a']))
print('b = {}'.format(m['b']))
print('c = {}'.format(m['c']))
print()

print('Keys = {}'.format(list(m.keys())))
print('Values = {}'.format(list(m.values())))
print()

print('Items:')
for k, v in m.items():
    print(' {} = {}'.format(k, v))
print()

print('"d" in m: {}'.format(('d' in m)))
```

子映射按传递到构造函数的顺序进行搜索，因此为键 `c` 报告的值来自 `a` 字典。

```
Individual Values
a = A
b = B
c = C

Keys = ['b', 'c', 'a']
Values = ['B', 'C', 'A']

Items:
 b = B
 c = C
 a = A

"d" in m: False
```

### 重新排序

`ChainMap` 将其搜索的映射列表存储在其 maps 属性的列表中。该列表是可变，因此可以直接添加新的映射或更改元素的顺序以控制查找和更新行为。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}

m = collections.ChainMap(a, b)
print(m.maps)
print('c = {}\n'.format(m['c']))

m.maps = list(reversed(m.maps))

print(m.maps)
print('c = {}'.format(m['c']))
```

当映射列表反转时，与 `c` 关联的值将更改。

```
[{'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'}]
c = C

[{'b': 'B', 'c': 'D'}, {'a': 'A', 'c': 'C'}]
c = D
```

### 更新值

`ChainMap` 不会在子映射中缓存值。因此如果修改了它们的内容，则在访问 `ChainMap` 时将反映结果。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}

m = collections.ChainMap(a, b)
print('Before: {}'.format(m['c']))
a['c'] = 'E'
print('After : {}'.format(m['c']))
```

更改与现有键关联的值并添加新元素的方式相同。

```
Before: C
After : E
```

尽管实际上仅修改了链中的第一个映射，但也可以直接通过 `ChainMap` 设置值。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}

m = collections.ChainMap(a, b)
print('Before:', m)
m['c'] = 'E'
print('After :', m)
print('a', a)
```

使用 `m` 存储的新值时，将更新映射。

```
Before: ChainMap({'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'})
After : ChainMap({'a': 'A', 'c': 'E'}, {'b': 'B', 'c': 'D'})
a {'a': 'A', 'c': 'E'}
```

`ChainMap` 提供了一种方便的方法来创建新实例，并在 `maps` 列表的最前面添加一个额外的映射，从而可以轻松避免修改现有的基础数据结构。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}

m1 = collections.ChainMap(a, b)
m2 = m1.new_child()

print('m1 before:', m1)
print('m2 before:', m2)

m2['c'] = 'E'

print('m1 after:', m1)
print('m2 after:', m2)
```

这种堆叠行为使使用 `ChainMap` 实例作为模板或应用程序上下文很方便。特别是很容易在一个迭代中添加或更新值，然后为下一次迭代放弃更改。

```
m1 before: ChainMap({'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'})
m2 before: ChainMap({}, {'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'})
m1 after: ChainMap({'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'})
m2 after: ChainMap({'c': 'E'}, {'a': 'A', 'c': 'C'}, {'b': 'B', 'c': 'D'})
```

对于已知或预先建立新上下文的情况，也可以将映射传递给 `new_child()`。

```python
import collections

a = {'a': 'A', 'c': 'C'}
b = {'b': 'B', 'c': 'D'}
c = {'c': 'E'}

m1 = collections.ChainMap(a, b)
m2 = m1.new_child(c)   # m2 = collections.ChainMap(c, *m1.maps) 

print('m1["c"] = {}'.format(m1['c']))
print('m2["c"] = {}'.format(m2['c']))
```

结果为：

```python
m1["c"] = C
m2["c"] = E
```

## Counter 可哈希对象计数

计数器是一个容器，用于跟踪添加等效值得次数。它通常可用于实现与其它语言使用 `bag` 或 `multise` 数据结构的算法相同得算法。

### 初始化中

计数器支持三种初始化形式。可以使用一系列项目，包含键和计数的字典或使用将字符串名称映射到计数的关键字参数来调用其构造函数。

```python
import collections

print(collections.Counter(['a', 'b', 'c', 'a', 'b', 'b']))
print(collections.Counter({'a': 2, 'b': 3, 'c': 1}))
print(collections.Counter(a=2, b=3, c=1))
```

三种初始化形式的结果都相同。

```
Counter({'b': 3, 'a': 2, 'c': 1})
Counter({'b': 3, 'a': 2, 'c': 1})
Counter({'b': 3, 'a': 2, 'c': 1})
```

空的计数器可以不带任何参数构造，并可以通过 `update()` 方法填充。

```python
import collections

c = collections.Counter()
print('Initial :', c)

c.update('abcdaab')
print('Sequence:', c)

c.update({'a': 1, 'd': 5})
print('Dict    :', c)
```

计数值基于新数据增加，而不是被替换。在前面的示例中，a 的计数从 3 到 4 。

```
Initial : Counter()
Sequence: Counter({'a': 3, 'b': 2, 'c': 1, 'd': 1})
Dict    : Counter({'d': 6, 'a': 4, 'b': 2, 'c': 1})
```

### 访问计数

一旦填充了 `Counter` ，就可以使用字典 API 检索其值。

```python
import collections

c = collections.Counter('abcdaab')

for letter in 'abcde':
    print('{} : {}'.format(letter, c[letter]))
```

`Counter` 不会针对未知项目引发 `KeyError`。如果未在输入中看到值（本例中的 `e` 所示），则其计数为 0 。

```
a : 3
b : 2
c : 1
d : 1
e : 0
```

`elements()` 方法返回一个迭代器，该迭代器生成 `Counter` 已知的所有项目。

```python
import collections

c = collections.Counter('extremely')
c['z'] = 0
print(c)
print(list(c.elements()))
```

不保证元素的顺序，不包括计数小于或等于零的项目。

```
Counter({'e': 3, 'x': 1, 't': 1, 'r': 1, 'm': 1, 'l': 1, 'y': 1, 'z': 0})
['e', 'e', 'e', 'x', 't', 'r', 'm', 'l', 'y']
```

使用 `most_common()` 生成 n 个最常遇到的输入值及其各自计数的序列。

```python
import collections

c = collections.Counter()
with open('words', 'rt') as f:
    for line in f:
        c.update(line.rstrip().lower())

print('Most common:')
for letter, count in c.most_common(3):
    print('{}: {:>7}'.format(letter, count))
```

本示例对出现在系统单词中所有单词中的字母进行计数以产生频率分布，然后打印三个最常见的字母。将参数保留给 ` most_common() ` 会按频率顺序生成所有项目的列表。

```
Most common:
e:  235331
i:  201032
a:  199554
```

### 算术

计数器实例支持算术运算和设置运算以汇总结果。此示例显示了用于创建新计数器实例的标准运算符，但也支持就地运算符  `+=`, `-=`, `&=`,  和 |= 。

```python
import collections

c1 = collections.Counter(['a', 'b', 'c', 'a', 'b', 'b'])
c2 = collections.Counter('alphabet')

print('C1:', c1)
print('C2:', c2)

print('\nCombined counts:')
print(c1 + c2)

print('\nSubtraction:')
print(c1 - c2)

print('\nIntersection (taking positive minimums):')
print(c1 & c2)

print('\nUnion (taking maximums):')
print(c1 | c2)
```

每次通过操作产生新的计数器时，任何计数为零或为负的项目都将被丢弃。`a` 的计数在 c1 和 c2 中相同，因此减法将其保留为零。

```
C1: Counter({'b': 3, 'a': 2, 'c': 1})
C2: Counter({'a': 2, 'l': 1, 'p': 1, 'h': 1, 'b': 1, 'e': 1, 't': 1})

Combined counts:
Counter({'a': 4, 'b': 4, 'c': 1, 'l': 1, 'p': 1, 'h': 1, 'e': 1, 't': 1})

Subtraction:
Counter({'b': 2, 'c': 1})

Intersection (taking positive minimums):
Counter({'a': 2, 'b': 1})

Union (taking maximums):
Counter({'b': 3, 'a': 2, 'c': 1, 'l': 1, 'p': 1, 'h': 1, 'e': 1, 't': 1})
```

## defaultdict  缺少键返回默认值

标准字典包含 `setdefault()` 方法吗，该方法用于检索值并在该值不存在时建立默认值。相比之下，defaultdict 允许调用在初始化容器时预先指定默认值。

```python
import collections

def default_factory():
    return 'default value'


d = collections.defaultdict(default_factory, foo='bar')
print('d:', d)
print('foo =>', d['foo'])
print('bar =>', d['bar'])
```

只要适合所有键具有相同默认值的方法，此方法就可以很好的工作。如果默认值是用于聚合或累积值的类型（例如列表，集合，甚至 int），则它特别有用。标准库文档包括几个示例，其中以这种方式使用 defaultdict 。

```
d: defaultdict(<function default_factory at 0x000002D091C92828>, {'foo': 'bar'})
foo => bar
bar => default value
```

## deque 双端队列

双端队列 （deque） 支持从队列的任意一端添加和删除元素。更常用的堆栈和队列是双端队列的退化形式，其中输入和输出限制为单端。

```python
import collections

d = collections.deque('abcdefg')
print('Deque:', d)
print('Length:', len(d))
print('Left end:', d[0])
print('Right end:', d[-1])

d.remove('c')
print('remove(c)', d)
```

由于双端队列是序列容器的一种，因此它们支持一些与列表相同的操作，例如使用 ` __getitem__() `检查内容，确定长度以及通过匹配标识从队列中间删除元素。

```
Deque: deque(['a', 'b', 'c', 'd', 'e', 'f', 'g'])
Length: 7
Left end: a
Right end: g
remove(c) deque(['a', 'b', 'd', 'e', 'f', 'g'])
```

### 填充

可以从任一端填充双端队列，在 Python 实现中称为 `left` 和 `right`。

```python
import collections

d1 = collections.deque()
d1.extend('abcdefg')
print('extend    :', d1)
d1.append('h')
print('append    :', d1)

d2 = collections.deque()
d2.extendleft(range(6))
print('extendleft:', d2)
d2.appendleft(6)
print('appendleft:', d2)
```

`extendleft()` 函数在其输入上进行迭代，并为每个项目执行相当于 `appendleft()` 的操作。最终结果是双端队列包含相反顺序的输入序列。

```
extend    : deque(['a', 'b', 'c', 'd', 'e', 'f', 'g'])
append    : deque(['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h'])
extendleft: deque([5, 4, 3, 2, 1, 0])
appendleft: deque([6, 5, 4, 3, 2, 1, 0])
```

### Consuming

同样，取决于所应用的算法，双端队列的元素可以从两端或任一端使用。

```python
import collections

print('From the right:')
d = collections.deque('abcdefg')
while True:
    try:
        print(d.pop(), end='')
    except IndexError:
        break
print()

print('\nFrom the left:')
d = collections.deque(range(6))
while True:
    try:
        print(d.popleft(), end='')
    except IndexError:
        break
print()
```

使用 `pop()` 从双端队列的 "右" 端删除一个项目，使用 `popleft()` 从 “左” 端取一个项目。

```
From the right:
gfedcba

From the left:
012345
```

由于双端队列是线程安全的，因此甚至可以从单独的线程同时从两端消费内容。

```python
import collections
import threading
import time

candle = collections.deque(range(5))


def burn(direction, nextSource):
    while True:
        try:
            next = nextSource()
        except IndexError:
            break
        else:
            print(' {:>8}: {}'.format(direction, next))
            time.sleep(0.1)
    print(' {:>8} done'.format(direction))
    return


left = threading.Thread(target=burn, args=('Left', candle.popleft))
right = threading.Thread(target=burn, args=('Right', candle.pop))

left.start()
right.start()

left.join()
right.join()
```

在此示例中，线程在两端之间交替，删除项，直到双端队列为空。

注意：该结果是在 Linux 系统中运行的。

```
     Left: 0
    Right: 4
     Left: 1
    Right: 3
     Left: 2
    Right done
     Left done
```

### Rotating

双端队列的另一个用用方面是能够沿任一方向旋转，从而可以跳过某些项目。

```python
import collections

d = collections.deque(range(10))
print('Normal        :', d)

d = collections.deque(range(10))
d.rotate(2)
print('Right rotation:', d)

d = collections.deque(range(10))
d.rotate(-2)
print('Left rotation :', d)
```

将双端队列向右旋转（使用正方向旋转）可将项目从右端移至左端。向左旋转（值为负）会从左端获取项目并将其移动右端。

```
Normal        : deque([0, 1, 2, 3, 4, 5, 6, 7, 8, 9])
Right rotation: deque([8, 9, 0, 1, 2, 3, 4, 5, 6, 7])
Left rotation : deque([2, 3, 4, 5, 6, 7, 8, 9, 0, 1])
```

### 限制队列大小

可以将双端队列实例配置为具有最大长度，以使其永远不会超过该大小。当队列达到指定的长度时，现有项目将随着新项目的添加而被丢弃。此行为对于在长度不确定的流中查找最后 n 个项目很有用。

```python
import collections
import random

random.seed(1)

d1 = collections.deque(maxlen=3)
d2 = collections.deque(maxlen=3)

for i in range(5):
    n = random.randint(0, 100)
    print('n =', n)
    d1.append(n)
    d2.appendleft(n)
    print('D1:', d1)
    print('D2:', d2)

```

无论将项目添加到哪个端，都保持双端队列长度。

```
n = 17
D1: deque([17], maxlen=3)
D2: deque([17], maxlen=3)
n = 72
D1: deque([17, 72], maxlen=3)
D2: deque([72, 17], maxlen=3)
n = 97
D1: deque([17, 72, 97], maxlen=3)
D2: deque([97, 72, 17], maxlen=3)
n = 8
D1: deque([72, 97, 8], maxlen=3)
D2: deque([8, 97, 72], maxlen=3)
n = 32
D1: deque([97, 8, 32], maxlen=3)
D2: deque([32, 8, 97], maxlen=3)
```

## namedtuple 具有命名字段的元组子类

标准元组使用数字索引访问其成员。

```python

```









