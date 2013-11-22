---
layout: post
title: "过去和现在"
date: 2012-03-10 11:47
comments: true
categories: [blog, octopress, jekyll, github, markdown]
---

## 过去的选择

- 喜欢 [Dotclear](http://dotclear.org/) 胜过 [WordPress](http://wordpress.org/)
- 喜欢结构化文档的理念
  - 开源文档世界中的 [DocBook](http://docbook.org/) 虽好，但 XML 书写繁琐
  - 轻量级的文本标记语言和各种 Wiki 标记语言虽然简单，但语义表达能力弱，但写 BLOG 等一般的短文章也足够用了
    - 轻量级的文本标记语言： [markdown](http://daringfireball.net/projects/markdown/)、 [reST](http://docutils.sourceforge.net/rst.html)、  [txt2tags](http://txt2tags.sf.net) ...
      - [Markdown Wiki](http://markdown.infogami.com/)
	  - [Markdown 和多种标记语言的在线转换](http://johnmacfarlane.net/pandoc/try)
	  - [Markdown 多种实现的在线比较](http://babelmark.bobtfish.net/)
    - 各种 Wiki 标记语言： Creole、 dokuwiki、 moinmoin、 Mediawiki ...
	  - [各种 Wiki 系统（包括标记语言）的比较](http://www.wikimatrix.org/)
  - [AsciiDoc](http://www.methods.co.nz/asciidoc/)：折中的选则，架起了文本标记语言和 DocBook 的桥梁，基本实现了 DocBook 的语义表达
- 逐渐更喜欢不带关系数据库的个人站点应用，因为备份简单（`rsync`）
  - 最初使用 [txt2tags](http://txt2tags.sf.net) -- 仅实现了txt2tags语法到html的转换，缺乏版本控制
  - 后转向 [Dokuwiki](http://dokuwiki.org) -- dokuwiki语法+内置的版本控制
  - 曾经想切换到 [ikiwiki](http://ikiwiki.info/) -- MarkDown语法+git版本控制

## 痛苦的过去

- 曾经注册过自己的域名 
- 曾经在国内和国外购买过虚拟主机空间


> **无奈皆因家中琐事繁多，无暇打理，导致域名被抢注、主机空间因未续费而被停用**


## 如今的选择


- 使用 [github](http://github.com) 免费的二级域名，免去因忘记域名续费而被抢注的困扰
- 使用 [github](http://github.com) 提供的 300M 免费空间，对于一般情况也足够用了，免去因忘记空间续费而被停用的困扰


- 使用 git 的版本控制替代 wiki 内置的版本控制，在 github 上 **BLOG 即 REPO**
- 使用 markdown 语法替代以前的 DokuWiki 语法书写文档
- 在 github 上创建个人站点的工具主要是 [Octopress](http://octopress.org/) 和 [Jekyll-Bootstrap](http://jekyllbootstrap.com/)，因后者似乎还不太成熟遂选择前者
