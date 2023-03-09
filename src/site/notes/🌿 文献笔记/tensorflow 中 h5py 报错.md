---
{"dg-publish":true,"permalink":"/🌿 文献笔记/tensorflow 中 h5py 报错/"}
---


具体错误：UserWarning: h5py is running against HDF5 1.10.5 when it was built against 1.10.4

错误原因：h5py 版本不一致

解决方案：

> [!note]- conda 命令
>  [[🌿 文献笔记/conda 怎么用#切换环境\|conda 怎么用#切换环境]]

- 切换 [[🌿 文献笔记/conda\|conda]] 环境为当前环境
- 先卸载 h5py 
- 在安装 h5py

```shell
conda activate newtf

pip uninstall h5py

pip install h5py
```