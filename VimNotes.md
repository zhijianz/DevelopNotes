
<!-- toc orderedList:0 -->

- [操作文本](#操作文本)
- [复制粘贴系列](#复制粘贴系列)

<!-- tocstop -->

> Vim 编辑器的使用笔记，记录常用的一些命令

# 操作文本

这是一段用于练习 的操作文本，这段文本的内容没有多少意义，中间可能会包含English中文和数字。
这是一段用于练习 Vim 的操作文本，这段文本的内容没有多少意义，中间可能会包含English中文和数字。
这是一段用于练习 Vim 的操作文本，这段文本的内容没有多少意义，中间可能会包含English中文和数字。
这是一段用于练习 Vim 的操作文本，这段文本的内容没有多少意义，中间可能会包含English中文和数字。

# 复制粘贴系列

复制粘贴的时候必然会伴随着选择过程的发生。下面的命令优先介绍选择、复制、删除的基础操作然后再进行常用命令的介绍

```
v: 字符选择，将光标经过的字符选择
V：行选择，将光标经过的行选择
Ctrl + V：矩形选择， 可以用矩形的方式选择光标
y: 将选中的地方复制起来
d: 将选中的地方删除
```

```
dd：剪切当前行
ndd：n表示大于1的数字，剪切n行
dw：从光标处剪切至一个单子/单词的末尾，包括空格
de：从光标处剪切至一个单子/单词的末尾，不包括空格
d$：从当前光标剪切到行末
d0：从当前光标位置（不包括光标位置）剪切之行首
d3l：从光标位置（包括光标位置）向右剪切3个字符
d5G：将当前行（包括当前行）至第5行（不包括它）剪切
d3B：从当前光标位置（不包括光标位置）反向剪切3个单词
dH：剪切从当前行至所显示屏幕顶行的全部行
dM：剪切从当前行至命令M所指定行的全部行
dL：剪切从当前行至所显示屏幕底的全部行
```

```
yy：复制当前行
nyy：n表示大于1的数字，复制n行
yw：从光标处复制至一个单子/单词的末尾，包括空格
ye：从光标处复制至一个单子/单词的末尾，不包括空格
y$：从当前光标复制到行末
y0：从当前光标位置（不包括光标位置）复制之行首
y3l：从光标位置（包括光标位置）向右复制3个字符
y5G：将当前行（包括当前行）至第5行（不包括它）复制
y3B：从当前光标位置（不包括光标位置）反向复制3个单词
```