---
{"dg-publish":true,"permalink":"/🌿 文献笔记/软件技能/tensorflow 中 h5py 报错/"}
---


具体错误：UserWarning: h5py is running against HDF5 1.10.5 when it was built against 1.10.4

错误原因：h5py 版本不一致

解决方案：

> [!note]- conda 命令
> 
<div class="transclusion internal-embed is-loaded"><a class="markdown-embed-link" href="///conda/conda/#" aria-label="Open link"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="svg-icon lucide-link"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></a><div class="markdown-embed">



## 切换环境

在 `conda` 终端中，

- `conda info -e` 查看 conda 有什么环境，`*` 表示当前环境
- `conda activate $环境名` 激活环境（切换环境）
- `conda deactivate` 取消环境

</div></div>


- 切换 [[🌿 文献笔记/软件技能/conda/conda\|conda]] 环境为当前环境
- 先卸载 h5py 
- 在安装 h5py

```shell
conda activate newtf

pip uninstall h5py

pip install h5py
```