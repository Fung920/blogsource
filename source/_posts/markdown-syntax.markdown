---
layout: post
title: "Markdown Syntax"
date: 2016-07-08 02:49:35
comments: false
categories: miscellaneous
tags: markdown
keywords: markdown syntax
description: markdown syntax
name: markdown syntax
author: Fung Kong
datePublished: 2016-07-08 02:49:35
---
Markdown is a lightweight markup language, the original purpose of markdown is to provide a handy tool for text to html,  with easy read and easy write syntax format. Markdown can be defined to: 1. plain-text syntax; 2. a software converting the plain-text to html.
<!--more-->
When I was using wordpress as my personal blog, I didn't know which text editor should I use for blogging. After I changed my blog frame to octopress, I begin to use markdown. With handy and simple syntax, it save my a lot of time. Because of its efficiency, I write all of my plain-text documents with markdown.

## 1. Markdown Syntax
Normally, markdown line breaks are identical like other editors' line break--Enter. But in some situation, There may be different, either by two Enters, or by three spaces in the end of line.
### 1.1 Headers

```vim
# This is H1
## This is H2
### This is H3
#### This is H4
##### This is H5
###### This is H6
Alternative H1
===============
Alternative H2
----------------
```

In WEB, it will be shown as:

# This is H1
## This is H2
### This is H3
#### This is H4
##### This is H5
###### This is H6
Alternative H1
===============
Alternative H2
----------------


### 1.2 Blockquotes

```vim
William Wallace said:
> What would you do without freedom,
> would you fight?

> This is first level
> > this is nested blockquotes
```

William Wallace said:
> What would you do without freedom,   
> would you fight?
 
> This is first level
> > this is nested blockquotes

Blockquotes also can combine other syntax, such as Header, Lists etc.

```vim
> ##This is a Header
>
> 1. List item one
> 2. List item two
>
> Here's some code block example:
>     #!/bin/bash
>     echo "Hello World"
>     exit 0
```

> ##This is a Header
>
> 1. List item one
> 2. List item two
>
> Here's some code block example:
>
>     #!/bin/bash
>     echo "Hello World"
>     exit 0

### 1.3 Lists

#### 1.3.1 Unordered lists

Unordered list use hyphens or asterisks or pluses

```vim
- Unordered list item 1
- Unordered list item 2
   + Unorder nested list item 2.1
+ Unordered list item 3
```

- Unordered list item 1
- Unordered list item 2
   + Unorder nested list item 2.1
+ Unordered list item 3


#### 1.3.2 Ordered lists

Ordered lists use numbers followed by periods:

```vim
1. Ordered list item 1
2. Ordered list item 2
3. Ordered list item 3
```

1. Ordered list item 1
2. Ordered list item 2
3. Ordered list item 3


### 1.4 Code Blocks and Syntax Highlighting
Indented with four spaces will fenced code block

<pre>
Indented with four spaces
    #!/bin/bash
    echo "hello world"
</pre>

    #!/bin/bash
    echo "hello world"

Sometimes, back ticks are more efficiency. Besides, only back-ticks are supporting syntax highlighting, it's recommended to use back ticks. Below examples actually no bakslash here, just for escaping.

<pre>
\`\`\`javascript
var s = "JavaScript syntax highlighting";
alert(s);
\`\`\`

\`\`\`python
s = "Python syntax highlighting"
print s
\`\`\`

\`\`\`
No language indicated, so no syntax highlighting.
\`\`\`

Inline code block `syntax highlighting`.
</pre>

```javascript
var s = "JavaScript syntax highlighting";
alert(s);
```

```python
s = "Python syntax highlighting"
print s
```

```
No language indicated, so no syntax highlighting.
```

Inline code block `Inline code highlighting`.

### 1.5 Horizontal rules

```vim
With three or more asterisks, hyphens, or underscores:

Three or more asterisks

*******

or hyphens

---------

or underscores

___________

```

With three or more asterisks, hyphens, or underscores:

Three or more asterisks

*******

or hyphens

---------

or underscores

___________



### 1.6 Links
Markdown supports two style of line: *inline* or *reference*.

```vim
This is an **inline** [Link for my blog](http://www.kkdba.com). 

This is an **reference** [Link for my current post](/markdown-syntax.html).

Also reference [Link][markdown syntax] can be used like this way.

[markdown syntax]: http://www.kkdba.com/markdown-syntax.html.

URLs and URLs in angle brackets will automatically get turned into links.  http://www.kkdba.com or <http://www.kkdba.com>.

```

This is an **inline** [Link for my blog](http://www.kkdba.com). 

This is an **reference** [Link for my current post](/markdown-syntax.html).

Also reference [Link][markdown syntax] can be used like this way.

[markdown syntax]: http://www.kkdba.com/markdown-syntax.html.

URLs and URLs in angle brackets will automatically get turned into links: http://www.kkdba.com or <http://www.kkdba.com>.

### 1.7 Images
Images quote just like Link usages: *inline* or *reference*.

```
This is my image with inline ![Inline](http://www.kkdba.com/images/rss.png).

This is my image with reference ![Reference](/images/rss.png).
```

This is my image with inline ![Inline](http://www.kkdba.com/images/rss.png).

This is my image with reference ![Reference](/images/rss.png).


### 1.8 Emphasis
The way emphasize a word is using asterisks or underscores around the word.

```vim
Emphasis with one *asterisk*

With two __underscores__

Or with three ***asterisks***

With two ~~tildes~~ add strikethrough to the word
```

Emphasis with one *asterisk*

With two __underscores__

Or with three ***asterisks***

With two ~~tildes~~ add strikethrough to the word

### 1.9 Backslash Escapes

Markdown provides backslash escapes for the following characters:

```vim
\   backslash
`   backtick
*   asterisk
_   underscore
{}  curly braces
[]  square brackets
()  parentheses
#   hash mark
+   plus sign
-   minus sign (hyphen)
.   dot
!   exclamation mark
```

### 1.10 Tables

Tables in markdown use pipes and hyphens(Three or more) with colon to align columns, the outer pipes can be optional.

```vim
| First name(Default Left align) | Last name(Centre align) | Salary(Right align) |
| ----------                     | :---------:             | ------------------: |
| Fung                           | KK Kong                 | $3000               |
| Oracle                         | Larry                   | $20                 |
| Larry                          | Jordan                  | $300000             |
```


<center> Table 1 -- Colons can be used to align columns </center>

| First name(Default Left align) | Last name(Centre align) | Salary(Right align) |
| ----------                     | :---------:             | ------------------: |
| Fung                           | KK Kong                 | $3000               |
| Oracle                         | Larry                   | $20                 |
| Larry                          | Jordan                  | $300000             |


## 2. Markdown extension on google chrome

Google Chrome extension [Markdown Preview Plus](https://chrome.google.com/webstore/detail/markdown-preview-plus/febilkbfcbhebfnokafefeacimjdckgl) is for previewing markdown plain-text file via chrome browser, it also can convert the markdown plain-text file to mhtml file or PDF file.

### Configuration for markdown preview
This extension supports customizing css style, I choose this one [zhangjikai/markdown-css](https://github.com/zhangjikai/markdown-css).


When using export tool by default, the html file is mhtml file format, in my laptop environment, I can't open it with firefox browser, so I use this tool [mht2htmcl](https://sourceforge.net/projects/mht2htm/?source=typ_redirect) (or you can [download](/download/mht2htmcl) from my blog) to convert mht file to html file, after converting, it works perfectly for me.




***EOF***

