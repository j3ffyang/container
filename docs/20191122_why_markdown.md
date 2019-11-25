<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>

**<center><font size=48>Why Markdown?</font></center>**

<br>
<br>
<br>

<center><img src="../imgs/20191125_markdown_logo.svg" width="400"></center>

<br>
<br>
<br>
<br>
<br>

<div class="pagebreak"></div>

<br>
<br>

<font size=5>**Content**</font>

<!-- TOC depthFrom:2 depthTo:4 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Introduction](#introduction)
- [Standard, Simple and General](#standard-simple-and-general)
- [Plugins and Examples](#plugins-and-examples)
		- [Font control](#font-control)
		- [Image](#image)
		- [Check-list](#check-list)
		- [Table](#table)
		- [Pagebreak](#pagebreak)
		- [mscGen - message sequence chart](#mscgen-message-sequence-chart)
		- [LaTeX](#latex)
		- [Tutorial](#tutorial)

<!-- /TOC -->

<div class="pagebreak"></div>

## Introduction

We do have lots of chances to write documents and guideline books in our business. I've tried many editors before and there is no doubt that Markdown is one of the simplest and standard format to make our life relaxed. I want to use this document to prove how easily Markdown generates beautiful documentation.

## Standard, Simple and General
- Managing everything in code
- WYSIWYG
- Used for everything: document, book, presentation (marp, https://marp.app/), email and web
- OS independent
- Portable, like at Github, Stackoverflow, and Reddit etc
- Available for collobration work at Gitbook

## Plugins and Examples

- Github integrated, vim enabled and python lib, etc, like an IDE
- PDF convertion
- Modern book format and toolchain using Git and Markdown
	> https://github.com/GitbookIO/gitbook
	> http://www.chengweiyang.cn/gitbook/introduction/README.html
- Table-of-Content generated on the fly

#### Font control
> Number of space for indent matters

- <font size=3 color=darkorange>Font well-controlled</font>

&nbsp;&nbsp;&nbsp;Code:
```shell
   <font size=3 color=darkorange>Font well-controlled</font>
```

- Comment out

<!--
	You won't see this
-->

&nbsp;&nbsp;&nbsp;Code:
```shell
   <!--
     You won't see this
   -->
```

- ~~Strike-through what you don't want~~

&nbsp;&nbsp;&nbsp;Code:
```shell
    ~~Strike-through what you don't want~~
```

- __Bold__ and _Italic_

&nbsp;&nbsp;&nbsp;Code:
```shell
   __Bold__ and _Italic_
```


#### Image

<td>
	<div class=”no-break”>
		<center><img src="../imgs/20190729_docker_logo.png" alt="Docker"
		title="We love container" width="550"></center>
		![Docker](../imgs/20190729_docker_logo.png)
	</div>
</td>

<div class=”no-break”>
![Docker](../imgs/20190729_docker_logo.png)
</div>

#### Check-list

   - [x] Write the press release
   - [ ] Update the website
   - [ ] Contact the media

<br>

&nbsp;&nbsp;&nbsp;Code:

```shell
   - [x] Write the press release
   - [ ] Update the website
   - [ ] Contact the media
```


#### Table

  item | content | location
  -- | -- | --
  Monday Morning | We have a conference | Office
  Friday Afternoon | We go for drink :-D | A bar outside of office

&nbsp;&nbsp;&nbsp;Code:
```shell
   item | content | location
   -- | -- | --
   Monday Morning | We have a conference | Office
   Friday Afternoon | We go for drink :-D | A bar outside of office
```

#### Pagebreak

```bash
   <div class="pagebreak"></div>
```
#### mscGen - message sequence chart

> Reference > http://www.mcternan.me.uk/mscgen/ and https://mscgen.js.org/

```bash
# fictional process

msc {
  hscale = "2";

  a,b,c;

  b->c [ label = "bc(TRUE)"];
  c=>c [ label = "process(1)" ];
  c=>c [ label = "process(2)" ];
  ...;
  c=>c [ label = "process(n)" ];
  c=>c [ label = "process(END)" ];
  a<<=c [ label = "callback()"];
  ---  [ label = "If more to run", ID="*" ];
  a->a [ label = "next()"];
  a->c [ label = "ac1()\nac2()"];
  b<-c [ label = "cb(TRUE)"];
  b->b [ label = "stalled(...)"];
  a<-b [ label = "ab() = FALSE"];
}
```

#### LaTeX

Lay-tech, a document preparation system for high-quality typesetting

$$
\sum_{n = 0}^{\infty}\frac{210,000*50}{2^n} = 210,000*50*\frac{1}{1-\frac{1}{2}} = 21,000,000
$$

&nbsp;&nbsp;&nbsp;Code:
```shell
$$
\sum_{n = 0}^{\infty}\frac{210,000*50}{2^n} = 210,000*50*\frac{1}{1-\frac{1}{2}} = 21,000,000
$$
```


#### Tutorial

> https://markdown-guide.readthedocs.io/en/latest/basics.html
> https://commonmark.org/help/tutorial/
