---
{"dg-publish":true,"permalink":"/🌿 文献笔记/obsidian 插件 digital garden 使用/"}
---


Digital Garden 是一个类似 Obsidian 官方 Publish 的工具，教程详细，构建方便。

- 官方地址：https://github.com/oleeskild/obsidian-digital-garden
- 当前版本：[2.37.0](https://github.com/oleeskild/obsidian-digital-garden/tree/2.37.0)
- 插件地址：[Digital Garden](obsidian://show-plugin?id=digitalgarden)
- 教程：[https://dg-docs.ole.dev](https://dg-docs.ole.dev)
- 插件支持特性：[https://dg-docs.ole.dev/features/](https://dg-docs.ole.dev/features/)

将插件配置、自动化部署配置完成后，基本上日常只需要点两下，就可以自动完成文章的新增，更新，部署，发布。

在使用过程中遇到的问题

- 目前 2023-03-07 存在的问题：
	- 名称中不许包含 `+`，不会生成 web 页面
	- 图片名称不能包含 ` ` 空格，否则图片在 web 只会显示其名称
	- 时间显示有问题
		- ![Pasted_image_20230307151438.png](/img/user/Resources/Images/Pasted_image_20230307151438.png)
	- 文档中使用语法 `![[]]` 嵌入 excalidraw 图时，excalidraw 名为中文不显示，切成英文就好了，[详情](https://github.com/Gonglja/my-garden/commit/eb905b02a9cf4142105b5754e4cc4a37ab9ae3c4)
