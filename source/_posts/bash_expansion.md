---
title: Shell Expansion
date: 2019-08-21 17:55:57
toc: true
tags:
- shell
---

## 说明

在将命令行拆分为 tokenS 后，将在命令行上执行扩展。

## 七种 expansion

按照 expansion 的顺序排序如下：

- Brace Expansion
- Tilde Expansion
- Parameter and Variable Expansion
- Arithmetic Expansion
- Command Substitution
- Word Splitting
- Filename Expansion

<!-- more -->

在能够支持 Process substitution 的机器上，这个流程与 tilde，parameter，variable 和 arithmetic expansion、command substitution 同时执行。

只有 brace expansion，word splitting 和 filename expansion 才能增加扩展的字数；其他扩展将单个单词扩展为单个单词（意思是不增加字数）。唯一的例外是 `"$@"` 和 `$*` 扩展（见 [Special Parameter](https://www.gnu.org/software/bash/manual/html_node/Special-Parameters.html#Special-Parameters)）和 `"${name[@]}"` 和 `"${name[*]}"`（见 [Arrays](https://www.gnu.org/software/bash/manual/html_node/Arrays.html#Arrays)）

在执行这些扩展之后，将删除原单词中出现的引号字符，除非它们本身已经被引用(Quote Removal)。

### [Brace Expansion](https://www.gnu.org/software/bash/manual/html_node/Brace-Expansion.html#Brace-Expansion)

括号内扩展，可以嵌套，每个扩展的结果都没有排序，保持从左到右的顺序。

#### 表达式一，用逗号分割

```bash
$ echo a{b,c,d}e
abe ace ade
```

#### 表达式二，`{x..y[..incr]}`

x 和 y 可以是数字或者字母(单字母), incr 是 int 类型递增量，默认是 1 或者 -1:

- 数字

```bash
$ echo a{10..40..10}c
a10c a20c a30c a40c
```

- 子母

```bash
$ echo a{a..c}d
aad abd acd
```

#### 注意点

- Brace expansion 会优先于其他扩展执行，并且在结果中保留对其他扩展特殊的任何字符。这是严格的文字话的，bash 不对 brace expansion 之间的文本应用任何语法解释

- 包含不带引号的 `{` 或 `}`，以及至少一个不带引号的 `,` 或者有效的序列表达式

- `{` 或 `,` 可以用 `\` 引用来防止它们被视为 brace expansion 表达式的一部分。为了避免冲与 parameter expansion 冲突，字符串 `${` 对 brace expansion 来说是非法的，并且禁止 brace expansion 直至遇到一个 `}`

### [Tilde Expansion](https://www.gnu.org/software/bash/manual/html_node/Tilde-Expansion.html#Tilde-Expansion)

波浪线扩展，下面显示了 bash 如何处理不带引号的波浪号前缀：

表达式 | 描述
--- | ---
`~` | $HOME 值
`~/foo`| $HOME/foo
`~fred/foo` |用户 fred 下的 foo 目录，即 /Users/fred/foo
`~+/foo` | $PWD/foo
`~-/foo` | ${OLDPWD-'~-'}/foo
`~N` | 等同于 `dirs +N`
`~+N` | 等同于 `dirs +N`
`~-N` | 等同于 `dirs -N`

#### 注意点

- 当 Bash 作为简单命令的参数出现时，Bash 还会对满足变量赋值条件的单词执行波浪扩展（请参阅[Shell参数](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html#Shell-Parameters）)。在 POSIX 模式下，Bash 不执行此操作，但上面列出的声明命令除外。

### [Shell Parameter Expansion](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html#Shell-Parameter-Expansion)

参数扩展：

- 普通扩展，如果省略冒号，则不检查是否为 null，而是只检查是否存在

表达式 | 描述
--- | ---
${parameter:-word} | 如果 parameter 未设置或者为 null，则替换为 word
${parameter:=word} | 如果 parameter 未设置或者为 null，则将 word 赋值给 parameter，不能以这种方式赋值给 positional parameters 和 special parameters
${parameter:?word} | 如果 parameter 未设置或者为 null，则报错并写入 stderr，如果 shell 不是交互式的就退出，否则参数值将被替换
${parameter:+word} | 如果 parameter 未设置或者为 null，则不替换，否则替换

- 间接扩展

```bash
${!parameter}
# 相当于${var}，而var=${parameter}。比如说：
$ param="var"
$ var="Hello, Bread"
$ echo ${!param}
Hello, Bread
```

- 子串扩展（截取子串）

表达式 | 描述
--- | ---
${parameter:offset} | 省略 length，从 offset 至末尾
${parameter:offset:length} | 如果 offset 小于零，则该值将用作 parameter 末尾的字符偏移量。如果 length 小于零，则将其解释为字符与 parameter 末尾的偏移量而不是字符数。请特别注意：负偏移量必须与冒号分隔至少一个空格以避免 `:-` expansion。

以下是一些示例，说明 parameter 和下标数组的字符串扩展：

```bash
$ str=01234567890abcdefgh
$ echo ${str:7}
7890abcdefgh
$ echo ${str:7:0}

$ echo ${str:7:2}
78
$ echo ${str:7:-2}
7890abcdef
$ echo ${str: -7}
bcdefgh
$ echo ${str: -7:0}

$ echo ${str: -7:2}
bc
$ echo ${str: -7:-2}
bcdef
```

如果参数是 `'@'`，结果是从 offset 开始的 length 长度的参数。相对于大于最大位置参数的负偏移，因此偏移 -1 为最后一个参数的位置。如果 length 小于零，则报错。

```bash
$ set -- 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${@:7}
7 8 9 0 a b c d e f g h
$ echo ${@:7:0}

$ echo ${@:7:2}
7 8
$ echo ${@:7:-2}
bash: -2: substring expression < 0
$ echo ${@: -7:2}
b c
$ echo ${@: -7:0}
/bash 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${@:0:2}
/bash 1
$ echo ${@: -7:0}

```

如果参数是由 `@` 或 `*` 索引的数组，结果是从 offset 开始的 length 长度的参数。相对于大于最大位置参数的负偏移，因此偏移 -1 为最后一个参数的位置。如果 length 小于零，则报错。

```bash
$ array=(0 1 2 3 4 5 6 7 8 9 0 a b c d e f g h)
$ echo ${array[@]:7}
7 8 9 0 a b c d e f g h
$ echo ${array[@]:7:2}
7 8
$ echo ${array[@]: -7:2}
b c
$ echo ${array[@]: -7:-2}
bash: -2: substring expression < 0
$ echo ${array[@]:0}
0 1 2 3 4 5 6 7 8 9 0 a b c d e f g h
$ echo ${array[@]:0:2}
0 1
$ echo ${array[@]: -7:0}

```

应用于关联数组的子串扩展会产生未定义的结果

除非使用位置参数，否则子串索引从零开始，在这种情况下，索引默认从 1 开始。如果 offset 为0，并且使用了位置参数，则会在 list 中添加 `$@` 前缀

- 查找参数名

```bash
${!prefix*}
${!prefix@}
```

shell 会自动将其展开为所有以 prefix 开头的参数名。如：

``` bash
$ var1=
$ var2=
$ echo ${!var*}
var1 var2
```

- 数组 key 展开

```bash
${!name[@]}
${!name[*]}
```

如果 name 是数组变量，则展开到 name 中指定的数组索引列表。如果 name 不是数组，则在设置 name 时扩展为 0，否则为 null。用 `@` 的时候，扩展出现在双引号内，每个键扩展为一个单独的单词。

- 参数长度

```bash
${#parameter}
```

例子:

```bash
$ name=123456789
$ echo ${#name}
9
$ array=(a b c)
$ echo ${#array[@]} # echo ${#array[*]}
3
```

- "掐头去尾"

截取又分两种操作，"掐头"和去"去尾"。与正则表达式中不同，头和尾在这里对应的符号分别是"#"和"%"。

```bash
# 掐头
${parameter#word} 或 ${parameter##word}
```

作用：从 parameter 头部开始匹配 word，并删除成功匹配的部分。在构造 word 时可以使用 "*" 表示任意长度字符，"?" 表示单位长度字符，并可用形如 "[a-c]" 的方式来指定匹配 "abc" 中的任意字符。如果参数是 `@` 或者 `*`，则将规则应用于每个位置的参数并返回结果列表，如果参数是数组且下标为 `@` 或者 `*`，则将规则应用于每个数组元素，并返回数组

另外，"#" 和 "##" 的区别在于前者是最短匹配，而后者是最长匹配；实际上就是正则表达式中的"懒惰"和"贪婪"的概念。下面的例子能够很清楚地看出这两者的区别。比如说:

```bash
$ var=brbread
$ echo ${var##*br} # 匹配最长
ead
$ echo ${var#*br} # 只匹配一次
bread
```

```bash
# 去尾
${parameter%word} 或 ${parameter%%word}
```

作用: 与掐头相同，唯一不同的是从 ${parameter} 的尾部开始匹配。如果参数是 `@` 或者 `*`，则将规则应用于每个位置的参数并返回结果列表，如果参数是数组且下标为 `@` 或者 `*`，则将规则应用于每个数组元素，并返回数组，例子:

```bash
$ var="La.Maison.en.Petits.Cubes.avi"
$ echo ${var%.*}
La.Maison.en.Petits.Cubes
$ echo ${var%%.*}
La
```

- 字符串替换

```bash
${parameter/pattern/string}
```

作用: 将 ${parameter| 中出现的第一个 pattern 替换为 string。如果参数是 `@` 或者 `*`，则将规则应用于每个位置的参数并返回结果列表，如果参数是数组且下标为 `@` 或者 `*`，则将规则应用于每个数组元素，并返回数组。值得注意的是，除了 "*", "?" 和 "[]" 以外，pattern 的头部还可以使用下面几个字符:

表达式 | 描述
--- | ---
"/" | 如果 pattern 以 "/" 起始，则所有的匹配项都要替换。而默认的行为只是替换最左侧的一个
"#" | 如果 pattern 以 "#" 起始，则与正则表达式匹配规则相同，只有在 ${parameter} 的头部找到匹配项才会进行替换
"%" | 与 "#" 类似，只是这次变成了尾部匹配

例子：

```bash
var="see"
echo ${var/e/a} #只有第一个e会被替换
sae

echo ${var/ee/it}
sit

echo ${var//e/a} #所有的e都会被替换
saa

echo ${var/#e/a} #头部的e才会被替换
see

echo ${var/#s/b}
bee

echo ${var/%e/a}
sea
```

- 大小写转换

此展开修改参数中字母字符的大小写。该模式被展开以生成与文件名展开中一样的模式。参数展开值中的每个字符都将根据模式进行测试，如果匹配模式，则转换其大小写。该模式不应尝试匹配多个字符。

表达式 | 描述
--- | ---
${parameter^pattern} | `^` 操作符将匹配模式的小写字母转换为大写字母，只转换第一个字符
${parameter,pattern} | `,` 操作符将匹配的大写字母转换为小写字母，只转换第一个字符
${parameter,,pattern} <br> ${parameter^^pattern} | `^^` 和 `,,` 反转所有匹配到的字符

如果省略了 pattern，则认为是 `?`，将匹配所有字符。如果参数是 `@` 或者 `*`，则将规则应用于每个位置的参数并返回结果列表，如果参数是数组且下标为 `@` 或者 `*`，则将规则应用于每个数组元素，并返回数组。

- 一些特殊展开

```bash
${parameter@operator}
```

这个展开可以是参数值的转换，也可以是参数本身的信息，这取决于操作符的值。每一个操作符是一个字母：

操作符 | 描述
--- | ---
Q | 展开是一个字符串，它是可以作为输入重用的格式中引用的参数值
E | 展开是一个字符串，它是参数的值，使用 $'…' 引用机制展开反斜杠转义序列
P | 展开是一个字符串，它是将 parameter 的值展开为提示字符串的结果(请参见[控制提示](https://www.gnu.org/software/bash/manual/html_node/Controlling-the-Prompt.html#Controlling-the-Prompt))
A | 展开是赋值语句或声明命令形式的字符串，如果对其求值，将用其属性和值重新创建参数
a | 展开是由表示参数属性的标志值组成的字符串

如果参数是 `@` 或者 `*`，则将规则应用于每个位置的参数并返回结果列表，如果参数是数组且下标为 `@` 或者 `*`，则将规则应用于每个数组元素，并返回数组。

### [Command Substitution](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html#Command-Substitution)

```bash
$(command) or `command`
```

Bash 通过在子 shell 环境中执行命令并使用命令的标准输出替换命令替换来执行扩展，并删除任何尾随的换行符。嵌入的换行不会被删除。但是在 word splitting 的时候会被移除。 命令 `${cat file}` 可以被执行得更快的 `$(< file)` 替代。

当使用反引号的形式进行替换时，反斜杠保留其字面意义，除非后面跟着 `'$'`，``'`'``，要么 `'\'`。第一个不带反斜杠的反引号会终止命令替换。当用 `${command}` 的形式，所有括号之间的字符组成命令，没有任何特殊的情况。

command substitution 可以嵌套，要在使用反引号形式时进行嵌套，请使用反斜杠转义内部反引号。如果替换出现在双引号中，则不会对结果执行 word splitting 和 文件名扩展。

### [Arithmetic Expansion](https://www.gnu.org/software/bash/manual/html_node/Arithmetic-Expansion.html#Arithmetic-Expansion)

```bash
$(( expression ))
```

该表达式被视为在双引号内，但括号内的双引号没有被特别处理。表达式中的所有标记都经 parameter expansion，variable expansion，command substitution，quote removal。结果被视为要计算的算术表达式。算术扩展可以嵌套。如果表达式无效，Bash 将打印一条消息，指示标准错误失败并且不会发生替换。

计算规则: [Shell Arithmetic](https://www.gnu.org/software/bash/manual/html_node/Shell-Arithmetic.html#Shell-Arithmetic)

### [Process Substitution](https://www.gnu.org/software/bash/manual/html_node/Process-Substitution.html#Process-Substitution)

Process substitution 允许使用文件名引用进程的输入或输出。

```bash
<(list)
# or
>(list)
```

进程列表以异步的方式运行，其输入或输出显示为文件名。作为扩展的结果，此文件名作为参数传递给当前命令。如果使用 `>(list)` 形式，写入文件将提供列表输入。如果使用 `<(list)`，则应读取作为参数传递的文件以获取列表的输出。请注意，在 `<` 或 `>` 与左括号之间不能有空格，否则会被解释为重定向。支持命名管道（FIFO）或者命名打开文件的 `/dev/fd` 方法的系统支持 process substitution。

可用时，进程替换与参数和变量扩展，命令替换和算术扩展同时执行。

### [Word Splitting](https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html#Word-Splitting)

Shell 扫描参数扩展、命令替换和算术扩展的结果，这些结果在双引号内没有应用 word splitting。

Shell 将 `IFS` 的每个字符视为分隔符，并使用这些字符作为字段终止符将其他扩展的结果拆分为单词。

如果 `IFS` 没有设置或者就是默认值 `<space><tab><newline>`，那么扩展结果的开始及末尾的 `<space><tab><newline>` 会被忽略，并且任何不在开头或者结尾的 `IFS` 字符序列都用于分隔单词。

如果 `IFS` 具有缺省值以外的值，则忽略开始及结尾的空白字符序列 `<space><tab><newline>`，只要空白字符是在`IFS` 里的值（一个 `IFS` 空白字符）。非 `IFS` 空白的 `IFS` 中的任何字符，以及任何相邻的 `IFS` 空白字符，都将分隔字段。`IFS` 空格字符序列也被当作分隔符。如果 `IFS` 的值为null，则不会发生分词。

保留显式空参数（" " 或 ' '）并将其作为空字符串传递给命令。将删除由于没有值的参数的扩展而产生的不带引号的隐式空参数。如果在双引号内扩展没有值的参数，则会生成并返回 null 参数，并将其作为空字符串传递给命令。当引用的 null 参数作为扩展为非 null 的单词的一部分出现时，将删除 null 参数。也就是说，在 word splitting 和 null argument removal 之后，单词 `-d' '`变为 `-d`。

请注意，如果未发生 expansion，则不会进行拆分

### [Filename Expansion](https://www.gnu.org/software/bash/manual/html_node/Filename-Expansion.html#Filename-Expansion)

- [Pattern Matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html#Pattern-Matching): How the shell matches patterns

word splitting 之后，除非设置 `-f` 选项（请参阅 [The Set Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html#The-Set-Builtin)）， bash 会扫描每个单词的字符 `'*'`，`'?'`，`'['`。如果出现了这些字符的任意一个，那么这个单词被视为是个 pattern，并替换为与该 pattern 匹配的按字母顺序排列的文件名列表（请参阅 [Pattern Matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html#Pattern-Matching)）。如果未找到匹配的文件名，并且禁用了 shell 的 `nullglob` 选项，则该字保持不变。如果设置了 `nullglob` 选项，但未找到匹配项，则删除该单词。如果设置了 shell 的 `failglob` 选项，并且未找到匹配项，则会打印错误消息并且不执行该命令。如果启用了 shell 的 `nocaseglob` 选项，则执行匹配而不考虑字母字符的情况。

当一个 pattern 用于文件名扩展时，除非设置了 shell 的 `dotglob` 这个选项，否则文件名开头的字符 `'.'` 或者 `'./'` 应该明确匹配。即使 `dotglob` 已经设置，`'.'` 及 `'..'` 文件名都必须显式匹配。其他情况下，`'.'` 字符不受特别对待。

匹配文件名时，斜杠字符必须始终通过模式中的斜杠显式匹配，但在其他匹配上下文中，它可以通过特殊模式字符进行匹配，如下所述（请参阅 [Pattern Matching](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html#Pattern-Matching)）。

`nocaseglob`, `nullglob`, `failglob` 和 `dotglob` 的描述请见 [The Shopt Builtin](https://www.gnu.org/software/bash/manual/html_node/The-Shopt-Builtin.html#The-Shopt-Builtin) `shopt` 描述。

Shell 的 `GLOBIGNORE` 变量可以用来限制该组匹配的文件名。如果设置了 `GLOBIGNORE`,`GLOBIGNORE` 则从匹配列表中删除与其中一个模式匹配的每个匹配文件名 。如果设置了 `nocaseglob`，`GLOBIGNORE` 则执行与模式的匹配 而不考虑大小写。文件名 `.` 和 `..` 在 `GLOBIGNORE` 设置时始终忽略，而不是 null。但是，设置 `GLOBIGNORE` 为非 null 值具有启用 `dotglob` shell 选项的效果，因此所有其他以 `'.'` 开头的文件都会匹配。为了获得忽略文件名以 `'.'` 开头的行为，就在 `GLOBIGNORE` 模式下设置 `'.*'` 的 patterns。在未设置 `GLOBIGNORE` 时， `dotglob` 是禁用的。

### [Quote Removal](https://www.gnu.org/software/bash/manual/html_node/Quote-Removal.html#Quote-Removal)

在进行前面的扩展后，所有出现的未被的引号引用的字符如：`\`，`'`，`"`，这些不是由上述某个扩展产生的结果都会被删除
