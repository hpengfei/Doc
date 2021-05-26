# difflib 字符串比较

`difflib` 模块包含用来计算字符序列之间差异的工具。它在比较文本方面十分有效，同时包括使用集中常见差异格式生成报告的功能。

下面是示例所使用的测试数据，保存在 `difflib_data.py` 文件中。

```python
text1 = """Lorem ipsum dolor sit amet, consectetuer adipiscing
elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
pharetra tortor.  In nec mauris eget magna consequat
convalis. Nam sed sem vitae odio pellentesque interdum. Sed
consequat viverra nisl. Suspendisse arcu metus, blandit quis,
rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
tristique vel, mauris. Curabitur vel lorem id nisl porta
adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
tristique enim. Donec quis lectus a justo imperdiet tempus."""

text1_lines = text1.splitlines()

text2 = """Lorem ipsum dolor sit amet, consectetuer adipiscing
elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
pharetra tortor. In nec mauris eget magna consequat
convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
consequat viverra nisl. Suspendisse arcu metus, blandit quis,
rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
tristique vel, mauris. Curabitur vel lorem id nisl porta
adipiscing. Duis vulputate tristique enim. Donec quis lectus a
justo imperdiet tempus.  Suspendisse eu lectus. In nunc."""

text2_lines = text2.splitlines()
```

## 比较文本体

`Differ` 类适用于文本行序列并产生人类可读的增量，或更改指令，包括各行内的差异。`Differ` 生成的默认输出类是于 Unix 下的 `diff` 命令行工具。它包括来自两个列表的原始输入值（包括公共值）和标记数据，以指示进行了哪些更改。

* 以 `-` 为前缀的行在第一个序列中，但不在第二个序列中。
* 以 `+` 为前缀的行在第二个序列中，但不在第一个序列中。
* 如果某行之间的版本之间存在增量差异，则使用前缀为 `?` 的额外行来突出显示新版本中的更改。
* 如果一条线没有改变，则在左列上打印一个额外的空白区域，使其与可能存在差异的另一个输出对齐。

在将文本传递给 `compare()` 之前将文本分解为一系列单独的行会生成比传入大字符串更可读的输出。

```python
import difflib
from difflib_data import *

d = difflib.Differ()
diff = d.compare(text1_lines, text2_lines)
print('\n'.join(diff))
```

样本数据中两个文本段的开头是相同的，因此第一二行打印时没有任何额外的注释。

```
  Lorem ipsum dolor sit amet, consectetuer adipiscing
  elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
```

数据第三行已更改为在修改后包含逗号的文本。改行的两个版本都打印出来，第 5 行的额外信息显示了修改文本的列，包括添加了 `,` 字符的事实。

```
- pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
+ pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
?         +
```

输出的下几行显示删除了额外的空间。

```
- pharetra tortor.  In nec mauris eget magna consequat
?                 -

+ pharetra tortor. In nec mauris eget magna consequat
```

接下来，进行了更复杂的更改，替换了短语中的多个单词。

```
- convalis. Nam sed sem vitae odio pellentesque interdum. Sed
?                 - --

+ convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
?               +++ +++++   +
```

段落中的最后一句被明显的改变，因此通过删除旧版本并添加新版本来表示差异。

```
  consequat viverra nisl. Suspendisse arcu metus, blandit quis,
  rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
  molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
  tristique vel, mauris. Curabitur vel lorem id nisl porta
- adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
- tristique enim. Donec quis lectus a justo imperdiet tempus.
+ adipiscing. Duis vulputate tristique enim. Donec quis lectus a
+ justo imperdiet tempus.  Suspendisse eu lectus. In nunc.
```

`ndiff()` 函数产生基本相同的输出。该处理专门用于处理文本数据并消除输入中的“noise”。

## 其他格式输出

虽然 `Differ` 类展示了所有的输入行，unified diff 仅包括修改过的行和一些上下文。`unified_diff()` 函数产生这种输出。

```python
import difflib
from difflib_data import *

diff = difflib.unified_diff(text1_lines, text2_lines, lineterm='',)
print('\n'.join(diff))
```

`lineterm` 参数用于告诉 `unified_diff()` 跳过追加新行到它返回的控制行，因为输入行不包含它们。打印时，新行将添加到所有行。

```
--- 
+++ 
@@ -1,11 +1,11 @@
 Lorem ipsum dolor sit amet, consectetuer adipiscing
 elit. Integer eu lacus accumsan arcu fermentum euismod. Donec
-pulvinar porttitor tellus. Aliquam venenatis. Donec facilisis
-pharetra tortor.  In nec mauris eget magna consequat
-convalis. Nam sed sem vitae odio pellentesque interdum. Sed
+pulvinar, porttitor tellus. Aliquam venenatis. Donec facilisis
+pharetra tortor. In nec mauris eget magna consequat
+convalis. Nam cras vitae mi vitae odio pellentesque interdum. Sed
 consequat viverra nisl. Suspendisse arcu metus, blandit quis,
 rhoncus ac, pharetra eget, velit. Mauris urna. Morbi nonummy
 molestie orci. Praesent nisi elit, fringilla ac, suscipit non,
 tristique vel, mauris. Curabitur vel lorem id nisl porta
-adipiscing. Suspendisse eu lectus. In nunc. Duis vulputate
-tristique enim. Donec quis lectus a justo imperdiet tempus.
+adipiscing. Duis vulputate tristique enim. Donec quis lectus a
+justo imperdiet tempus.  Suspendisse eu lectus. In nunc.
```

使用 `context_diff()` 产生类似的可读输出。

## 垃圾数据

生成差异序列的所有函数都接受参数，以指示应忽略哪些行以及应忽略行中的哪些字符。例如，这些参数可用于跳过一个文件的两个版本中的标记或空白变化。

```python
from difflib import SequenceMatcher

def show_results(match):
    print('  a    = {}'.format(match.a))
    print('  b    = {}'.format(match.b))
    print('  size = {}'.format(match.size))
    i, j, k = match
    print('  A[a:a+size] = {!r}'.format(A[i:j+k]))
    print('  B[b:b+size] = {!r}'.format(B[j:j+k]))

A = " abcd"
B = "abcd abcd"

print('A = {!r}'.format(A))
print('B = {!r}'.format(B))

print('\nWithout junk detection:')
s1 = SequenceMatcher(None, A, B)
match1 = s1.find_longest_match(0, len(A), 0, len(B))
show_results(match1)

print('\nTreat spaces as junk:')
s2 = SequenceMatcher(lambda x: x == " ", A, B)
match2 = s2.find_longest_match(0, len(A), 0, len(B))
show_results(match2)
```

`Differ` 的默认设置时不要忽略任何行或明确的字符，而是依赖于 `SequenceMatcher` 检测噪声的能力。`ndiff()` 默认忽略空白符和制表符。

```
A = ' abcd'
B = 'abcd abcd'

Without junk detection:
  a    = 0
  b    = 4
  size = 5
  A[a:a+size] = ' abcd'
  B[b:b+size] = ' abcd'

Treat spaces as junk:
  a    = 1
  b    = 0
  size = 4
  A[a:a+size] = 'abc'
  B[b:b+size] = 'abcd'
```

## 比较任意类型

`SequenceMatcher` 类比较任意类型的两个序列，只要值是可散列的。它使用一种算法来识别序列中最长的连续匹配块，消除了对真实数据没用的 “垃圾”值。

函数 `get_opcodes()`返回一个指令列表，用于修改第一个序列以使其与第二个序列匹配。指令被编码为五元素组，包括一个字符串指令（[操作码]）和两队开始和停止索引到序列中。

| 操作码    | 定义                           |
| --------- | ------------------------------ |
| 'replace' | 将`a[i1:i2]` 替换为 `b[j1:j2]` |
| 'delete'  | 完全移除`a[i1:i2]` entirely    |
| 'insert'  | 将`b[j1:j2]` 插入到 `a[i1:i1]` |
| 'equal'   | 子序列完全相等                 |

```python
import difflib

s1 = [1, 2, 3, 5, 6, 4]
s2 = [2, 3, 5, 4, 6, 1]

print('Initial data:')
print('s1 =', s1)
print('s2 =', s2)
print('s1 == s2', s1 == s2)
print()

matcher = difflib.SequenceMatcher(None, s1, s2)
for tag, i1, i2, j1, j2 in reversed(matcher.get_opcodes()):

    if tag == 'delete':
        print('Remove {} from positions [{}:{}]'.format(s1[i1:i2], i1, i2))
        print('  before =', s1)
        del s1[i1:i2]

    elif tag == 'equal':
        print('s1[{}:{}] and s2[{}:{}] are the same'.format(i1, i2, j1, j2))

    elif tag == 'insert':
        print('Insert {} from s2[{}:{}] are the same'.format(s2[j1:j2], j1, j2, i1))
        print('  before =', s1)
        s1[i1:i2] = s2[j1:j2]

    elif tag == 'replace':
        print(('Replace {} from s1[{}:{}] '
              'with {} from s2[{}:{}]').format(s1[i1:i2], i1, i2, s2[j1:j2], j1, j2))
        print('  before =', s1)
        s1[i1:i2] = s2[j1:j2]

    print('  after =', s1, '\n')

print('s1 == s2', s1 == s2)
```

此示例比较两个整数列表，并使用 `get__opcodes()` 派生将原始列表转换为较新版本的指令。以相反的顺序修改，以便在添加和删除项目后列表索引保持准确。

```
Initial data:
s1 = [1, 2, 3, 5, 6, 4]
s2 = [2, 3, 5, 4, 6, 1]
s1 == s2 False

Replace [4] from s1[5:6] with [1] from s2[5:6]
  before = [1, 2, 3, 5, 6, 4]
  after = [1, 2, 3, 5, 6, 1] 

s1[4:5] and s2[4:5] are the same
  after = [1, 2, 3, 5, 6, 1] 

Insert [4] from s2[3:4] are the same
  before = [1, 2, 3, 5, 6, 1]
  after = [1, 2, 3, 5, 4, 6, 1] 

s1[1:4] and s2[0:3] are the same
  after = [1, 2, 3, 5, 4, 6, 1] 

Remove [1] from positions [0:1]
  before = [1, 2, 3, 5, 4, 6, 1]
  after = [2, 3, 5, 4, 6, 1] 

s1 == s2 True
```

`SequenceMatcher` 适用于自定义类以及内置类型，只要它们是可散列的。