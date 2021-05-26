# string 字符串常量和模板

## 函数

`capwords()` 将字符串中所有单词的首字母大写。

```python
import string

s = 'The quick brown fox jumped over the lazy dog.'
print(s)
print(string.capwords(s))

s_new = []
for i in range(len(s.split())):
    s_new.append(s.split()[i].capitalize())

print(' '.join(s_new))
```

输出如下：

```
The quick brown fox jumped over the lazy dog.
The Quick Brown Fox Jumped Over The Lazy Dog.
The Quick Brown Fox Jumped Over The Lazy Dog.
```

## 模板

在 `string.Template` 中通过前置 $ 来识别变量（例如，$var）。另外，可以通过大括号将它们从周围的文本中分开（例如 ${var}）。

```python
import string
values = {'var': 'foo'}

t = string.Template("""
Variable        : $var
Escape          : $$
Variable in text: ${var}iable
""")

print('TEMPLATE:', t.substitute(values))

s = """
Variable        : %(var)s
Escape          : %%
Variable in text: %(var)siable
"""

print('INTERPOLATION:', s % values)

s = """
Variable        : {var}
Escape          : {{}}
Variable in text: {var}iable
"""

print('FORMAT:', s.format(**values))
```

输出如下：

```
TEMPLATE: 
Variable        : foo
Escape          : $
Variable in text: fooiable

INTERPOLATION: 
Variable        : foo
Escape          : %
Variable in text: fooiable

FORMAT: 
Variable        : foo
Escape          : {}
Variable in text: fooiable
```



上面三个关键不同点：没有考虑参数的类型。这些值被转换成字符串，然后插入到结果当中，没有可用的格式化选项。例如，无法控制用来表示浮点数的数字的个数。

如果模板需要的值没有全部作为参数提供给模板的话，可以使用 `safe_substitute()` 方法来避免发生异常。

```python
import string

values = {'var': 'foo'}

t = string.Template("$var is here but $missing is not provided")

try:
    print('substitute()    :', t.substitute(values))
except KeyError as err:
    print("ERROR:", str(err))

print('safe_substitute():', t.safe_substitute(values))
```

在values 字典中没有值提供给 missing，所以 `substitute()` 会抛出一个 `KeyError` 异常。而 `safe_substitute()` 将捕捉这个异常并将变量表达式单独留在文本中而不是抛出异常。

输出如下：

```
ERROR: 'missing'
safe_substitute(): foo is here but $missing is not provided
```

## 高级模板

` string.Template ` 缺省语法可以通过改变正则表达式模式来调整，这个正则表达式一般是用来寻找模板内容内变量名字的。语法的方法是通过改变 `delimiter` 和 `idpattern` 的类属性来做调整。

```python
import string

class MyTemplate(string.Template):
    delimiter = '%'
    idpattern = '[a-z]+_[a-z]+'


template_text = '''
  Delimiter : %%
  Replaced  : %with_underscore
  Ignored   : %notunderscored
'''

d = {
    'with_underscore': 'replaced',
    'notunderscored': 'not replaced',
}

t = MyTemplate(template_text)
print('Modified ID pattern:')
print(t.safe_substitute(d))
```

这个示例中替换规则进行了变更。分隔符用 `%` 来替代了 `$` 并且变量名字中必须包含下划线。`%notunderscored` 由于没有下划线，所以导致模式并没有被替换。结果如下：

```
Modified ID pattern:

  Delimiter : %
  Replaced  : replaced
  Ignored   : %notunderscored
```

对于更复杂的改变，可以通过覆写 `pattern` 属性和定义一个全新的正则表达式来实现。覆写的模式必须提供四个命名组来获取未识别的分隔符、命名的变量、大括号模式的变量名称、和无效的分隔符模式。

```python
import string

t = string.Template('$var')
print(t.pattern.pattern)
```

`t.pattern` 的值是编译好的正则表达式，但是原始字符串可以通过它的 `pattern` 属性来获取。

```
    \$(?:
      (?P<escaped>\$) |   # Escape sequence of two delimiters
      (?P<named>(?a:[_a-z][_a-z0-9]*))      |   # delimiter and a Python identifier
      {(?P<braced>(?a:[_a-z][_a-z0-9]*))}  |   # delimiter and a braced identifier
      (?P<invalid>)              # Other ill-formed delimiter exprs
    )
```

## 常量

`string` 模块包含了与 ASCII 、数字字符相关的一系列常量。

```py
import string
import inspect

def is_str(value):
    return isinstance(value, str)


for name, value in inspect.getmembers(string, is_str):
    if name.startswith('_'):
        continue
    print('%s=%r\n' % (name, value))
```

这些常量在处理 ASCII 数据时是非常有效的，但是现在会遇到越来越多的 Unicode 类型的非 ASCII 文本，在这个情况下这些常量使用就比较有限。

```
ascii_letters='abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'

ascii_lowercase='abcdefghijklmnopqrstuvwxyz'

ascii_uppercase='ABCDEFGHIJKLMNOPQRSTUVWXYZ'

digits='0123456789'

hexdigits='0123456789abcdefABCDEF'

octdigits='01234567'

printable='0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~ \t\n\r\x0b\x0c'

punctuation='!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~'

whitespace=' \t\n\r\x0b\x0c'
```

