---
title: Markdown 常用指令
date: 2016-04-25 21:50:22
description: 这篇文章是使用hexo写文章会用到的基本语法，记录下来学习过程中自己遇到的问题，也让读者们少走弯路。
tags: 
- Markdown
---
## 标题
标题有1-6级，以#数量为基准
```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
###### 六级标题
```
***************************
<!--more-->
## 斜体、加粗、删除线、下划线
```
*斜体*
_斜体_
**加粗**
__加粗__
**加粗_斜体_加粗**
~~删除线~~
<u>下划线</u>
```
*斜体*
_斜体_
**加粗**
__加粗__
**加粗_斜体_加粗**
~~删除线~~
<u>下划线</u>
***************************
## 阅读全文
如果不想在首页中显示全部文章，在需要显示“阅读全文”的地方加
```
<!--more-->
```
或者，在 markdown 头部的 Front-matter 里面加入 description 字段。优先会使用 description 字段的内容。
***************************
## 分割线
行内不能有其他元素
```
******
```
***
***************************
## 引用
引用使用">"来表示，没有上限。
```
> 一级引用   
>> 二级引用
>>> 三级引用
>>>> 四级引用
>>>>> 五级引用
>>>>>> 。。。。。。
```
> 一级引用   
> > 二级引用
> > > 三级引用
> > > > 四级引用
> > > > > 五级引用
> > > > >
> > > > > > 。。。。。。

***************************

## 有序列表
数字接一个英文句点
```
1. 一
2. 二
```
1. 一
2. 二
***************************
## 无序列表
使用-、*、+都可以，父级与子级不能使用相同的符号。
```
- 嵌套列表
  + 嵌套列表
  + 嵌套列表
    - `嵌套列表`
      * 嵌套列表
- 嵌套列表
```
- 嵌套列表
  + 嵌套列表
  + 嵌套列表
    - `嵌套列表`
      * 嵌套列表
- 嵌套列表
***************************
## 链接
```
[Ronrwin](http://ronrwin.github.io/)

[Ronrwin][a]
[a]: http://ronrwin.github.io/

[Google][]
[google]: http://google.com
```

[Ronrwin](http://ronrwin.github.io/)
[Ronrwin][a]
[a]: http://ronrwin.github.io/
[Google][]
[google]: http://google.com
***************************
## 图片
Markdown语法：![描述]（图片链接地址）
```
![替代文字](http://img.hb.aicdn.com/645a7a8bce82d7ffb515e9f84ffe576e2f5c4c231b448-hnZ9Ml_fw658 "这是图片标题")
```
![替代文字](http://img.hb.aicdn.com/645a7a8bce82d7ffb515e9f84ffe576e2f5c4c231b448-hnZ9Ml_fw658 "这是图片标题")
***************************
## 标记
```
这是一个`标记`
```
这是一个`标记`
***************************
## 代码块
代码块的前后需要\`\`\`
\`\`\`python
`s = "hello Markdown"`
`print s`
\`\`\` 
```python
s = "hello Markdown"
print s
```
***************************
## 表格
```
| Command | Description |
| --- | --- |
| `git status` | List all *new or modified* files |
| `git diff` | Show file differences that **haven't been** staged |
```
| Command | Description |
| --- | --- |
| `git status` | List all *new or modified* files |
| `git diff` | Show file differences that **haven't been** staged |
***************************
## 表格靠左、居中、靠右
```
| Left-aligned | Center-aligned | Right-aligned |
| :---         |     :---:      |          ---: |
| git status   | git status     | git status    |
| git diff     | git diff       | git diff      |
```
| Left-aligned | Center-aligned | Right-aligned |
| :---         |     :---:      |          ---: |
| git status   | git status     | git status    |
| git diff     | git diff       | git diff      |
***************************
## 反斜杠
Markdown 支持在以下这些符号前面加上反斜杠来帮助插入普通的符号：
```
\   反斜线
`   反引号
*   星号
_   底线
{}  花括号
[]  方括号
()  括弧
#   井字号
+   加号
-   减号
.   英文句点
!   惊叹号
```

***************************
## 锚点
```
[回到标题](#标题)
```
[回到标题](#标题)

***************************
## 折叠代码块
[Next主题添加内容折叠](https://www.zybuluo.com/MrXiao/note/1360159)
```html
{% fold 点击显/隐内容 %}
something you want to fold, include code block.
{% endfold %}
```

***************************
## 链接其他站内文章
```html
{% post_link hexo/hexo-install 点击这里查看这篇文章 %}
```
+ {% post_link hexo/hexo-install %} 
+ {% post_link hexo/hexo-install 点击这里查看这篇文章 %}

`markdown-learning-by-maxiang`是在_post目录下，你的文章名称。如果文章不存在，这段代码将会被直接忽略。
`点击这里查看这篇文章`是该链接的标题。如果置空，则自动提取文章的标题。

***************************
## 参考文章
[Basic writing and formatting syntax](https://help.github.com/enterprise/2.1/user/articles/basic-writing-and-formatting-syntax/)
[Markdown 语法说明 (简体中文版)](http://wowubuntu.com/markdown/)
