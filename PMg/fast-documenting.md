# A fast and interactive documentation 

This article originates from a simple question : What is the easiest way to publish content over the Internet, so that it can be shared with people involved in the development and the final use of the product ?

Here is a strategy that uses Typora, Github and MkDocs to produce an interactive documentation that can be deployed within minutes:

Create a git repository, fill it with markdown-formatted files (.md) following the below structure : 

- directory tree of markdown formatted files
- each directory contains one or more .md file
- to avoid troubles, avoid filenames with chars other than a-z0-9 and dash (-)



From there, anyone with a git client (having an access to the repository) can clone, pull and push updates through traditional git commands : 
```
git clone https://github.com/{username}/{repository}.git
git commit -a -m"update"
git push
git pull
...
```



To create or edit files, any text editor would do the trick. But for those who enjoy a little WYSIWYG, there are some great tools available for free : [Typora](https://typora.io/) is one of them.

The nice thing with markdown is that you immediately get a nicely formatted document with headers hierarchy and related table of content, with very little formatting syntax : just use [Adam Pritchard cheat sheet](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) as a memo.



At last, to have a doc accessible through the internet and with search capabilities, you can take advantage of a [docker image](https://hub.docker.com/r/polinux/mkdocs/) that offers an out-of-the box version of [MkDocs](https://www.mkdocs.org/).

