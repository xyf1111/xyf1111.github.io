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
