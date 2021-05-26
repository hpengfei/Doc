# re 正则表达式

## 在文本中查找模式

`re` 最常见的用法就是使用文本中查找功能。`search()` 函数接受目标模式和要扫描的文本，当找到模式时返回一个 `Match` 对象，如果没能找到目标模式， `search()` 返回 `None`。

每一个 `Match` 对象都包括了匹配到的对象各种信息，包括原始输入字符串，使用的正则表达式，和这一模式在原始字符串中出现的位置。

```python
import re

pattern = 'this'
text = 'Does this text match the pattern?'

match = re.search(pattern, text)

s = match.start()
e = match.end()

print('Found "{}"\nin "{}"\nfrom {} to {} ("{}")'.format(match.re.pattern, match.string, s, e, text[s:e]))
```

`start()` 和 `end()` 方法以字符串索引形式展示了模式是在文本中的何处匹配的。

```
Found "this"
in "Does this text match the pattern?"
from 5 to 9 ("this")
```

## 编译表达式

将程序使用表达式进行 `compile` 从而可以提升效率。`compile()` 函数将一个表达式字符串转换为一个 ` RegexObject `。

```python
regexes = [re.compile(p) for p in ['this', 'that']]
text = 'Does this text match the pattern?'

print('Text: {!r}\n'.format(text))

for regex in regexes:
    print('Seeking "{}" ->'.format(regex.pattern), end=' ')

    if regex.search(text):
        print('match!')
    else:
        print('no match!')
```

模块级函数在缓存中存储已编译过的表达式，但该缓存的大小是有限的，并且直接使用经过编译的表达式可以避免查找缓存所需要的额外开销。使用已编译过的表达式的另一个优点是通过在模块被装载时预编译全部的表达式，编译工作就可转变为在程序启动的时候进行，而不是可能在程序响应用户操作的时候进行。

```
Text: 'Does this text match the pattern?'

Seeking "this" -> match!
Seeking "that" -> no match!
```

##  多重匹配

前面的示例全部是使用 `search()` 来在字面文本字符串中进行单一情况的查找。而 `findall()` 函数在不重复的情况下返回全部的匹配输入模式的字串。

```python
import re

text = 'abbaaabbbbaaaaa'
pattern = 'ab'

for match in re.findall(pattern, text):
    print('Found {!r}'.format(match))
```

示例中输入的字符串包含两个 `ab`。

```
Found 'ab'
Found 'ab'
```

与 `findall()` 返回字符串不同的是，`findter()` 函数返回一个产生 `Match` 实例的迭代器。

```python
import re

text = 'abbaaabbbbaaaaa'
pattern = 'ab'

for match in re.finditer(pattern, text):
    s = match.start()
    e = match.end()
    print('Found {!r} at {:d}:{:d}'.format(text[s:e], s, e))
```

上面的示例能找到两个相同的 `ab`， 同时 `Matcch` 实例将显示出这两个 `ab` 是在原始输入的何处找到的。

```
Found 'ab' at 0:2
Found 'ab' at 5:7
```

## 模式语法

正则表达式支持比简单的字面文本字符串更加强大的匹配模式。模式能够重复，能够被锚定到输入中不同的逻辑位置，还能够被表达成简洁的不需要模式中每一个字面值字符都出现的形式。这些功能需要将字面文本值与通过 `re` 实现的作为正则表达式模式语法的一部分的元字符相结合来使用。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return


test_patterns('abbaaabbbbaaaaa', [('ab', "'a', followed by 'b'")])
```

接下来的例子将使用 `test_patterns()` 来探索随着模式的变化，同一个输入文本的匹配结果是如何变化的。输出结果显示了输入的文本以及根据输入中的各个不同部分匹配模式的子串的变化。

```
'ab' ('a', followed by 'b')

  'abbaaabbbbaaaaa'
  'ab'
  .....'ab'
```

## 重复

在一个模式中有五种方式来表示重复。一个以元字符 `*` 跟随的模式被重复 0 次或更多次（允许一个模式重复 0 次意味着其并不需要出现便可以匹配）。如果以 `+` 替代  `*` ，则模式必须至少出现一次。使用 `?` 意味着模式出现 0 次或者 1 次。若想表示特定的出现次数，在模式后使用 `{m}` ，其中 `{m}` 是模式重复的次数。最后要想允许一个可变但有限的重复次数使用 `{m, n}` ，其中 `m` 是重复次数的最小值而 `n` 是重复次数的最大值。省略 `n` （`{m,}`） 意味着该值必须出现至少 `m` 次，并没有上限。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return


test_patterns(
    'abbaabbba',
    [('ab*', 'a followed by zero or more b'),
     ('ab+', 'a followed by one or more b'),
     ('ab?', 'a followed by zero or one b'),
     ('ab{3}', 'a followed by three b'),
     ('ab{2,3}', 'a followed by two to three b')],
)
```

结果如下：

```
'ab*' (a followed by zero or more b)

  'abbaabbba'
  'abb'
  ...'a'
  ....'abbb'
  ........'a'

'ab+' (a followed by one or more b)

  'abbaabbba'
  'abb'
  ....'abbb'

'ab?' (a followed by zero or one b)

  'abbaabbba'
  'ab'
  ...'a'
  ....'ab'
  ........'a'

'ab{3}' (a followed by three b)

  'abbaabbba'
  ....'abbb'

'ab{2,3}' (a followed by two to three b)

  'abbaabbba'
  'abb'
  ....'abbb'
```

上面的匹配使用了默认的 贪婪模式进行匹配，如果想禁用该模式则可以通过在重复指令后跟随 `?` 来禁用。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return


test_patterns(
    'abbaabbba',
    [('ab*?', 'a followed by zero or more b'),
     ('ab+?', 'a followed by one or more b'),
     ('ab??', 'a followed by zero or one b'),
     ('ab{3}?', 'a followed by three b'),
     ('ab{2,3}?', 'a followed by two to three b')],
)
```

为任一允许 `b` 出现 0 次的模式禁用输入贪婪消耗意味着匹配到的子字符串将不包括任何 `b` 字符。

```
'ab*?' (a followed by zero or more b)

  'abbaabbba'
  'a'
  ...'a'
  ....'a'
  ........'a'

'ab+?' (a followed by one or more b)

  'abbaabbba'
  'ab'
  ....'ab'

'ab??' (a followed by zero or one b)

  'abbaabbba'
  'a'
  ...'a'
  ....'a'
  ........'a'

'ab{3}?' (a followed by three b)

  'abbaabbba'
  ....'abbb'

'ab{2,3}?' (a followed by two to three b)

  'abbaabbba'
  'abb'
  ....'abb'
```

## 字符集

字符集代表一组字符，其中的任何一个字符都能在模式中的某一点得到匹配，例如，`[ab]` 会与 `a` 或者 `b` 匹配。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return


test_patterns(
    'abbaabbba',
    [('[ab]', 'either a or b'),
     ('a[ab]+', 'a followed by 1 or more a or b'),
     ('a[ab]+?', 'a followed by 1 or more a or b, not greedy')],
)
```

（`a[ab]+`）表达式的贪婪模式将得到整个字符串因为字符串首字母字符是 `a` 并且接下来的每一个字符都是 `a` 或者 `b`。

```
'[ab]' (either a or b)

  'abbaabbba'
  'a'
  .'b'
  ..'b'
  ...'a'
  ....'a'
  .....'b'
  ......'b'
  .......'b'
  ........'a'

'a[ab]+' (a followed by 1 or more a or b)

  'abbaabbba'
  'abbaabbba'

'a[ab]+?' (a followed by 1 or more a or b, not greedy)

  'abbaabbba'
  'ab'
  ...'aa'
```

还能用字符集来排除特定的字符。脱字符（`^`）意味着要寻找不存在于跟随着脱字符的字符集中的字符。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    'This is some text -- with punctuation.',
    [('[^-. ]+', 'sequences without -, ., or space')],
)
```

这一模式会寻找所有不包含字符 `-`、`.` 和空格的子字符串。

```
'[^-. ]+' (sequences without -, ., or space)

  'This is some text -- with punctuation.'
  'This'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'
```

随着字符集规模的增大，打出每一个需要包含（或排除）的字符会变得枯燥乏味。使用字符区间是一种更加紧凑的方式，它可以通过来包含所有在指定的开始和结束点之间的所有连续的字符来指定一个字符集。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    'This is some text -- with punctuation.',
    [('[a-z]+', 'sequences of lowercase letters'),
     ('[A-Z]+', 'sequences of uppercase letters'),
     ('[a-zA-Z]+', 'sequences of letters of either case'),
     ('[A-Z][a-z]+', 'one uppercase followed by lowercase')],
)
```

在此处区间 `a-z`包含了小写 ASCII 字符，而区间 `A-Z` 包含了大写的 ASCII 字母。多个区间也可以被组合并成为一个单独的字符集。

```
'[a-z]+' (sequences of lowercase letters)

  'This is some text -- with punctuation.'
  .'his'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'[A-Z]+' (sequences of uppercase letters)

  'This is some text -- with punctuation.'
  'T'

'[a-zA-Z]+' (sequences of letters of either case)

  'This is some text -- with punctuation.'
  'This'
  .....'is'
  ........'some'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'[A-Z][a-z]+' (one uppercase followed by lowercase)

  'This is some text -- with punctuation.'
  'This'
```

作为字符集中的特殊案例，元字符中的点号（`.`）表示这一模式应当与在此处的任一单一字符相匹配。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    'abbaabbba',
    [('a.', 'a followed by any one character'),
     ('b.', 'b followed by any one character'),
     ('a.*b', 'a followed by anything, ending in b'),
     ('a.*?b', 'a followed by anything, ending in b')],
)
```

如果没有使用非贪婪模式，将点号与重复结合会导致较长的匹配结果。

```
'a.' (a followed by any one character)

  'abbaabbba'
  'ab'
  ...'aa'

'b.' (b followed by any one character)

  'abbaabbba'
  .'bb'
  .....'bb'
  .......'ba'

'a.*b' (a followed by anything, ending in b)

  'abbaabbba'
  'abbaabbb'

'a.*?b' (a followed by anything, ending in b)

  'abbaabbba'
  'ab'
  ...'aab'
```

## 转义码

一个更加紧凑的表示方法是为若干预先定义的字符集使用转义吗。下表列出了被 `re` 所识别的转义码。

| Code | Meaning                          |
| ---- | -------------------------------- |
| `\d` | 一个数字                         |
| `\D` | 一个非数字                       |
| `\s` | 空白符（制表符，空格，换行符等） |
| `\S` | 非空白字符                       |
| `\w` | 由字母和数字组成的               |
| `\W` | 非字母和数字组成的               |

**注意**

转义符通过在字符前增加反斜杠 （`\`）来表示。但是通常要想在 Python 字符串中表示反斜杠，则反斜杠自身必须转义，进而导致了难以理解的表达式。通过在字面值前增加前缀 `r` 来使用原始字符串可以解决这个问题并保持可读性。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    'A prime #1 example!',
    [(r'\d+', 'sequence of digits'),
     (r'\D+', 'sequence of non-digits'),
     (r'\s+', 'sequence of whitespace'),
     (r'\S+', 'sequence of non-whitespace'),
     (r'\w+', 'alphanumeric characters'),
     (r'\W+', 'non-alphanumeric')],
)
```

这些示例中的表达式将转义符与重复相结合以在输入字符串中寻找相似的字符序列。

```
'\d+' (sequence of digits)

  'A prime #1 example!'
  .........'1'

'\D+' (sequence of non-digits)

  'A prime #1 example!'
  'A prime #'
  ..........' example!'

'\s+' (sequence of whitespace)

  'A prime #1 example!'
  .' '
  .......' '
  ..........' '

'\S+' (sequence of non-whitespace)

  'A prime #1 example!'
  'A'
  ..'prime'
  ........'#1'
  ...........'example!'

'\w+' (alphanumeric characters)

  'A prime #1 example!'
  'A'
  ..'prime'
  .........'1'
  ...........'example'

'\W+' (non-alphanumeric)

  'A prime #1 example!'
  .' '
  .......' #'
  ..........' '
  ..................'!'
```

要想匹配正则表达式语法中的元字符，可以在搜索模式中对这些字符进行转义。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    r'\d+ \D+ \s+',
    [(r'\\.\+', 'escape code')],
)
```

示例中的模式将反斜杠和加号进行了转义，因为它们都是元字符并且在正则表达式中具有特殊的含义。

```
'\\.\+' (escape code)

  '\d+ \D+ \s+'
  '\d+'
  .....'\D+'
  ..........'\s+'

```

## 锚定

除了通过描述模式包含的内容来进行匹配，还可以通过在输入文本中使用锚定指令指定模式应当出现的相对位置，下表列出了可用的锚定代码。

| Code | Meaning                          |
| ---- | -------------------------------- |
| `^`  | 一个字符串或一行的开头           |
| `$`  | 一个字符串或一行的结尾           |
| `\A` | 字符串的开通                     |
| `\Z` | 字符串的结尾                     |
| `\b` | 在一个单词开通或结尾的空字符串   |
| `\B` | 不在一个单词开头或结尾的空字符串 |

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return

test_patterns(
    'This is some text -- with punctuation.',
    [(r'^\w+', 'word at start of string'),
     (r'\A\w+', 'word at start of string'),
     (r'\w+\S*$', 'word near end of string'),
     (r'\w+\S*\Z', 'word near end of string'),
     (r'\w*t\w*', 'word containing t'),
     (r'\bt\w+', 't at start of word'),
     (r'\w+t\b', 't at end of word'),
     (r'\Bt\B', 't, not start or end of word')],
)
```

在示例中，用来在字符串开通和在字符串结尾来匹配单词所用的模式之间有所不同，因为在字符串结尾常有标点符号跟随单词以结束句子。所以使用模式 ` \w+$ ` 将不会成功匹配，因为 `.` 不是一个字母或者数字字符。

```
'^\w+' (word at start of string)

  'This is some text -- with punctuation.'
  'This'

'\A\w+' (word at start of string)

  'This is some text -- with punctuation.'
  'This'

'\w+\S*$' (word near end of string)

  'This is some text -- with punctuation.'
  ..........................'punctuation.'

'\w+\S*\Z' (word near end of string)

  'This is some text -- with punctuation.'
  ..........................'punctuation.'

'\w*t\w*' (word containing t)

  'This is some text -- with punctuation.'
  .............'text'
  .....................'with'
  ..........................'punctuation'

'\bt\w+' (t at start of word)

  'This is some text -- with punctuation.'
  .............'text'

'\w+t\b' (t at end of word)

  'This is some text -- with punctuation.'
  .............'text'

'\Bt\B' (t, not start or end of word)

  'This is some text -- with punctuation.'
  .......................'t'
  ..............................'t'
  .................................'t'
```

## 限制搜索

当提前知道只需要对完整的输入中的一个子集进行搜索的情况下，可以通过给 `re` 提供限制搜索区间的方式来对正则表达式匹配进行进一步的限制。例如，如果要求模式必须在输入的最开始出现，则可以使用 `match()` 而不是 `search()` ，这会将匹配锚定在输入的开头，而不需要明确地在搜索模式中包含锚点。

```python
import re

text = 'This is some text -- with punctuation.'
pattern = 'is'

print('Text   :', text)
print('Pattern:', pattern)

m = re.match(pattern, text)
print('Match  :', m)
s = re.search(pattern, text)
print('Search :', s)
```

由于字面值文本 `is` 没有在输入文本的最开头出现，使用 `match()` 进行搜索将找不到它，尽管如此，这一序列在文本中的其他位置还出现了一次，因此 `search()` 找到了它。

```
Text   : This is some text -- with punctuation.
Pattern: is
Match  : None
Search : <re.Match object; span=(2, 4), match='is'>
```

`fullmatch()` 方法要求输入的整个字符串都与模式匹配。

```python
text = 'This is some text -- with punctuation.'
pattern = 'is'

print('Text       :', text)
print('Pattern    :', pattern)

m = re.search(pattern, text)
print('Search     :', m)
s = re.fullmatch(pattern, text)
print('Full match :', s)
```

在此处 `search()` 的结果表明模式确实在输入中出现了，但是由于其没有匹配全部的输入，所以 `fullmatch()` 没有报告匹配项。

```
Text       : This is some text -- with punctuation.
Pattern    : is
Search     : <re.Match object; span=(2, 4), match='is'>
Full match : None
```

一个被编译过的正则表达式的 `search()` 方法接受可选的 `start` 和 `end`  位置参数以限制对输入中的子串进行搜索。

```python
import re

text = 'This is some text -- with punctuation.'
pattern = re.compile(r'\b\w*is\w*\b')

print('Text:', text)
print()

pos = 0
while True:
    match = pattern.search(text, pos)
    if not match:
        break
    s = match.start()
    e = match.end()
    print('  {:>2d} : {:>2d} = "{}"'.format(s, e - 1, text[s:e]))
    pos = e
```

示例实现了一个不如 `iterall()` 高效的方式。每当找到一个匹配，该匹配的结尾位置将被作用下一个搜索的开始位置。

```
Text: This is some text -- with punctuation.

   0 :  3 = "This"
   5 :  6 = "is"
```

## 组的匹配

搜索模式匹配是正则表达式提供强大功能的基础。在模式里可以隔离匹配的部分，这个功能扩展可以用来创建解析器。组通过括号中的模式来定义。

```python
import re

def test_patterns(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            substr = text[s:e]
            n_backslashes = text[:s].count('\\')
            prefix = '.' * (s + n_backslashes)
            print("  {}'{}'".format(prefix, substr))
        print()
    return


test_patterns(
    'abbaaabbbbaaaaa',
    [('a(ab)', 'a followed by literal ab'),
     ('a(a*b*)', 'a followed by 0-n a and 0-n b'),
     ('a(ab)*', 'a followed by 0-n ab'),
     ('a(ab)+', 'a followed by 1-n ab')],
)
```

任何完整的正则表达式都可以转换为组，也可以嵌套在较大的表达式中。所有的重复修饰符都可以应用于一个组，使得整个组模式重复。

```
'a(ab)' (a followed by literal ab)

  'abbaaabbbbaaaaa'
  ....'aab'

'a(a*b*)' (a followed by 0-n a and 0-n b)

  'abbaaabbbbaaaaa'
  'abb'
  ...'aaabbbb'
  ..........'aaaaa'

'a(ab)*' (a followed by 0-n ab)

  'abbaaabbbbaaaaa'
  'a'
  ...'a'
  ....'aab'
  ..........'a'
  ...........'a'
  ............'a'
  .............'a'
  ..............'a'

'a(ab)+' (a followed by 1-n ab)

  'abbaaabbbbaaaaa'
  ....'aab'
```

`Match` 对象的 `groups()` 方法可以用来访问模式中各个组匹配的子字符串。

```python
import re

text = 'This is some text -- with punctuation.'
print(text)
print()

patterns = [
    (r'^(\w+)', 'word at start of string'),
    (r'(\w+)\S*$', 'word at end, with optional punctuation'),
    (r'(\bt\w+)\W+(\w+)', 'word starting with t, another word'),
    (r'(\w+t)\b', 'word ending with t'),
]

for pattern, desc in patterns:
    regex = re.compile(pattern)
    match = regex.search(text)
    print("'{}' ({})\n".format(pattern, desc))
    print('  ', match.groups())
    print()
```

`Match.groups()` 按照匹配的表达式中组的顺序返回字符串序列。

```
This is some text -- with punctuation.

'^(\w+)' (word at start of string)

   ('This',)

'(\w+)\S*$' (word at end, with optional punctuation)

   ('punctuation',)

'(\bt\w+)\W+(\w+)' (word starting with t, another word)

   ('text', 'with')

'(\w+t)\b' (word ending with t)

   ('text',)
```

`group()` 方法可以请求单个组的匹配。这在分组查找字符串，但在结果中不需要组匹配的部分的时候非常有用。

```python
import re

text = 'This is some text -- with punctuation.'
print('Input text            :', text)

regex = re.compile(r'(\bt\w+)\W+(\w+)')
print('Pattern               :', regex.pattern)

match = regex.search(text)
print('Entire match          :', match.group(0))
print('Word starting with "t":', match.group(1))
print('Word after "t" word   :', match.group(2))
```

组 `0` 表示整个表达式匹配的字符串，子组以 `1` 开始，以它的左括号在表达式中出现的顺序编号。

```
Input text            : This is some text -- with punctuation.
Pattern               : (\bt\w+)\W+(\w+)
Entire match          : text -- with
Word starting with "t": text
Word after "t" word   : with
```

Python 扩展了基本的分组语法，添加了命名组。使用名称引用组可以使模式更容易修改，而不必使用匹配结果来修改代码。设置组名称的语法是 `(?P<name>pattern)` 。

```python
import re

text = 'This is some text -- with punctuation.'
print(text)
print()

patterns = [
    r'^(?P<first_word>\w+)',
    r'(?P<last_word>\w+)\S*$',
    r'(?P<t_word>\bt\w+)\W+(?P<other_word>\w+)',
    r'(?P<ends_with_t>\w+t)\b',
]

for pattern in patterns:
    regex = re.compile(pattern)
    match = regex.search(text)
    print("'{}'".format(pattern))
    print('  ', match.groups())
    print('  ', match.groupdict())
    print()
```

`groupdict()` 可以用来获得映射匹配组名到子字符串的字典。命名模式也包含在 `groups()` 返回的有序序列中。

```
This is some text -- with punctuation.

'^(?P<first_word>\w+)'
   ('This',)
   {'first_word': 'This'}

'(?P<last_word>\w+)\S*$'
   ('punctuation',)
   {'last_word': 'punctuation'}

'(?P<t_word>\bt\w+)\W+(?P<other_word>\w+)'
   ('text', 'with')
   {'t_word': 'text', 'other_word': 'with'}

'(?P<ends_with_t>\w+t)\b'
   ('text',)
   {'ends_with_t': 'text'}
```

更新 `test_patterns` 函数，更新包括模式匹配的编号和命名组。由于组本身是一个完整的正则表达式，所以可以将组嵌套在其他组中，以构建更复杂的表达式。

```python
import re

def test_patterns_new(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            prefix = ' ' * (s)
            print('  {}{!r}{}'.format(prefix, text[s:e], ' ' * (len(text) - e)), end=' ')
            print(match.groups())
            if match.groupdict():
                print('{}{}'.format(' ' * (len(text) - s), match.groupdict()))
        print()
    return

test_patterns_new('abbaabbba', [(r'a((a*)(b*))', 'a followed by 0-n a and 0-n b')])
```

这个例子中，组 `(a*)` 匹配一个空字符串，因此 `groups()` 的返回结果包含该空字符串。

```
'a((a*)(b*))' (a followed by 0-n a and 0-n b)

  'abbaabbba'
  'abb'       ('bb', '', 'bb')
     'aabbb'  ('abbb', 'a', 'bbb')
          'a' ('', '', '')
```

组对于指定选择模式也很有用，使用管道符号 （`|`） 将两个模式分开，指示任何一个模式都可以匹配，但需要仔细考虑管道的位置。本例中的第一个表达式与 `a` 序列匹配，后面是完全由单个字母 `a` 或 `b` 组成的序列。第二个模式匹配 `a` ，后面可能包含 `a` 或 `b` 的序列。模式是相似的，但匹配结果完全不同。

```python
import re

def test_patterns_new(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            prefix = ' ' * (s)
            print('  {}{!r}{}'.format(prefix, text[s:e], ' ' * (len(text) - e)), end=' ')
            print(match.groups())
            if match.groupdict():
                print('{}{}'.format(' ' * (len(text) - s), match.groupdict()))
        print()
    return

test_patterns_new(
    'abbaabbba',
    [(r'a((a+)|(b+))', 'a then seq. of a or seq. of b'),
     (r'a((a|b)+)', 'a then seq. of [ab]')],
)
```

当一个选择组不匹配而整个模式匹配时，`groups()` 的返回结果会在选择组应该出现的顺序中处包含一个 `None` 值。

```
'a((a+)|(b+))' (a then seq. of a or seq. of b)

  'abbaabbba'
  'abb'       ('bb', None, 'bb')
     'aa'     ('a', 'a', None)

'a((a|b)+)' (a then seq. of [ab])

  'abbaabbba'
  'abbaabbba' ('bbaabbba', 'a')
```

如果匹配子模式的字符串不是从全文中提取的字符串的一部分，那么定义包含子模式的组合会很有用，这些类型的组被称为非捕获。非捕获组可用于描述重复模式或选择方案，而无需在返回的值中隔离字符串的匹配部分。创建非捕获组的语法是 `(?:pattern)`。

```python
import re

def test_patterns_new(text, patterns):
    """Given source text and a list of patterns, look for
    matches for each pattern within the text and print
    them to stdout.
    """
    # Look for each pattern in the text and print the results
    for pattern, desc in patterns:
        print("'{}' ({})\n".format(pattern, desc))
        print("  '{}'".format(text))
        for match in re.finditer(pattern, text):
            s = match.start()
            e = match.end()
            prefix = ' ' * (s)
            print('  {}{!r}{}'.format(prefix, text[s:e], ' ' * (len(text) - e)), end=' ')
            print(match.groups())
            if match.groupdict():
                print('{}{}'.format(' ' * (len(text) - s), match.groupdict()))
        print()
    return

test_patterns(
    'abbaabbba',
    [(r'a((a+)|(b+))', 'capturing form'),
     (r'a((?:a+)|(?:b+))', 'noncapturing')],
)
```

在下面示例中，请比较匹配结果相同的捕获和非捕获返回组的区别。

```
'a((a+)|(b+))' (capturing form)

  'abbaabbba'
  'abb'       ('bb', None, 'bb')
     'aa'     ('a', 'a', None)

'a((?:a+)|(?:b+))' (noncapturing)

  'abbaabbba'
  'abb'       ('bb',)
     'aa'     ('a',)
```

## 搜索选项

选项标志用于改变匹配引擎处理表达式的方式。这些标志可以使用按位或运算进行组合，然后传递给 `compile()`，`search()`，`match()` 以及其它接受搜索模式的函数。

### 不区分大小写的匹配

 `IGNORECASE ` 会使该模式下的文字字符和字符范围既能匹配大写字符又能匹配小写字符。

```python
import re

text = 'This is some text -- with punctuation.'
pattern = r'\bT\w+'
with_case = re.compile(pattern)
without_case = re.compile(pattern, re.IGNORECASE)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('Case-sensitive:')
for match in with_case.findall(text):
    print('  {!r}'.format(match))
print('Case-insensitive:')
for match in without_case.findall(text):
    print('  {!r}'.format(match))
```

由于该模式包含大写字母 `T` ，如果没有设置 `IGNORECASE` ，那么唯一的匹配项是单词 `This` 。如果设置了 `IGNORECASE` ，单词 `text` 也匹配。

```
Text:
  'This is some text -- with punctuation.'
Pattern:
  \bT\w+
Case-sensitive:
  'This'
Case-insensitive:
  'This'
  'text'
```

### 多行输入

在多行输入的情况下，有两个标志会影响搜索的方式，分别是：` MULTILINE ` 和 `DOTALL`。` MULTILINE ` 标志控制模式匹配代码如何处理包含换行符文本的锚定指令。当多行模式打开时，除了整个字符串以外， `^`和 `$` 的锚定规则还适用于每一行的开头和结尾。

```python
import re

text = 'This is some text -- with punctuation.\nA second line.'
pattern = r'(^\w+)|(\w+\S*$)'
single_line = re.compile(pattern)
multiline = re.compile(pattern, re.MULTILINE)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('Single Line :')
for match in single_line.findall(text):
    print('  {!r}'.format(match))
print('Multline    :')
for match in multiline.findall(text):
    print('  {!r}'.format(match))

```

示例中的模式与输入的第一个或最后一个单词匹配。它匹配字符串末尾的 `line.`，即使此处并没有换行符。

```
Text:
  'This is some text -- with punctuation.\nA second line.'
Pattern:
  (^\w+)|(\w+\S*$)
Single Line :
  ('This', '')
  ('', 'line.')
Multline    :
  ('This', '')
  ('', 'punctuation.')
  ('A', '')
  ('', 'line.')
```

另一个与多行文本相关的标志就是 `DOTALL`。通常来讲，点字符（`.`）可以匹配输入文本中除了换行符以外其它的所有内容，而标志 `DOTALL` 则允许点字符（`.`）也能匹配换行符。

```python
import re

text = 'This is some text -- with punctuation.\nA second line.'
pattern = r'.+'
no_newlines = re.compile(pattern)
dotall = re.compile(pattern, re.DOTALL)

print('Text:\n  {!r}'.format(text))
print('Pattern:\n  {}'.format(pattern))
print('No newlines :')
for match in no_newlines.findall(text):
    print('  {!r}'.format(match))
print('Dotall      :')
for match in dotall.findall(text):
    print('  {!r}'.format(match))
```

如果没有标志，输入文本只能逐行与模式相匹配。添加标志后会使整个字符串与模式相匹配。

```
Text:
  'This is some text -- with punctuation.\nA second line.'
Pattern:
  .+
No newlines :
  'This is some text -- with punctuation.'
  'A second line.'
Dotall      :
  'This is some text -- with punctuation.\nA second line.'
```

### Unicode

在 Python 3 中，`str` 对象使用完整的 Unicode 字符集，并且在 `str` 上的正则表达式处理有这样一个假设：模式和输入文本均为 Unicode。前面所说的转义码在默认情况下是按照 Unicode 定义的。这些假设就意味着模式 `\w+` 会匹配 “French” 和 “ Français ” 两个单词。为了将转义字符限制为 ASCII 字符集，我们可以在编译模式或者在调用模块级函数 `search()` 和 `match()` 时使用 ASCII 标志。

```python
import re

text = u'Français złoty Österreich'
pattern = r'\w+'
ascii_pattern = re.compile(pattern, re.ASCII)
unicode_pattern = re.compile(pattern)

print('Text    :', text)
print('Pattern :', pattern)
print('ASCII   :', list(ascii_pattern.findall(text)))
print('Unicode :', list(unicode_pattern.findall(text)))
```

对于 ASCII 文本，其它的转义字符序列（`\W`，`\b`，`\B`，`\d`，`\D`，`\s`，`\S`）的处理方式也不尽相同。`re` 并不是通过查询 Unicode 数据库的方式去查找每个字符的属性，而是使用字符集的 ASCII 定义。

```
Text    : Français złoty Österreich
Pattern : \w+
ASCII   : ['Fran', 'ais', 'z', 'oty', 'sterreich']
Unicode : ['Français', 'złoty', 'Österreich']
```

### 详细的表达式语法

随着表达式变得越来越复杂，正则表达式的紧凑格式可能会成为障碍。随着表达式中组的数目增加，将会有更多的工作来跟踪为什么需要每个元素，以及表达式的各个部分以何种方式相互作用，使用命名组有助于缓解这些问题，但更好的解决方案时使用  *verbose mode*  表达式，它允许在模式中插入注释和多余的空格。

按照电子邮箱地址的模式将阐明详细模式（ *verbose mode* ）如何使得正则表达式更加易用。第一个版本识别的是这样一个地址，它以三个顶级域名中的一个域名作为结尾，这三个顶级域名分别为：`.com`、`.org` 或者`.edu`。

```python
import re

address = re.compile('[\w\d.+-]+@([\w\d.]+\.)+(com|org|edu)')

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
]

for candidate in candidates:
    match = address.search(candidate)
    print('{:<30}  {}'.format(candidate, 'Matches' if match else 'No match'))
```

该表达式包括了几个字符类、组以及重复表达式。

```
first.last@example.com          Matches
first.last+category@gmail.com   Matches
valid-address@mail.example.com  Matches
not-valid@example.foo           No match
```

将表达式转换为更详细的格式可以使表达式更易于扩展。

```python
import re

address = re.compile(
    '''
    [\w\d.+-]+     # 用户名
    @
    ([\w\d.]+\.)+  # 域名前缀
    (com|org|edu)
    ''',
    re.VERBOSE)

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
]

for candidate in candidates:
    match = address.search(candidate)
    print('{:<30}  {}'.format(candidate, 'Matches' if match else 'No match'))
```

表达式匹配的是相同的输入，但是在这种扩展格式中，它具有更好的可读性。注释还有助于识别模式的不同部分，以便扩展它以匹配更多的输入。

```
first.last@example.com          Matches
first.last+category@gmail.com   Matches
valid-address@mail.example.com  Matches
not-valid@example.foo           No match
```

这个扩展版本可以解析包含个人姓名和电子邮件地址的输入，正如电子邮件标题所示，独立存在，电子邮件地址紧随其后，两边有尖括号（`<` 和 `>`）。

```python
import re

address = re.compile(
    '''
    # 名字是由字母组成的，包含 0 个或多个 "." 表示标题缩写和中间首字母
    ((?P<name>
      ([\w.,]+\s+)*[\w.,]+)
      \s*
      # 电子邮件地址被封装在尖括号<>中，但是只有在找到一个名字的情况下，因此将起始括号保留在此组中。
      <
    )? # 整个名字是可选的
    
    # 地址本身：username@domain.tld
    (?P<email>
      [\w\d.+-]+     # 用户名
      @
      ([\w\d.]+\.)+  # 域名前缀
      (com|org|edu)
    )
    >? # 可选的闭合尖括号
    ''',
    re.VERBOSE
)

candidates = [
    u'first.last@example.com',
    u'first.last+category@gmail.com',
    u'valid-address@mail.example.com',
    u'not-valid@example.foo',
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'First Last',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
    u'<first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Name :', match.groupdict()['name'])
        print('  Email:', match.groupdict()['email'])
    else:
        print('  No match')
```

与其他编程语言一样，将注释输入插入详细正则表达式的能力有助于提高它们的可维护性。这个最终版本包括对未来维护人员的维护注意事项和空格。这些空格能够将各组彼此分开，并突出显示他们的嵌套级别。

```
Candidate: first.last@example.com
  Name : None
  Email: first.last@example.com
Candidate: first.last+category@gmail.com
  Name : None
  Email: first.last+category@gmail.com
Candidate: valid-address@mail.example.com
  Name : None
  Email: valid-address@mail.example.com
Candidate: not-valid@example.foo
  No match
Candidate: First Last <first.last@example.com>
  Name : First Last
  Email: first.last@example.com
Candidate: No Brackets first.last@example.com
  Name : None
  Email: first.last@example.com
Candidate: First Last
  No match
Candidate: First Middle Last <first.last@example.com>
  Name : First Middle Last
  Email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Name : First M. Last
  Email: first.last@example.com
Candidate: <first.last@example.com>
  Name : None
  Email: first.last@example.com
```

## 在模式中嵌入标志

在编译表达式时不能添加标志的情况下，例如将模式作为参数传递给稍后编译的库函数时，可以将标志嵌入表达式字符串本身。例如，想要打开不区分大小写的匹配项，就要在表达式的开头加 `(?i)`。

```python
import re

text = 'This is some text -- with punctuation.'
pattern = r'(?i)\bT\w+'
regex = re.compile(pattern)

print('Text      :', text)
print('Pattern   :', pattern)
print('Matches   :', regex.findall(text))
```

因为选项控制了整个表达式的评估或解析方式，所以它们应该始终显示在表达式的开头。

```
Text      : This is some text -- with punctuation.
Pattern   : (?i)\bT\w+
Matches   : ['This', 'text']
```

下表列出了所有标志的缩写：

| Flag         | Abbreviation |
| ------------ | ------------ |
| `ASCII`      | `a`          |
| `IGNORECASE` | `i`          |
| `MULTILINE`  | `m`          |
| `DOTALL`     | `s`          |
| `VERBOSE`    | `x`          |

嵌入的标志可以通过将它们放在同一个组中来组合。例如，`(?im)` 为多行输入打开了不区分大小写的匹配。

## 前向查看还是后向查看

在很多情况下，只有当其它部分也匹配时，匹配一部分模式才有用。例如，在电子邮件解析表达式中，尖括号标记为可选。实际上，括号应该时成对的，并且表达式只有在两者都存在或者都不存在时才匹配。这个表达式的修改版本使用 positive look ahead 断言来匹配这一对。前向查看的断言语是 `(?=pattern)`

```python
import re

address = re.compile(
    '''
    # 名字是由字母组成的，包含 0 个或多个 "." 表示标题缩写和中间首字母
    ((?P<name>
        ([\w.,]+\s+)*[\w.,]+
     )
     \s+
    ) # 名字不再是可选的

    # LOOKAHEAD
    # 电子邮件地址被封装在尖括号中，但是只有在尖括号都存在或者都不存在时才如此
    (?= (<.*>$)  # 用尖括号括起来的剩余部分
        |
        ([^<].*[^>]$)  # 不用 尖括号括起来的剩余部分
      )

    <?  # 可选的左尖括号
    # 地址本身：username@domain.tld
    (?P<email>
      [\w\d.+-]+  # 用户名
      @
      ([\w\d.]+\.)+  # 域名前缀
      (com|org|edu)
    )

    >? # 可选的右尖括号
    ''',
    re.VERBOSE)

candidates = [
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'Open Bracket <first.last@example.com',
    u'Close Bracket first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Name :', match.groupdict()['name'])
        print('  Email:', match.groupdict()['email'])
    else:
        print('  No match')
```

在这个表达式中有几个重要的变化。首先，名字部分不再是可选的。这意味着独立地址不匹配，但也防止匹配格式不正确的名字/地址组合。在 <name> 组之后的正向前查看规则 （ positive look ahead rule ）断言，要么字符串的剩余部分用一对尖括号括起来，要么没有不匹配的括号；要么两个括号都存在，要么两个括号都不存在。前向查看被表示为一个组，但是前向查看组的匹配不会消耗任何输入文本，所以在前向查看匹配之后，模式的其余部分将从统一点采集。

```
Candidate: First Last <first.last@example.com>
  Name : First Last
  Email: first.last@example.com
Candidate: No Brackets first.last@example.com
  Name : No Brackets
  Email: first.last@example.com
Candidate: Open Bracket <first.last@example.com
  No match
Candidate: Close Bracket first.last@example.com>
  No match
```

 *negative look ahead*  断言（`(?!pattern)`）表示模式与当前点后面的文本不匹配。例如，可以修改电子邮件识别模式，以忽略自动化系统常用的 `noreplay` 邮件地址。

```python
import re

address = re.compile(
    '''
    ^
    
    # 忽略 noreply 地址
    (?!noreply@.*$)
    
    [\w\d.+-]+
    @
    ([\w\d.]+\.)+
    (com|org|edy)
    ''',
    re.VERBOSE
)

candidates = [
    u'first.last@example.com',
    u'noreply@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match:', candidate[match.start():match.end()])
    else:
        print('  No match')
```

以 `noreply` 开头的地址模式不匹配，因为前向查看断言失败。

```
Candidate: first.last@example.com
  Match: first.last@example.com
Candidate: noreply@example.com
  No match
```

与其在电子邮件地址的用户名部分中前向查看 `noreply` ，在使用语法 `(?<!pattern)` 匹配用户名后，可以使用  *negative look behind*  断言编写模式。

```python
import re

address = re.compile(
    '''
    ^
    
    [\w\d.+-]+    # 用户名
    
    # 忽略 noreply 地址
    (?<!noreply)
    
    @
    ([\w\d.]+\.)+
    (com|org|edy)
    
    $
    ''',
    re.VERBOSE
)

candidates = [
    u'first.last@example.com',
    u'noreply@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match:', candidate[match.start():match.end()])
    else:
        print('  No match')
```

前向查看的工作方式与后向查看的工作方式略有不同，因为表达式必须使用固定长度的模式。只要它们的数量是固定的（没有通配符或者范围），允许重复。

```
Candidate: first.last@example.com
  Match: first.last@example.com
Candidate: noreply@example.com
  No match
```

 *positive look behind*  断言可以使用语法 `(?<=pattern)`，遵循一种模式查找文本下的文本。在下面的例子中，该表达式查找 Twitter 句柄。

```python
import re

twitter = re.compile(
    '''
    (?<=@)
    ([\w\d_]+)
    ''',
    re.VERBOSE
)

text = '''This text includes two Twitter handles.
One for @ThePSF, and one for the author, @doughellmann.
'''

print(text)
for match in twitter.findall(text):
    print('Handle:', match)
```

该模式与这样的字符序列匹配，它能够组成 Twitter 句柄，只要字符序列的开头有一个 `@` 即可。

```
This text includes two Twitter handles.
One for @ThePSF, and one for the author, @doughellmann.

Handle: ThePSF
Handle: doughellmann
```

## 自引用表达式

匹配值可以用于表达式的后面部分。例如，通过包含对这些组的反向引用，可以更新电子邮件示例，使其只匹配由此人的姓和名组成的地址。实现这一点最简单的方法是使用 `\num` ，通过 ID 号引用前面匹配的组。

```python
address = re.compile(
    r'''
    # 正则名称
    (\w+)           # 名
    \s+
    (([\w.]+)\s+)?  # 可选的中间名或者首字母
    (\w+)           # 姓
    
    \s+
    
    <
    
    (?P<email>
      \1            # 名
      \.
      \4            # 姓
      @
      ([\w\d.]+\.)+ # 域名前缀
      (com|org|edu) # 限制允许的顶级域
    )
    
    >
    ''',
    re.VERBOSE | re.IGNORECASE)

candidates = [
    u'First Last <first.last@example.com>',
    u'Different Name <first.last@example.com>',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match name :', match.group(1), match.group(4))
        print('  Match email:', match.group(5))
    else:
        print('  No match')
```

虽然语法很简单，但是通过数字 ID 创建反向引用有一些缺点。从实践的角度来看，随着表达式的改变，这些组必须重新计算，每一个引用都需要更新。另一个缺点是，使用标准后向引用语法 `\n` 只能创建 99 个引用，因为如果 ID 号是三位数，它会被理解为一个八进制字符值而不是一个组引用。当然，如果一个表达式中有超过 99 个组，将会遇到更为严重的维护挑战，而不仅仅是不能引用它们。

```
Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: Different Name <first.last@example.com>
  No match
Candidate: First Middle Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
```

Python 的表达式解析器包含一个扩展，该扩展使用 `(?P=name)` 来引用表达式中前面匹配的已命名组的值。

```python
import re

address = re.compile(
    '''
    (?P<first_name>\w+)
    \s+
    (([\w.]+)\s+)?     # 可选的中间名或首字母
    (?P<last_name>\w+)
    
    \s+
    
    <
    
    (?P<email>
      (?P=first_name)
      \.
      (?P=last_name)
      @
      ([\w\d.]+\.)+
      (com|org|edu)
    )
    >
    ''',
    re.VERBOSE | re.IGNORECASE)

candidates = [
    u'First Last <first.last@example.com>',
    u'Different Name <first.last@example.com>',
    u'First Middle Last <first.last@example.com>',
    u'First M. Last <first.last@example.com>',
]

for cadidate in candidates:
    print('Candidate:', cadidate)
    match = address.search(cadidate)
    if match:
        print('  Match name :', match.groupdict()['first_name'], end=' ')
        print(match.groupdict()['last_name'])
        print('  Match email:', match.groupdict()['email'])
    else:
        print('  No match')
```

地址表达式是在 ` IGNORECASE ` 标志打开的情况下编译的，因为专有名称通常是大写字母，而电子邮件地址不是。

```
Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: Different Name <first.last@example.com>
  No match
Candidate: First Middle Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: First M. Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
```

在表达式中使用反向引用的另一种机制根据前一个组是否匹配来选择不同的模式。电子邮件模式可以纠正，这样在名称出现时就需要尖括号，如果时电子邮件本身则不需要尖括号。检测一个组是否匹配的语法是 `(?(id)yes-expression|no-expression)`，其中 `id` 是组名或者组号，`yes-expression` 是组有一个值得情况下要使用的模式，`no-expression` 是其它情况下要使用的模式。

```python
address = re.compile(
    '''
    ^
    
    # 名字是由字母组成的，包含 0 个或多个 "." 表示标题缩写和中间首字母
    (?P<name>
       ([\w.]+\s+)*[\w.]+
    )?
    \s*
    
    # 电子邮件地址被封装在尖括号中，但是只有在找到一个名字时如此。
    (?(name)
    # 剩余部分被封装在尖括号中因为此处有一个名字
      (?P<brackets>(?=(<.*>$)))
      |
    # 没有名字的剩余部分不含有尖括号
      (?=([^<].*[^>]$))
    )
    
    # 只有在前向查看断言找到成对的括号时，才算找到一个括号。
    (?(brackets)<|\s*)
    
    (?P<email>
      [\w\d.+-]+         # 用户名
      @
      ([\w\d.]+\.)+      # 域名前缀
      (com|org|edu)
    )
    
    # 只有在前向查看断言找到成对括号时，才算找到一个括号。
    (?(brackets)>|\s*)
    $
    ''',
    re.VERBOSE)

candidates = [
    u'First Last <first.last@example.com>',
    u'No Brackets first.last@example.com',
    u'Open Bracket <first.last@example.com',
    u'Close Bracket first.last@example.com>',
    u'no.brackets@example.com',
]

for candidate in candidates:
    print('Candidate:', candidate)
    match = address.search(candidate)
    if match:
        print('  Match name :', match.groupdict()['name'])
        print('  Match email:', match.groupdict()['email'])
    else:
        print('  No match')
```

此版本的电子邮寄地址解析器使用两个测试。如果 `name` 组匹配，前向查看断言需要一对尖括号并设置 `brackets` 组。如果 `name` 组不匹配，断言需要剩余的文本周围不存在尖括号。之后，如果设置了 `brackests`组，实际模式匹配代码将使用文本模式消耗输入的括号。否则它会消耗任何空格。

```
Candidate: First Last <first.last@example.com>
  Match name : First Last
  Match email: first.last@example.com
Candidate: No Brackets first.last@example.com
  No match
Candidate: Open Bracket <first.last@example.com
  No match
Candidate: Close Bracket first.last@example.com>
  No match
Candidate: no.brackets@example.com
  Match name : None
  Match email: no.brackets@example.com
```

## 用模式修改字符串

除了通过文本进行搜索之外，`re` 还支持使用正则表达式作为搜索机制修改文本，并且替换可以引用在模式中匹配的组，并将其作为替换文本的一部分。使用 `sub()` 函数来用另一个字符串替换所有出现的模式。

```python
import re

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\1</b>', text))
```

使用用于反向引用的 `\num` 语法，我们可以插入与模式相匹配文本的引用。

```
Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This <b>too</b>.
```

要在替换中使用命名组，请使用语法 `\g<name>` 。

```python
import re

bold = re.compile(r'\*{2}(?P<bold_text>.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\g<bold_text></b>', text))
```

`\g<name>` 语法也适用于编号引用，使用它们可以消除组号和周围文字数字之间的歧义。

```
Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This <b>too</b>.
```

将值传递给 `count` 以限制执行的替换次数。

```python
import re

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.sub(r'<b>\1</b>', text, count=1))
```

因为 `count` 是 `1` ，所以只进行第一次替换。

```
Text: Make this **bold**.  This **too**.
Bold: Make this <b>bold</b>.  This **too**.
```

`subn()` 的工作方式与 `sub()` 类是，只是它同时会返回修改后的字符串以及所作的替换次数。

```python
import 

bold = re.compile(r'\*{2}(.*?)\*{2}')

text = 'Make this **bold**.  This **too**.'

print('Text:', text)
print('Bold:', bold.subn(r'<b>\1</b>', text))
```

搜索模式在示例中匹配两次。

## 用模式分割

`str.split()` 是拆分、解析字符串最常用的方法之一。不过它只支持将文本值作为分隔符。而且，如果输入的格式不一致，有时候还需要一个正则表达式。例如许多纯文本标记语言将段落分隔符定义为两个或者更多的换行符（`\n`），在这种情况下，`str.split()` 不能使用，因为定义中有 [or more]  部分。

使用 `findall()` 来识别段落的策略将使用 `(.+?)\n{2,}` 这样的模式。

```python
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

for num, para in enumerate(re.findall(r'(.+?)\n{2,}', text, flags=re.DOTALL)):
    print(num, repr(para))
    print()
```

在输入文本末尾的段落中，这种模式就不管用了。比如。 [ Paragraph three. ] 不是输出的一部分。

```
0 'Paragraph one\non two lines.'

1 'Paragraph two.'
```

虽然扩展这个模式  一个段落以两个或者更多的换行符结尾或输入结束  可以解决问题，但会令模式更加复杂，将其转换为 `re.split()` 而不是 `re.findall()` 会自动处理边界条件，使模式更加简单。

```python
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

print('With findall:')
for num, para in enumerate(re.findall(r'(.+?)(\n{2,}|$)', text, flags=re.DOTALL)):
    print(num, repr(para))
    print()

print()
print('With split:')
for num, para in enumerate(re.split(r'\n{2,}', text)):
    print(num, repr(para))
    print()
```

`split()` 的模式参数更精确地表达了标记规范。两个或更多地换行符标志着输入字符串段落之间地分隔点。

```
With findall:
0 ('Paragraph one\non two lines.', '\n\n')

1 ('Paragraph two.', '\n\n\n')

2 ('Paragraph three.', '')


With split:
0 'Paragraph one\non two lines.'

1 'Paragraph two.'

2 'Paragraph three.'
```

将表达式扩在圆括号中，以此来定义组，这会使 `split()` 更接近 `str.partition()` ，所以它返回分隔符值以及字符串的其它部分。

```python
import re

text = '''Paragraph one
on two lines.

Paragraph two.


Paragraph three.'''

print('With split:')
for num, para in enumerate(re.split(r'(\n{2,})', text)):
    print(num, repr(para))
    print()
```

现在输出地包括每个段落，以及段落分开地换行符序列。

```
With split:
0 'Paragraph one\non two lines.'

1 '\n\n'

2 'Paragraph two.'

3 '\n\n\n'

4 'Paragraph three.'
```

