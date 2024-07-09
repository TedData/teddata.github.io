[Ted Yu](https://teddata.github.io/)
================================

---

Firstly, the title must be in the format yyyy-mm-dd-PascalCaseTitle.md.

For example:
```
2015-02-14-HelloWorld.md
```

---

The 1st line is: --- # This format cannot be changed.

The 2nd line is: layout: post # This format cannot be changed.

The 3rd line is: title: "How to write a post" # Title

The 4th line is: subtitle: "Using markdown to make a post" # Subtitle (optional)

The 5th line is: date: yyyy-mm-dd # This format cannot be changed.

The 6th line is: author: "Ted" # This format cannot be changed.

The 7th line is: header-style: text # This format cannot be changed.

The 8th line is: tags:
  - key 1
  - key 2
  ... # Replace "key" with relevant keywords

The last line is: --- # This format cannot be changed.

Then, leave a blank line, and after that, you can start writing content in markdown format.

For example:
```
---
layout: post
title: "How to write a post"
subtitle: 'Using markdown to make a post'
date: 2013-10-15
author: "Ted"
header-style: text  /  header-img: "img/post-bg-infinity.jpg"
header-mask: 0.3
mathjax: true
tags:
  - markdown
  - python
---
```

---

\> Followed by content, which can be formatted using markdown.

For example:
```
> The type is very important!!!
```

---

When adding image addresses, you must use URLs and not local addresses. 

For example:
```
![](https://raw.githubusercontent.com/TedData/TedData.github.io/master/img/icon_wechat.png)
```

---

Add code, using "```" followed by the programming language name.

For example:
```
\```python
# Not having \
print("Hello, World!")
\```

    Hello, World!
```

```python
# Your Python code goes here
print("Hello, World!")
```

    Hello, World!
