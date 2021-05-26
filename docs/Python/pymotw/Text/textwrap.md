# textwrap 文本段落格式化

## 示例数据

定义模块 ` textwrap_example.py ` ，其中包括字符串 ` sample_text ` 。

```
sample_text = '''
    The textwrap module can be used to format text for output in
    situations where pretty-printing is desired.  It offers
    programmatic functionality similar to the paragraph wrapping
    or filling features found in many text editors.
    '''
```

## 填充段落

函数 `fill()` 可以输入文字并输出用户要求格式的文本。

```python
import textwrap
from textwrap_example import sample_text

print(textwrap.fill(sample_text, width=50))
```

文本现在是左对齐，只有第一行保留了缩进，但是原来的每一行的末尾和下一行的开头之间仍有空格。

```
     The textwrap module can be used to format
text for output in     situations where pretty-
printing is desired.  It offers     programmatic
functionality similar to the paragraph wrapping
or filling features found in many text editors.
```

## 移除已有缩进

在之前的示例中，输出的文本中间夹杂着许多多余的空格，使得文本格式不是很整洁。使用 `dedent()` 函数移除所有示例文本中的空格前缀可以使结果更好，并且在移除自身代码格式的同时，允许直接从 Python 的代码中使用文档字符串或嵌入多行字符串。

```python
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
print('Dedented:')
print(dedented_text)
```

输出如下：

```
Dedented:

The textwrap module can be used to format text for output in
situations where pretty-printing is desired.  It offers
programmatic functionality similar to the paragraph wrapping
or filling features found in many text editors.
```

输出结果是一段删除了每一行中都存在的缩进空白文字。但是如果某一行比其他缩进的更多，多出的部分将不会被移除。

输出示例：

```
␣Line one.
␣␣␣Line two.
␣Line three.
```

输出示例：

```
Line one.
␣␣Line two.
Line three.
```

## 组合缩进及填充

```python
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text).strip()
for width in [45, 60]:
    print('{} Columns:\n'.format(width))
    print(textwrap.fill(dedented_text, width=width))
    print()
```

上面会以特定的宽度输出段落：

```
45 Columns:

The textwrap module can be used to format
text for output in situations where pretty-
printing is desired.  It offers programmatic
functionality similar to the paragraph
wrapping or filling features found in many
text editors.

60 Columns:

The textwrap module can be used to format text for output in
situations where pretty-printing is desired.  It offers
programmatic functionality similar to the paragraph wrapping
or filling features found in many text editors.
```

## 前缀块

用 `indent()` 函数在字符串每一行开头加入前缀文本。这个例子非常类是电子邮件回复被引用的部分，使用 `>` 符号来做每行文字的前缀。

```python
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
wrapped = textwrap.fill(dedented_text, width=50)
wrapped += '\n\nSecond paragraph after a blank line.'
final = textwrap.indent(wrapped, '> ')

print('Quoted block:\n')
print(final)
```

一段文字被分成了几行，每一行文字前都加了前缀，然后每行文字重新组成整个文字段落并返回。

```
Quoted block:

>  The textwrap module can be used to format text
> for output in situations where pretty-printing is
> desired.  It offers programmatic functionality
> similar to the paragraph wrapping or filling
> features found in many text editors.

> Second paragraph after a blank line.
```

要控制特定的一行接受新前缀，给 `indent()` 的 `predicate` 参数赋值。改操作会轮流遍历每行的文本，当值为真时将在改行加上前缀。

```python
import textwrap
from textwrap_example import sample_text

def should_indent(line):
    print('Indent {!r}?'.format(line))
    return len(line.strip()) % 2 == 0


dedented_text = textwrap.dedent(sample_text)
wrapped = textwrap.fill(dedented_text, width=50)
final = textwrap.indent(wrapped, 'EVEN ', predicate=should_indent)

print('\nQuoted block:\n')
print(final)
```

这个例子将在字符数为偶数的行加上 `EVEN` 前缀。

```
Indent ' The textwrap module can be used to format text\n'?
Indent 'for output in situations where pretty-printing is\n'?
Indent 'desired.  It offers programmatic functionality\n'?
Indent 'similar to the paragraph wrapping or filling\n'?
Indent 'features found in many text editors.'?

Quoted block:

EVEN  The textwrap module can be used to format text
for output in situations where pretty-printing is
EVEN desired.  It offers programmatic functionality
EVEN similar to the paragraph wrapping or filling
EVEN features found in many text editors.
```

## 悬挂缩进

可以设置输出段落的宽度，可以单独控制首行的缩进。

```python
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text).strip()
print(textwrap.fill(dedented_text, initial_indent='', subsequent_indent='*' * 4, width=50,))
```

可以产生一个悬挂缩进，缩进值这边定义的 `*` ，输出如下：

```
The textwrap module can be used to format text for
****output in situations where pretty-printing is
****desired.  It offers programmatic functionality
****similar to the paragraph wrapping or filling
****features found in many text editors.
```

## 减短长文本

为了查看长文本的摘要或预览，可以使用 `shorten()` 。所有的空格，比如制表符、换行符以及一系列的空格都将标准化为单个空格。然后此文本将减短为要求的长度来显示，在字词边界之间，将不包括不完整的词。

```python
import textwrap
from textwrap_example import sample_text

dedented_text = textwrap.dedent(sample_text)
original = textwrap.fill(dedented_text, width=50)

print('Original:\n')
print(original)

shortened = textwrap.shorten(original, 100)
shortened_wrapped = textwrap.fill(shortened, width=50)

print('\nShortened:\n')
print(shortened_wrapped)
```

如果非空字元在原文本中被当作减短的部分被移除，他将替换为占位符。默认值 `[...]` 可以被替换。

```
Original:

 The textwrap module can be used to format text
for output in situations where pretty-printing is
desired.  It offers programmatic functionality
similar to the paragraph wrapping or filling
features found in many text editors.

Shortened:

The textwrap module can be used to format text for
output in situations where pretty-printing [...]
```

