---
{"dg-publish":true,"permalink":"/🍀 永久笔记/效率工具/git/git 怎么用/","created":"2023/03/06 13:49:32","updated":"2023/03/07 13:17:34"}
---


最简单的用法：

- `git add .`
- `git commit -m "update"`
- `git push`

升级版的用法：

- `git init` 初始化仓库
- `git remote add origin git@github.com/Gonglja/test.git` 与 github 远端仓库关联起来
- `git add xxx` 添加指定文件
- `git commit -m " "` `" "` 中填写要提交的内容，在输入单引号后直接回车可以输入多行
- `git push -set-upstream origin master` 设置远程仓库分支为本地分支的上游分支，并推送

submodule 子仓库的用法

- TODO

完整用法详见 [[🍀 永久笔记/效率工具/git/Pro Git\|Pro Git]]