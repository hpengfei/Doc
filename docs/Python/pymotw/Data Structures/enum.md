# enum 枚举类型

`enum` 模块定义了一个具备可迭代性和可比较性的枚举类型。它可以为值创建具有良好定义的标识符，而不是直接使用字面上的字符串或者整数。

## 创建枚举

可以用过使用 `class` 语法创建一个继承自 `Enum` 的类并且加入描述值的类变量来定义一个新的枚举类型。

```python
import enum


class BugStatus(enum.Enum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1


print('\nMember name: {}'.format(BugStatus.wont_fix.name))
print('Member value: {}'.format(BugStatus.wont_fix.value))
```

`Enum` 的成员在类被解析的时候转化为实例。每一个实例都有一个 `name` 属性对应成员的名称，一个 `value` 属性对应在类中赋值给成员名称的值。

```
Member name: wont_fix
Member value: 4
```

## 迭代

对 `enum` 类的迭代将产生独立的枚举成员。

```python
import enum

class BugStatus(enum.Enum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1


for status in BugStatus:
    print('{:15} = {}'.format(status.name, status.value))
```

成员以它们在类中被定义的顺序逐个产生，成员的名称和值没有以任何方式用来对它们进行排序。

```
new             = 7
incomplete      = 6
invalid         = 5
wont_fix        = 4
in_progress     = 3
fix_committed   = 2
fix_released    = 1
```

## 对若干 Enum 类进行比较

由于枚举成员不是有序的，所以它们只支持按 id 或按值得相等性进行比较。

```python
import enum

class BugStatus(enum.Enum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1


actual_state = BugStatus.wont_fix
desired_state = BugStatus.fix_released

print('Equality:', actual_state == desired_state, actual_state == BugStatus.wont_fix)
print('Identity:', actual_state is desired_state, actual_state is BugStatus.wont_fix)
print('Ordered by value:')
try:
    print('\n'.join('  ' + s.name for s in sorted(BugStatus)))
except TypeError as err:
    print('  Cannot sort: {}'.format(err))
```

对枚举成员应用大于和小于比较操作符将抛出 `TypeError` 异常。

```
Equality: False True
Identity: False True
Ordered by value:
  Cannot sort: '<' not supported between instances of 'BugStatus' and 'BugStatus'
```

若想使枚举成员表现得更类似于数学，例如：要支持大于或小于比较，则需要使用 `IntEnum` 类。

```python
import enum

class BugStatus(enum.IntEnum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1


print('Ordered by value:')
print('\n'.join('  ' + s.name for s in sorted(BugStatus)))
```

输出结果：

```
Ordered by value:
  fix_released
  fix_committed
  in_progress
  wont_fix
  invalid
  incomplete
  new
```

## 唯一得枚举值

具有相同值得 Enum 成员会被当作同一对象得别名引用进行跟踪。别名不会导致 `Enum` 得迭代器里面出现重复值。

```python
import enum

class BugStatus(enum.Enum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1

    by_design = 4
    closed = 1


for status in BugStatus:
    print('{:15} = {}'.format(status.name, status.value))

print('\nSame: by_design is wont_fix: ', BugStatus.by_design is BugStatus.wont_fix)
print('Same: closed is fix_released: ', BugStatus.closed is BugStatus.fix_released)
```

因为 `by_design` 和 `closed` 是其他成员得别名，在遍历 `Enum` 得时候，它们都不会出现在输出中。枚举成员中第一个关联到成员值得名称是规范名称。

```
new             = 7
incomplete      = 6
invalid         = 5
wont_fix        = 4
in_progress     = 3
fix_committed   = 2
fix_released    = 1

Same: by_design is wont_fix:  True
Same: closed is fix_released:  True
```

如果想要枚举成员只包含唯一值，可以在 `Enum` 上加上装饰器 `@unique`。

```python
import enum

@enum.unique
class BugStatus(enum.Enum):

    new = 7
    incomplete = 6
    invalid = 5
    wont_fix = 4
    in_progress = 3
    fix_committed = 2
    fix_released = 1

    by_design = 4
    closed = 1
```

当 `Enum` 类被解释器执行得时候，重复得成员会触发一个 `ValueError` 异常。

```
...
ValueError: duplicate values found in <enum 'BugStatus'>: by_design -> wont_fix, closed -> fix_released
```

## 编码创建枚举

在某种情况下，比起以类定义得方式硬编码枚举，用在编码中动态创建得枚举得方式更加方便。针对这种情况，`Enum` 提供了将成员名称和值传递到类构造器得方式创建枚举。

```python
import enum

BugStatus = enum.Enum(
    value='BugStatus',
    names=('fix_released fix_committed in_progress wont_fix invalid incomplete new')
)

print('Member: {}'.format(BugStatus.new))

print('\nAll members:')
for status in BugStatus:
    print('{:15} = {}'.format(status.name, status.value))
```

其中 `value` 参数是枚举的名称，用于构建枚举成员的表示。`names` 变量列出了枚举所包含的成员。当只传递一个字符串的时候，它会被自动以空格和逗号进行分割，并且将结果序列作为枚举成员的名称，这些字段会被自动赋值为从 `1` 开始自增的数字。

```
Member: BugStatus.new

All members:
fix_released    = 1
fix_committed   = 2
in_progress     = 3
wont_fix        = 4
invalid         = 5
incomplete      = 6
new             = 7
```

为了更加方便地控制枚举成员的值，`names` 字符串可以被替换成二元元组组成的序列或者由名称和值组成的字典。

```python
import enum

BugStatus = enum.Enum(
    value='BugStatus',
    names=[
        ('new', 7),
        ('incomplete', 6),
        ('invalid', 5),
        ('wont_fix', 4),
        ('in_progress', 3),
        ('fix_committed', 2),
        ('fix_released', 1),
    ],
)

print('All members:')
for status in BugStatus:
    print('{:15} = {}'.format(status.name, status.value))
```

在这个例子中，二元元组组成的序列替换了由成员名称组成的字符串。

```
All members:
new             = 7
incomplete      = 6
invalid         = 5
wont_fix        = 4
in_progress     = 3
fix_committed   = 2
fix_released    = 1
```

## 非整数成员值

枚举成员并没有限制为整数类型。实际上，枚举成员可以关联任何对象的类型。如果值是一个元组，成员值会作为独立的参数传递给 `__init__()` 函数。

```python
import enum

class BugStatus(enum.Enum):

    new = (7, ['incomplete', 'invalid', 'wont_fix', 'in_progress'])
    incomplete = (6, ['new', 'wont_fix'])
    invalid = (5, ['new'])
    wont_fix = (4, ['new'])
    in_progress = (3, ['new', 'fix_committed'])
    fix_committed = (2, ['in_progress', 'fix_released'])
    fix_released = (1, ['new'])

    def __init__(self, num, transitions):
        self.num = num
        self.transitions = transitions

    def can_transition(self, new_state):
        return new_state.name in self.transitions


print('Name:', BugStatus.in_progress)
print('Value:', BugStatus.in_progress.value)
print('Custom attribute:', BugStatus.in_progress.transitions)
print('Using attribute:', BugStatus.in_progress.can_transition(BugStatus.new))
```

在这个例子中，每一个成员都是包含数字 ID 和从现在的状态可以转变得状态得列表组成得元组。

```
Name: BugStatus.in_progress
Value: (3, ['new', 'fix_committed'])
Custom attribute: ['new', 'fix_committed']
Using attribute: True
```

对于更复杂得情况来说，元组可能会变得比较笨拙。由于成员值可以是任何类型得对象，字典可以被用于需要跟踪每个枚举值得对立属性的情况。复杂的值作为除了 `self` 之外唯一的参数传递给 `__init__()`。

```python
import enum

class BugStatus(enum.Enum):

    new = {
        'num': 7,
        'transitions': [
            'incomplete',
            'invalid',
            'wont_fix',
            'in_progress',
        ],
    }
    incomplete = {
        'num': 6,
        'transitions': ['new', 'wont_fix'],
    }
    invalid = {
        'num': 5,
        'transitions': ['new'],
    }
    wont_fix = {
        'num': 4,
        'transitions': ['new'],
    }
    in_progress = {
        'num': 3,
        'transitions': ['new', 'fix_committed'],
    }
    fix_committed = {
        'num': 2,
        'transitions': ['in_progress', 'fix_released'],
    }
    fix_released = {
        'num': 1,
        'transitions': ['new'],
    }

    def __init__(self, vals):
        self.num =vals['num']
        self.transitions = vals['transitions']

    def can_transitions(self, new_state):
        return new_state.name in self.transitions


print('Name:', BugStatus.in_progress)
print('Value:', BugStatus.in_progress.value)
print('Custom attribute:', BugStatus.in_progress.transitions)
print('Using attribute:', BugStatus.in_progress.can_transitions(BugStatus.new))
```

这个例子将上个例子中的元组替换成了字典。

```
Name: BugStatus.in_progress
Value: {'num': 3, 'transitions': ['new', 'fix_committed']}
Custom attribute: ['new', 'fix_committed']
Using attribute: True
```

