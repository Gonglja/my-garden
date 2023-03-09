---
{"dg-publish":true,"permalink":"/🍀 永久笔记/yocto 命令行/"}
---


==核心==

`bitbake --help`

列出所有包 

`bitbake -s`

查询指定包的配方,XXX 是包名

`bitbake -e XXX`

查看 xxx 模块源码的路径

`bitbake -e xxx | grep
{ #S}
=``

查看 xxx 配方的路径

`bitbake -e xxx | grep '^FILE='``

找配方源文件,\*core\*.bb 是包名，具体替换成你自己要搜索的

`find . -name "*core*.bb"`

查看包支持的方法,XXX 是包名

`bitbake XXX -c listtasks`

把所有包拉下来不编译,XXX 是包名

`bitbake XXX -c fetchall`