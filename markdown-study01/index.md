# Markdown Study01

<!--more-->
# 1 标题
## h2 标题
### h3 标题
#### h4 标题
##### h5 标题
###### h6 标题、
要添加自定义标题ID，请在与标题相同的行中将自定义ID放在花括号中
<!-- 注释 -->
> ###一个很棒的标题  {#custom-id}

# 2 注释
注释和HTML兼容的
<!-- 这是注释 -->

# 3 水平线
HTML中的<hr\>标签是用来在段落元素之间创建一个“专题间隔”的
Markdown可以使用以下方式：

\---：三个连续的破折号

\___：三个连续的下划线

\***：三个连续的星号

---
___
***

# 4 段落
HTML中将用<p\></p\>标签包裹
Markdown直接按照纯文本的方式书写段落

# 5 内联HTML元素
<div>
这是<b>HTML</b>
</div>

# 6 强调
## 加粗
用于强调带有较粗字体的文本片段
以下文本会被渲染为粗体
```markdown
**渲染为粗体**
__渲染为粗体__
```
输出的HTML看起来像这样
```html
<strong>渲染为粗体</strong>
```
## 斜体
用于强调带有斜体的文本片段

以下文本片段被渲染成斜体
```markdown
*渲染为斜体*
_渲染为斜体_
```
输出的HTML看起来像这样
```html
<em>渲染为斜体</em>
```
## 删除线
以下文本片段会被渲染成删除线
```markdown
~~这段文本带有删除线~~
```
输出的HTML看起来像这样
```html
<del>这段文本带有删除线</del>
```
~~这段文本带有删除线~~
## 组合
加粗，斜体和删除线可以组合使用
```markdown
***加粗和斜体***
~~**删除线和加粗**~~
~~*删除和斜体*~~
~~***加粗，斜体和删除线***~~
```
输出的HTML看起来像这样
```html
<em><strong>加粗和斜体</strong></em>
<del><strong>删除线和加粗</strong></em>
<del><em>删除线和斜体<em></del>
<del><em><strong>加粗，斜体和删除线</strong></em></del>
```
***加粗和斜体***
~~**删除线和加粗**~~
~~*删除和斜体*~~
~~***加粗，斜体和删除线***~~
# 7 引用
用于在文档中引用其他来源的内容块
在要引用的任何文本之前添加>:
```markdown
> **Fusion Drive** combines a hard drive with a flash storage
```
输出的HTML看起来像这样
```html
<blockquote>
    <p>
        <strong>Fusion Drive</strong> combines a hard drive with a flash storage
    </p>
</blockquote>
```
引用也可以嵌套
```markdown
> **Fusion Drive** combines a hard drive with a flash storage
>> 111
```
> **Fusion Drive** combines a hard drive with a flash storage
>> 111
# 8 列表
## 无序列表
一系列项的列表，其中项的顺序没有明显关系
你可以使用以下任何符号来表示无序列表中的项
```markdown
* 一项内容
    * 子内容
- 一项内容
+ 一项内容
```

* 一项内容
    * 子内容
- 一项内容
+ 一项内容
## 有序列表
一系列项的列表，其中项的顺序确实很重要
```markdown
    1. a
    2. b
    3. c
```
输出的HTML看起来像这样
```html
<ol>
    <li>a</li>
    <li>b</li>
    <li>c</li>
</ol>
```
1. a
2. b
3. c
## 任务列表
任务列表使你可以创建带有复选框的列表，要创建列表，请在任务列表项之前添加破折号(-)和带有空格的方括号([ ])，要选择复选框，请在方括号之间添加x([x])
```markdown
    - [x] a
    - [ ] b
    - [ ] c
```
- [x] Write the press release
- [ ] Update the website
- [ ] Contact the media
# 9 代码
## 行内代码
用\包装行内代码段
```markdown
在这个例子中, \<section>\</section> 会被包裹成 **代码**.
```
```html
<p>
  在这个例子中, <code>&lt;section&gt;&lt;/section&gt;</code> 会被包裹成 <strong>代码</strong>.
</p>
```
呈现的输出效果如下：

在这个例子中, \<section>\</section> 会被包裹成 **代码**.
## 缩进代码
```markdown
    // Some comments
    line 1 of code
    line 2 of code
    line 3 of code
```
输出的HTML看起来像这样：
```html
<pre>
  <code>
    // Some comments
    line 1 of code
    line 2 of code
    line 3 of code
  </code>
</pre>
```
呈现的输出效果如下：

    // Some comments
    line 1 of code
    line 2 of code
    line 3 of code
## 围栏代码块
使用“围栏”```来生成一段带有语言属性的代码块
```markdown
Sample text here...
```
<pre language-html>
  <code>Sample text here...</code>
</pre>
# 10 表格
通过在每个单元格之间添加竖线作为分隔线，并在标题下添加一行破折号(也由竖线分隔)来创建表格，注意竖线不需要垂直对齐

在任何标题下方的破折号右侧添加冒号将使该列的文本右对齐

在任何标题下方的破折号两边添加冒号将使该列的对齐文本居中
```markdown
| Option | Description |
| ------ | ----------- |
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |
```
输出的HTML看起来像这样
```html
<table>
  <thead>
    <tr>
      <th>Option</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>data</td>
      <td>path to data files to supply the data that will be passed into templates.</td>
    </tr>
    <tr>
      <td>engine</td>
      <td>engine to be used for processing templates. Handlebars is the default.</td>
    </tr>
    <tr>
      <td>ext</td>
      <td>extension to be used for dest files.</td>
    </tr>
  </tbody>
</table>

```
| Option | Description |
| ------ | ----------- |
| data   | path to data files to supply the data that will be passed into templates. |
| engine | engine to be used for processing templates. Handlebars is the default. |
| ext    | extension to be used for dest files. |
# 链接
## 基本链接
```markdown
<https://assemble.io>
<contact@revolunet.com>
[Assemble](https://assemble.io)
```
输出的HTML看起来像这样
```html
<a href="https://assemble.io">https://assemble.io</a>
<a href="mailto:contact@revolunet.com">contact@revolunet.com</a>
<a href="https://assemble.io">Assemble</a>
```

呈现的输出效果如下(将鼠标悬停在链接上，没有提示)
<https://assemble.io>

<contact@revolunet.com>

[Assemble](https://assemble.io)
## 添加一个标题
```markdown
[Upstage](https://github.com/upstage/ "Visit Upstage!")
```
输出的HTML看起来像这样
```html
<a href="https://github.com/upstage/" title="Visit Upstage!">Upstage</a>
```
呈现的输出效果如下(将鼠标悬停在链接上，会有一行提示)

[Upstage](https://github.com/upstage/ "Visit Upstage!")
# 12 脚注
脚注使你可以添加注释和参考，而不会使文档正文混乱，当你使用脚注时，会在添加脚注引用的位置出现带有链接的上标标号，读者可以单击链接以跳至页面底部的脚注内容

要创建脚注引用，请在方括号中添加插入符号和标识符 ([^1])，标识符可以是数字或单词，但不能包含空格或制表符。标识符仅将脚注引用与脚注本身相关联 - 在脚注输出中, 脚注按顺序编号。

在中括号内使用插入符号和数字以及用冒号和文本来添加脚注内容 ([^1]：这是一段脚注)。你不一定要在文档末尾添加脚注. 可以将它们放在除列表，引用和表格等元素之外的任何位置。
```markdown
这是一个数字脚注[^1].
这是一个带标签的脚注[^label]

[^1]: 这是一个数字脚注
[^label]: 这是一个带标签的脚注
```

这是一个数字脚注[^1].
这是一个带标签的脚注[^label]

[^1]: 这是一个数字脚注
[^label]: 这是一个带标签的脚注
# 13 图片
图片的语法与链接相似，但包含一个在前面的感叹号
```markdown
![Minion](https://octodex.github.com/images/minion.png)
![Alt text](https://octodex.github.com/images/stormtroopocat.jpg "The Stormtroopocat")
```
![Minion](https://octodex.github.com/images/minion.png)
![Alt text](https://octodex.github.com/images/stormtroopocat.jpg "The Stormtroopocat")
