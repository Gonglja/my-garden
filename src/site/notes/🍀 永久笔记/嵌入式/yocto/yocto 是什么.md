---
{"dg-publish":true,"permalink":"/🍀 永久笔记/嵌入式/yocto/yocto 是什么/"}
---


yocto 是国际单位中指定的测量单位的最小部分，在本文中[yocto](https://www.yoctoproject.org/) 是一个[[构建工具\|构建工具]]，帮助开发人员快速定制化 linux 项目，而不需要关心其底层架构。

- 特点
	- 社区活跃
	- 灵活
	- 支持架构种类多
	- 适合嵌入式设备
	- 使用分层模型
	- 支持局部构建，可以单独构建某个「包，package」或者「层，Layer」
	- 发布计划严格
	- 生态系统丰富
- 挑战
	- 学习曲线陡峭
	- 项目工作流程不清晰
	- 第一次构建时间久
	- 需要科学上网
	- 交叉编译

[[🍀 永久笔记/嵌入式/yocto/yocto\|yocto]] 与 [[OpenEmbedded\|OpenEmbedded]]的关系
- 相辅相成
- OE 维护 bitbake 和一个元数据层
- YP 侧重构建系统本身和针对交叉编译工具化
- Poky 是 Yocto 的一个开发样例
![1949124-20210208230117888-1024529922.png](/img/user/Resources/Images/1949124-20210208230117888-1024529922.png)