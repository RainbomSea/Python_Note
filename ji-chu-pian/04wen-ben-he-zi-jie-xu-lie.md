# 文本和字节序列

## 字符问题

“字符串”是个相当简单的概念：一个字符串是一个字符序列。问题出在“字符”的定义上。 

在2015年，“字符”的最佳定义是 `Unicode` 字符。因此，从 **Python3** 的 `str` 对象中获取的元素是 **Unicode** 字符，这相当于从 **Python2** 的 `unicode` 对象中获取的元素，而不是从 **Python2** 的 `str` 对象中获取的原始字节序列。 

`Unicode`标准把字符的标识和具体的字节表述进行了如下的明确区分:

* 字符的标识，即码位，是 `0~1 114 111` 的数字（十进制），在 `Unicode` 标准中以 `4~6` 个十六进制数字表示，而且加前缀“`U+`”。例 如，字母 `A` 的码位是 `U+0041`，欧元符号的码位是 `U+20AC`，高音谱号的码位是 `U+1D11E`。在 `Unicode 6.3` 中（这是 **Python 3.4** 使用的 标准），约 10% 的有效码位有对应的字符。 

* 字符的具体表述取决于所用的编码。编码是在码位和字节序列之间 转换时使用的算法。在 `UTF-8` 编码中，`A（U+0041）`的码位编码成 单个字节 `\x41`，而在 `UTF-16LE` 编码中编码成两个字节 `\x41\x00`。

把码位转换成字节序列的过程是编码；把字节序列转换成码位的过程是解码。

示例 4-1 编码和解码 

```py
>>> s = 'café' 
>>> len(s)  # ➊ 
4 
>>> b = s.encode('utf8')  # ➋ 
>>> b 
b'caf\xc3\xa9'  # ➌ 
>>> len(b)  # ➍
5 
>>> b.decode('utf8')  # ➎ 
'café 
```
❶ `'café'` 字符串有 `4` 个 `Unicode` 字符。 

❷ 使用 `UTF-8` 把 `str` 对象编码成 `bytes` 对象。 

❸ `bytes` 字面量以 `b` 开头。 

❹ 字节序列 `b` 有 `5` 个字节（在 `UTF-8` 中，`“é”`的码位编码成两个字节）。 

❺ 使用 `UTF-8` 把 `bytes` 对象解码成 `str` 对象。

## 字节概要

新的二进制序列类型在很多方面与 **Python2** 的 `str` 类型不同。首先要知道，**Python** 内置了两种基本的二进制序列类型：**Python3** 引入的不可变 `bytes` 类型和 **Python 2.6** 添加的可变 `bytearray` 类型。（**Python2.6** 也 引入了 `bytes` 类型，但那只不过是 `str` 类型的别名，与 **Python3** 的 `bytes` 类型不同。） 

`bytes` 或 `bytearray` 对象的各个元素是介于 `0~255`（含）之间的整 数，而不像 **Python2** 的 `str` 对象那样是单个的字符。然而，二进制序列的切片始终是同一类型的二进制序列，包括长度为 1 的切片，如示例 4-2 所示。

示例 4-2 包含 5 个字节的 bytes 和 bytearray 对象  

```py
>>> cafe = bytes('café', encoding='utf_8') ➊ 
>>> cafe 
b'caf\xc3\xa9' 
>>> cafe[0] ➋ 
99 
>>> cafe[:1] ➌ 
b'c' 
>>> cafe_arr = bytearray(cafe) 
>>> cafe_arr ➍ 
bytearray(b'caf\xc3\xa9') 
>>> cafe_arr[-1:] ➎ 
bytearray(b'\xa9') 
```

❶ `bytes` 对象可以从 `str` 对象使用给定的编码构建。 

❷ 各个元素是 `range(256)` 内的整数。 

❸ `bytes` 对象的切片还是 `bytes` 对象，即使是只有一个字节的切片。 

❹ `bytearray` 对象没有字面量句法，而是以 `bytearray()` 和字节序列字面量参数的形式显示。 

❺ `bytearray` 对象的切片还是 `bytearray` 对象。

> my_bytes[0] 获取的是一个整数，而 my_bytes[:1] 返回的 是一个长度为 1 的 bytes 对象——这一点应该不会让人意 外。s[0] == s[:1] 只对 str 这个序列类型成立。不过，str 类 型的这个行为十分罕见。对其他各个序列类型来说，s[i] 返回一 个元素，而 s[i:i+1] 返回一个相同类型的序列，里面是 s[i] 元 素。 

虽然二进制序列其实是整数序列，但是它们的字面量表示法表明其中有 `ASCII` 文本。因此，各个字节的值可能会使用下列三种不同的方式显示。 

* 可打印的 `ASCII` 范围内的字节（从空格到 `~`），使用 `ASCII` 字符本 身。 

* 制表符、换行符、回车符和 `\` 对应的字节，使用转义序列 `\t`、`\n`、`\r` 和 `\\`。 

* 其他字节的值，使用十六进制转义序列（例如，`\x00` 是空字节）

因此，在示例 4-2 中，我们看到的是 `b'caf\xc3\xa9'`：前 3 个字节 `b'caf'` 在可打印的 `ASCII` 范围内，后两个字节则不然。 

## 了解编解码问题 

虽然有个一般性的 `UnicodeError` 异常，但是报告错误时几乎都会指明 具体的异常：`UnicodeEncodeError`（把字符串转换成二进制序列时） 或 `UnicodeDecodeError`（把二进制序列转换成字符串时）。如果源码 的编码与预期不符，加载 **Python** 模块时还可能抛出 `SyntaxError`。

> 出现与 Unicode 有关的错误时，首先要明确异常的类型。导 致编码问题的是 UnicodeEncodeError、UnicodeDecodeError， 还是如 SyntaxError 的其他错误？解决问题之前必须清楚这一 点。 

### 处理UnicodeEncodeError 

多数非 `UTF` 编解码器只能处理 `Unicode` 字符的一小部分子集。把文本转换成字节序列时，如果目标编码中没有定义某个字符，那就会抛出 `UnicodeEncodeError` 异常，除非把 `errors` 参数传给编码方法或函数，对错误进行特殊处理。

示例 4-6 编码成字节序列：成功和错误处理:

```py
>>> city = 'São Paulo' 
>>> city.encode('utf_8') ➊ 
b'S\xc3\xa3o Paulo' 
>>> city.encode('utf_16') 
b'\xff\xfeS\x00\xe3\x00o\x00 \x00P\x00a\x00u\x00l\x00o\x00' 
>>> city.encode('iso8859_1') ➋ 
b'S\xe3o Paulo' 
>>> city.encode('cp437') ➌ 
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "/.../lib/python3.4/encodings/cp437.py", line 12, in encode
        return codecs.charmap_encode(input,errors,encoding_map) 
UnicodeEncodeError: 'charmap' codec can't encode character '\xe3' in 
position 1: character maps to <undefined> 
>>> city.encode('cp437', errors='ignore') ➍
b'So Paulo' 
>>> city.encode('cp437', errors='replace') ➎ 
b'S?o Paulo' 
>>> city.encode('cp437', errors='xmlcharrefreplace') ➏ 
b'São Paulo' 
```

❶ `'utf_?'` 编码能处理任何字符串。 

❷ `'iso8859_1'` 编码也能处理字符串 `'São Paulo'`。 

❸ `'cp437'` 无法编码 `'ã'`（带波形符的`“a”`）。默认的错误处理方式 `'strict'` 抛出 `UnicodeEncodeError`。 

❹ `error='ignore'` 处理方式悄无声息地跳过无法编码的字符；这样做 通常很是不妥。 

❺ 编码时指定 `error='replace'`，把无法编码的字符替换成 `'?'`；数 据损坏了，但是用户知道出了问题。 

❻ `'xmlcharrefreplace'` 把无法编码的字符替换成 `XML` 实体。

### 处理UnicodeDecodeError 

不是每一个字节都包含有效的 `ASCII` 字符，也不是每一个字符序列都是有效的 `UTF-8` 或 `UTF-16`。因此，把二进制序列转换成文本时，如果假设是这两个编码中的一个，遇到无法转换的字节序列时会抛出 `UnicodeDecodeError`。 

示例 4-7 把字节序列解码成字符串：成功和错误处理 

```py
>>> octets = b'Montr\xe9al'  ➊ 
>>> octets.decode('cp1252')  ➋ 
'Montréal' 
>>> octets.decode('iso8859_7')  ➌ 
'Montrιal' 
>>> octets.decode('koi8_r')  ➍ 
'MontrИal' 
>>> octets.decode('utf_8')  ➎ 
Traceback (most recent call last):
    File "<stdin>", line 1, in <module> 
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 5: 
invalid continuation byte 
>>> octets.decode('utf_8', errors='replace')  ➏ 
'Montral' 
```

❶ 这些字节序列是使用 `latin1` 编码的`“Montréal”`；'`\xe9'` 字节对应`“é”`。 

❷ 可以使用 `'cp1252'`（`Windows 1252`）解码，因为它是 `latin1` 的有 效超集。 

❸ `ISO-8859-7` 用于编码希腊文，因此无法正确解释 `'\xe9'` 字节，而且 没有抛出错误。 

❹ `KOI8-R` 用于编码俄文；这里，`'\xe9'` 表示西里尔字母`“И”`。 

❺ `'utf_8'` 编解码器检测到 `octets` 不是有效的 `UTF-8` 字符串，抛出 `UnicodeDecodeError`。 

❻ 使用 `'replace'` 错误处理方式，`\xe9` 替换成了`“ ”`（码位是U+FFFD），这是官方指定的 `REPLACEMENT CHARACTER`（替换字符），表示未知字符。 

### 使用预期之外的编码加载模块时抛出的 SyntaxError 

**Python 3** 默认使用 `UTF-8` 编码源码，**Python 2**（从 `2.5` 开始）则默认使用 `ASCII`。如果加载的`.py`模块中包含 `UTF-8` 之外的数据，而且没有声明编码，会得到类似下面的消息： 

```py
SyntaxError: Non-UTF-8 code starting with '\xe1' in file ola.py on line
  1, but no encoding declared; see http://python.org/dev/peps/pep-0263/
  for details 
```

`GNU/Linux` 和 `OS X` 系统大都使用 `UTF-8`，因此打开在 `Windows` 系统中使用 `cp1252` 编码的 `.py` 文件时可能发生这种情况。注意，这个错误在 `Windows` 版 **Python** 中也可能会发生，因为 **Python 3** 为所有平台设置的默 认编码都是 `UTF-8`。 

为了修正这个问题，可以在文件顶部添加一个神奇的 coding 注释，如 示例 4-8 所示。 

示例 4-8 ola.py：“你好，世界！”的葡萄牙语版 

```py
# coding: cp1252 

print('Olá, Mundo!')
```



