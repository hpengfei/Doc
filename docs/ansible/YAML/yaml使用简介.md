## YAML

### 简介

YAML（/ˈjæməl/，尾音类似camel骆驼）是一个可读性高，用来表达数据序列化的格式。YAML参考了其他多种语言，包括：C语言、Python、Perl，并从XML、电子邮件的数据格式（RFC 2822）中获得灵感。Clark Evans在2001年首次发表了这种语言，另外Ingy döt Net与Oren Ben-Kiki也是这语言的共同设计者。当前已经有数种编程语言或脚本语言支持（或者说解析）这种语言。

### 语法规则

- 大小写敏感
- 使用缩进表示层级关系
- 缩进时不允许使用Tab键，只允许使用空格。
- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可

**支持的数据结构**

- 对象：键值对的集合，又称为映射（mapping）/ 哈希（hashes） / 字典（dictionary）
- 数组：一组按次序排列的值，又称为序列（sequence） / 列表（list）
- 纯量（scalars）：单个的、不可再分的值

### 对象

对象是一组键值对，使用冒号结构表示，冒号后面要加一个空格。

```yaml
animal: pets
```

对应 JavaScript：

```javascript
{ annimal: "pets" }
```

将所有键值对写成一个行内对象。

```yaml
hash: {name: Steve, foo: bar}
# 等同于
hash:
  name: Steve
  foo: bar
```

对应JavaScript:

```javascript
{ hash: { name: 'Steve', foo: 'bar' } }
```

### 数组

一组连词线开头的行，构成一个数组。

```yaml
- Cat
- Dog
- Goldfish
```

对应JavaScript：

```javascript
[ 'Cat', 'Dog', 'Goldfish' ]
```

数据结构的子成员是一个数组，则可以在该项下面缩进一个空格。

```yaml
-
 - Cat
 - Dog
 - Goldfish
-
 - Green
 - Red
```

对应JavaScript：

```javascript
[ [ 'Cat', 'Dog', 'Goldfish' ], [ 'Green', 'Red' ] ]
```

数组也可以采用行内表示法。

```yaml
animal: [Cat, Dog]
# 等同于
animal: 
  - Cat
  - Dog
```

对应JavaScript：

```javascript
{ animal: [ 'Cat', 'Dog' ] }
```

### 复合结构

对象和数组可以结合使用，形成复合结构。

```yaml
languages:
  - Ruby
  - Perl
  - Python
website:
  YAML: yaml.org
  Ruby: ruby-lang.org
  Python: python.org
  Perl: use.perl.org
```

转换为JavaScript：

```javascript
{ languages: [ 'Ruby', 'Perl', 'Python' ],
  website: 
   { YAML: 'yaml.org',
     Ruby: 'ruby-lang.org',
     Python: 'python.org',
     Perl: 'use.perl.org' } }
```

示例二：

```yaml
companies:
  - 
    id: 1
    name: company_1
    price: 200w
  -
    id: 2
    name: company_2
    price: 500w
```

转换为JavaScript：

```javascript
{ companies: 
   [ { id: 1, name: 'company_1', price: '200w' },
     { id: 2, name: 'company_2', price: '500w' } ] }
```

### 纯量

纯量是最基本的、不可再分的值。以下数据类型都属于 JavaScript 的纯量。

- 字符串
- 布尔值
- 整数
- 浮点数
- Null
- 时间
- 日期

数值直接以字面量的形式表示。

```yaml
number: 12.30
```

转为JavaScript：

```javascript
{ number: 12.3 }
```

布尔值用 true 和 false 表示：

```yaml
isSet: true
```

转为JavaScript：

```javascript
{ isSet: true }
```

null 用 ~ 表示：

```yaml
parent: ~
```

转为JavaScript：

```javascript
{ parent: null }
```

时间采用 ISO8601  格式：

```yaml
iso8601: 2001-12-14t21:59:43.10-05:00
```

转为JavaScript：

```javascript
{ iso8601: Sat Dec 15 2001 10:59:43 GMT+0800 (中国标准时间) }
```

日期采用复合 iso8601 格式的年、月、日表示。

```yaml
date: 2019-08-05
```

转为JavaScript：

```javascript
{ date: Mon Aug 05 2019 08:00:00 GMT+0800 (中国标准时间) }
```

YAML 允许使用两个感叹号，强制转换数据类型。

```yaml
e: !!str 123
f: !!str true
```

转为JavaScript：

```javascript
{ e: '123', f: 'true' }
```

### 字符串

字符串默认不使用引号表示。

```yaml
str: 这是一行字符串
```

转为JavaScript：

```javascript
{ str: '这是一行字符串' }
```

如果字符串之中包含空格或特殊字符，需要放在引号之中。

```yaml
str: '内容： 字符串'
```

转为JavaScript：

```javascript
{ str: '内容： 字符串' }
```

单引号和双引号都可以使用，双引号不会对特殊字符转义。

```yaml
s1: '内容\n字符串'
s2: "内容\n字符串"
```

转为JavaScript：

```javascript
{ s1: '内容\\n字符串', s2: '内容\n字符串' }
```

单引号之中如果还有单引号，必须连续使用两个单引号转义。

```yaml
str: 'labor''s day'
```

转为JavaScript：

```javascript
{ str: 'labor\'s day' }
```

字符串可以写成多行，从第二行开始，必须有一个单空格缩进。换行符会被转为空格。

```yaml
str: 这是一段
  多行
  字符串
```

转为JavaScript：

```javascript
{ str: '这是一段 多行 字符串' }
```

多行字符串可以使用`|`保留换行符，也可以使用`>`折叠换行。

```yaml
this: |
  Foo
  Bar
that: >
  Foo
  Bar
```

转为JavaScript：

```javascript
{ this: 'Foo\nBar\n', that: 'Foo Bar\n' }
```

`+`表示保留文字块末尾的换行，`-`表示删除字符串末尾的换行。

```yaml
s1: |
  Foo
  
  
s2: |+
  Foo
  
  
s3: |-
  Foo
  
  
```

转为JavaScript：

```javascript
{ s1: 'Foo\n', s2: 'Foo\n\n\n', s3: 'Foo' }
```

字符串之中可以插入 HTML 标记。

```yaml
message: |

  <p style="color: red"
    段落
  </p>
```

转为JavaScript：

```javascript
{ message: '\n<p style="color: red"\n  段落\n</p>\n' }
```

### 引用

锚点`&`和别名`*`，可以用来引用。

```yaml
defaults: &defaults
  adapter: postgres
  host:    localhost

development:
  datebase: myapp_development
  <<: *defaults

test:
  datebase: myapp_test
  <<: *defaults
```

`&`用来建立锚点（`defaults`），`<<`表示合并到当前数据，`*`用来引用锚点。

等同于下面：

```yaml
defaults:
  adapter:  postgress
  host:     localhost

develoyment:
  database: myapp_development
  adapter:  postgress
  host:     localhost

test:
  datebase: myapp_test
  adapter:  postgress
  host:     localhost
```

转为JavaScript：

```javascript
{ defaults: { adapter: 'postgres', host: 'localhost' },
  development: 
   { datebase: 'myapp_development',
     adapter: 'postgres',
     host: 'localhost' },
  test: 
   { datebase: 'myapp_test',
     adapter: 'postgres',
     host: 'localhost' } }
```

示例：

```yaml
- &showell Steve
- Clark
- Brian
- Oren
- *showell
```

转为JavaScript：

```javascript
[ 'Steve', 'Clark', 'Brian', 'Oren', 'Steve' ]
```



参考：

<http://www.ruanyifeng.com/blog/2016/07/yaml.html>

在线转换为JavaScript：

<https://www.bejson.com/validators/yaml_editor/>

