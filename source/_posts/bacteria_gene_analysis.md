---
title: 细菌基因生物信息学分析通用教程（从零到完整流程）
date: 2026-03-28
  - 生物信息学
  - 细菌基因分析
categories: 
  - 生信分析
  - 细菌
  - 基因分析
author: Mice33
description: 详细介绍一套通用的细菌基因生物信息学分析流程教程
keywords: 生物信息学, 数据分析
top: false
---

# 一、写在前面

这是一套**通用的细菌基因生物信息学分析流程教程**

特点：

✔ 工具简单（基本网页 + 少量命令行）  
✔ 可复现  
✔ 结构完整  

# 二、整体流程（核心框架）

```
基因序列
↓
蛋白翻译
↓
理化性质分析
↓
亚细胞定位
↓
结构域分析
↓
三级结构预测
↓
系统发育分析
↓
调控与功能分析
↓
密码子偏好分析
```

👉 记住一句话：

> 结构 + 定位 + 进化 + 调控 = 完整功能推断

---

# 三、操作

## 步骤1：获取基因与蛋白序列

来源：

- NCBI Genome / RefSeq
- 本地测序数据

常见文件：

```bash
.fna   # 基因组
.faa   # 蛋白
.ffn   # CDS
```

👉 目标：拿到

- CDS序列
- 蛋白序列
- 基因组

### 基本特征分析：

[EMBOSS: cusp](https://bioinformatics.nl/cgi-bin/emboss/cusp)

输入CDS 序列进行密码子使用分析，得到：

- 基因全长

| 长度        | 一般含义             |
| ----------- | -------------------- |
| <300 bp     | 小肽/调控蛋白        |
| 300–1500 bp | 常规酶蛋白（最常见） |
| >2000 bp    | 多结构域蛋白         |

- 编码AA数目

| 长度       | 结构特点 |
| ---------- | -------- |
| <100 aa    | 小蛋白   |
| 100–400 aa | 单结构域 |
| >400 aa    | 多结构域 |

- GC含量--> 与翻译效率和mRNA稳定性有关

| 类型    | GC含量 |
| ------- | ------ |
| AT-rich | <40%   |
| 平衡型  | 40–60% |
| GC-rich | >60%   |

如果某基因 GC ≠ 基因组平均值

可能说明：

> **水平基因转移（HGT）**

## 步骤2：蛋白理化性质分析

工具：ProtParam[Expasy - ProtParam](https://web.expasy.org/protparam/)

输入：蛋白序列

输出：

- 分子量
- pI
- GRAVY
- 稳定性

👉 作用：

> 判断蛋白是稳定/亲水/膜蛋白趋势

工具：ProtScale[Expasy - ProtScale](https://web.expasy.org/protscale/)

### 步骤3：亚细胞定位

工具：PSORTb[PSORTb Subcellular Localization Prediction Tool - version 3.0](https://psort.org/psortb/)

关注：

- Cytoplasmic
- Membrane
- Extracellular

👉 关键判断：

- 是否分泌蛋白

### 步骤4：结构域分析

工具：

- NCBI CDD[CD-Search: New Query](https://www.ncbi.nlm.nih.gov/Structure/cdd/wrpsb.cgi?SEQUENCE)
- Pfam / InterPro[InterPro](https://www.ebi.ac.uk/interpro/)

关注：

- E-value < 1e-5

👉 核心意义：

> 决定蛋白“干什么”

### 步骤5：三级结构预测

工具：

- AlphaFold / ColabFold

输出：

- PDB / CIF

验证：

- PROCHECK（Ramachandran）[PDBsum Generate](https://www.ebi.ac.uk/thornton-srv/databases/pdbsum/Generate.html)

👉 判定标准：

- 最优区 ≥ 90%（理想）

| 类型          | 工具         | 本质           |
| ------------- | ------------ | -------------- |
| AlphaFold指标 | pLDDT / PAE  | AI预测置信度   |
| PROCHECK      | Ramachandran | 几何结构合理性 |

注：现在的alphafold3输出的各种指标就足够判断了，不一定非要做PROCHECK

### 步骤6：系统发育分析

流程：

```bash
BLASTp → MAFFT → TrimAl → IQ-TREE
```

BLASTp找同源蛋白，根据亲缘关系远近选出合适的序列，使用MAFFT软件（--auto参数）进行多序列比对，并利用TrimAl（-gt 0.8）去除低质量比对区域。基于修剪后的序列，采用IQ-TREE[iTOL: Interactive Tree Of Life](https://itol.embl.de/)构建最大似然系统发育树，并利用ModelFinder自动选择最优替换模型，进行1000次ultrafast bootstrap重复评估分支支持率。

意义：

- 判断进化关系
- 判断是否保守蛋白

### 步骤7：启动子与调控元件

工具：MEME[MEME - Submission form](https://meme-suite.org/meme/tools/meme)

对 基因上游 200 bp 候选启动子区域进行 MEME 分析

> 在 DNA 序列中寻找**潜在的转录因子结合位点（motif）**

参数：

- nmotifs = 5

​            最多找几个“调控信号”

- minw = 6

​             最短 motif = 6 bp，太短假阳多

- maxw = 20

​             最长 motif = 20 bp，太长没有生物学意义

注意：

- 单序列结果参考意义有限

### 步骤8：蛋白互作网络

工具：STRING[STRING: functional protein association networks](https://string-db.org/)

参数：

- 物种指定
- score ≥ 0.7（推荐）

👉 看：

- 代谢通路
- 毒力相关蛋白

### 步骤9：信号肽预测

工具：SignalP 6.0[SignalP 6.0 - DTU Health Tech - Bioinformatic Services](https://services.healthtech.dtu.dk/services/SignalP-6.0/)

👉 输出：

- 是否分泌
- 切割位点

### 步骤10：翻译后修饰

工具：

- NetPhos（磷酸化）[NetPhos 3.1 - DTU Health Tech - Bioinformatic Services](https://services.healthtech.dtu.dk/services/NetPhos-3.1/)

注意：

- 细菌没有O-glycosylation预测工具（常规的如NetOGlyc 主要适用于真核蛋白 O-GalNAc 糖基化预测，与细菌蛋白修饰机制差异较大，不用）

### 步骤11：密码子偏好性

工具：CodonW

指标：

- 有效密码子数（ENC）
- 相对同义密码子使用度（RSCU）
- 密码子适应指数（CAI）
- 最优密码子使用频率（Fop）
- 密码子偏好指数（CBI）
- 基因整体GC含量
- 第三位密码子GC含量（GC3s）

👉 判断：

- 表达水平
- 偏好来源（突变 vs 选择）

## 关于sRNA调控

- IntaRNA
- sRNA互作

实际：

👉 前提是：

- 有已注释 sRNA

如果没有：可以先不做

## 最终如何整合结果

不要只是堆工具，要形成逻辑：

```
结构域 → 功能
定位 → 作用位置
系统树 → 保守性
修饰 → 调控
密码子 → 表达
```

👉 最终输出：

> 一个“功能模型”

