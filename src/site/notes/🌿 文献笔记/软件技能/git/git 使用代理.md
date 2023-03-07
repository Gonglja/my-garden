---
{"dg-publish":true,"permalink":"/🌿 文献笔记/软件技能/git/git 使用代理/","created":"2023/03/04 21:22:16","updated":"2023/03/07 13:15:42"}
---


配合 [v2raya](https://v2raya.org/) 使用，效果杠杠滴。

设置全局代理

```shell
git config --global http.proxy http://127.0.0.1:20172
git config --global https.proxy http://127.0.0.1:20172
```

关闭全局代理

```shell
git config --global --unset http.proxy
git config --global --unset https.proxy
```