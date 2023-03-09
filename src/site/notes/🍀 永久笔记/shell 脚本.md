---
{"dg-publish":true,"permalink":"/🍀 永久笔记/shell 脚本/"}
---


本处存放一些常用的脚本

1. 查找当前路径下所有文件并执行

```shell
#!/bin/bash
for i in `ls`
do
	echo run $i; ./$i
done

```