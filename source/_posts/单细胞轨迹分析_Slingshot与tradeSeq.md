---
title: 单细胞轨迹分析实战：Slingshot + tradeSeq 单核细胞拟时序分析流程
date: 2026-03-26 13:35:00
updated: 2026-03-26 13:35:00
tags:
  - 单细胞
  - Seurat
  - Slingshot
  - tradeSeq
  - 拟时序分析
  - Monocyte
categories:
  - 生信分析
  - 单细胞转录组
  - 轨迹分析
author: mice33
description: 结合 CD14+ Monocytes 与 CD16+ Monocytes 的实际分析流程，系统梳理从 Seurat 对象出发，使用 Slingshot 构建轨迹并结合 tradeSeq 挖掘动态基因的完整拟时序分析方法。
keywords: Slingshot, tradeSeq, Seurat, 单细胞轨迹分析, 单核细胞, 拟时序
top: false
cover:
---

# 🧭 单细胞轨迹分析实战：Slingshot + tradeSeq 单核细胞拟时序分析流程

在单细胞转录组分析中，聚类和注释回答的是“**这些细胞是谁**”，而轨迹分析更关心“**这些细胞可能如何发生连续变化**”。

对于单核细胞、巨噬细胞、T 细胞活化状态这类本身具有连续过渡特征的群体，仅仅依赖离散聚类往往是不够的。此时，拟时序分析（pseudotime analysis）可以帮助我们进一步回答：

- 某一类细胞可能沿着怎样的方向发生状态转换？
- 哪些基因会随着状态推进而动态变化？
- 哪些基因更接近“起点标记基因”或“终点标记基因”？
- 感兴趣基因在轨迹中的表达趋势如何？

本文结合我自己的实际分析流程，整理一套从 **Seurat 对象出发**，使用 **Slingshot** 构建轨迹，并结合 **tradeSeq** 进行拟时序差异基因分析的完整方法。

<!-- more -->

## 📚 分析目标与适用场景

本文示例细胞群为：

- `CD14+ Monocytes`
- `CD16+ Monocytes`

分析目标是探索这两类单核细胞之间是否存在连续状态变化，并进一步观察基因在轨迹中的表达趋势。

这套流程尤其适合下面几类问题：

- 某个细胞亚群之间具有明显连续过渡，而不是完全离散分开
- 想从聚类分析进一步走向“状态转换”解释
- 想筛选沿轨迹动态变化的候选基因
- 想验证特定目标基因是否和拟时序推进相关

---

## 1️⃣ 分析思路总览

整个流程可以概括为：

```text
Seurat对象
   ↓
筛选目标细胞亚群
   ↓
提取表达矩阵 + UMAP + metadata
   ↓
构建 SingleCellExperiment 对象
   ↓
Slingshot 轨迹推断
   ↓
拟时序可视化
   ↓
tradeSeq 拟合 NB-GAM 模型
   ↓
识别动态基因 / 起止点相关基因
   ↓
绘制感兴趣基因的拟时序表达图
```

如果你已经完成了 Seurat 的标准预处理、降维与细胞注释，那么这篇文章的起点就是：

> 从已有的 Seurat 对象中，进一步抽取目标细胞群，做拟时序分析。

---

## 2️⃣ 所需 R 包

```r
library(Seurat)
library(SingleCellExperiment)
library(tidyverse)
library(RColorBrewer)
library(ggplot2)
library(ggsci)
library(ggpubr)
library(slingshot)
library(tradeSeq)
```

如果尚未安装，可按需执行：

```r
# install.packages(c("tidyverse", "RColorBrewer", "ggplot2", "ggsci", "ggpubr"))
# BiocManager::install("slingshot")
# BiocManager::install(c("edgeR", "tradeSeq"))
```

---

## 3️⃣ 读取 Seurat 对象并筛选目标细胞

这里默认输入是一个已经完成预处理、降维和细胞注释的 Seurat 对象：

```r
data <- qread("output/data.qs")

scRNAsub <- subset(
  data,
  subset = cell_type %in% c("CD14+ Monocytes", "CD16+ Monocytes")
)
```

先简单查看两个群体的分布：

```r
table(scRNAsub$cell_type)
DimPlot(scRNAsub, group.by = "cell_type")
```

### 🔍 这一步主要看什么

- 细胞数是否足够
- UMAP 上是否存在一定连续过渡
- 是否适合做轨迹分析

> ⚠️ 如果两个细胞群在 UMAP 上完全分离、看不到任何连续性，那么强行做轨迹分析通常意义不大。

---

## 4️⃣ 从 Seurat 转为 SingleCellExperiment

`slingshot` 使用的是 `SingleCellExperiment` 对象，因此需要把表达矩阵、降维结果和元信息从 Seurat 中提取出来。

### 4.1 提取表达矩阵

这里直接从 `scale.data` 中提取已经进入标准化和缩放的基因，再手动保留自己关心的基因，比如 `S100A9`：

```r
scale.data <- scRNAsub@assays$RNA@scale.data
scale.gene <- rownames(scale.data) ##高变基因
"S100A9" %in% scale.gene

genes_keep <- c("S100A9")
genes_keep <- c(genes_keep, scale.gene)

counts <- scRNAsub@assays$RNA@counts
counts <- counts[genes_keep, ]
```

### 4.2 构建 SCE 对象

```r
sim <- SingleCellExperiment(assays = List(counts = counts))
```

这里要特别注意一点：

> `tradeSeq` 下游拟合使用的是 **counts 矩阵**，而不是单纯的标准化表达矩阵。

---

## 5️⃣ 添加 UMAP 与 metadata 信息

### 5.1 添加降维结果

```r
umap <- scRNAsub@reductions[["umap_harmony"]]@cell.embeddings
colnames(umap) <- c("UMAP-1", "UMAP-2")

reducedDims(sim) <- SimpleList(UMAP = umap)
```

### 5.2 添加元数据

```r
meta <- scRNAsub@meta.data

colData(sim)$sampleId <- meta$sample_id
colData(sim)$celltype <- meta$cell_type
```

简单看一下二维分布：

```r
rd <- umap
plot(rd, col = rgb(0, 0, 0, .5), pch = 16, asp = 1)
```

到这里，`sim` 已经具备了：

- counts 表达矩阵
- UMAP 坐标
- 细胞类型信息
- 样本信息

这就是进入 `slingshot` 分析的基本输入。

---

## 6️⃣ 使用 Slingshot 构建轨迹

最直接的写法如下：

```r
sim <- slingshot(
  sim,
  clusterLabels = "celltype",
  reducedDim = "UMAP",
  start.clus = "CD14+ Monocytes",
  end.clus = NULL
)
```

### 参数解释

- `clusterLabels = "celltype"`：用细胞类型注释作为聚类标签
- `reducedDim = "UMAP"`：基于 UMAP 空间拟合轨迹
- `start.clus = "CD14+ Monocytes"`：人为指定起点
- `end.clus = NULL`：不预设终点，让算法自己判断

查看新增列：

```r
colnames(colData(sim))
head(colData(sim)[, 4:5])
```

通常这里会新增：

- `slingPseudotime_1`
- `slingCurveWeights_1`

---

## 7️⃣ 轨迹可视化

### 7.1 绘制整体轨迹线

```r
plot(reducedDims(sim)$UMAP, pch = 16, asp = 1)
lines(SlingshotDataSet(sim), lwd = 2, col = brewer.pal(9, "Set1"))
legend(
  "right",
  legend = paste0("lineage", 1:1),
  col = unique(brewer.pal(6, "Set1")),
  inset = 0.8,
  pch = 16
)
```

### 7.2 按拟时序着色

为了更直观看轨迹推进方向，可以使用渐变色：

```r
summary(sim$slingPseudotime_1)

colors <- colorRampPalette(brewer.pal(11, "Spectral")[-6])(100)
plotcol <- colors[cut(sim$slingPseudotime_1, breaks = 100)]
plotcol[is.na(plotcol)] <- "lightgrey"

plot(reducedDims(sim)$UMAP, col = plotcol, pch = 16, asp = 1)
lines(SlingshotDataSet(sim), lwd = 2, col = brewer.pal(9, "Set1"))
legend(
  "right",
  legend = paste0("lineage", 1:1),
  col = unique(brewer.pal(6, "Set1")),
  inset = 0.8,
  pch = 16
)
```

### 7.3 这一步怎么看

这里的逻辑是：

- 轨迹上的细胞按 pseudotime 分成 100 个区间
- 每个区间赋予一个渐变颜色
- 不属于当前 lineage 的细胞标成灰色

这样可以很直观地看到：

> 拟时序是如何从起点逐渐过渡到终点的。

---

## 8️⃣ 为什么有时需要重新聚类再做轨迹

如果你的细胞成分比较复杂，直接用已有注释跑 `slingshot`，很容易出现下面这些问题：

- 多条轨迹严重重叠
- 轨迹方向不符合生物学常识
- 某些杂合细胞类型把轨迹“拉弯”

这也是原始流程里一个非常实用的经验：

> 用于轨迹分析的分群，未必越细越好。太细反而容易让谱系拟合变得混乱。

因此在进入 `tradeSeq` 前，经常会针对某一条 lineages 再做一次子集筛选和重新聚类。

---

## 9️⃣ tradeSeq 下游分析：筛选轨迹相关基因

`tradeSeq` 的核心思想可以概括为：

> 用负二项式广义加性模型（NB-GAM）拟合基因表达随拟时序变化的趋势。

但在正式建模前，一般会先做细胞筛选与降采样。

### 9.1 提取某一条 lineage 对应的细胞

```r
coldata <- colData(sim)
coldata <- data.frame(
  celltype = coldata@listData$celltype,
  sampleId = coldata@listData$sampleId,
  plotcol = plotcol
)
rownames(coldata) <- sim@colData@rownames

filter_cell <- dplyr::filter(coldata, plotcol != "lightgrey")
filter_cell <- rownames(filter_cell)

counts <- sim@assays@data@listData$counts
filter_counts <- counts[, filter_cell]
```

### 9.2 随机抽样细胞以降低计算负担

原始流程中也特别强调：`tradeSeq` 运行比较慢，建议先随机抽取约 2000~3000 个细胞，通常不会明显影响结果稳定性。

```r
set.seed(111)
scell <- sample(colnames(filter_counts), size = 2000)

filter_counts <- filter_counts[, scell]
dim(filter_counts)
```

### 9.3 重建过滤后的 SCE 对象

```r
filter_sim <- SingleCellExperiment(assays = List(counts = filter_counts))

filter_coldata <- colData(sim)[scell, 1:3]
filter_sim@colData <- filter_coldata

rd <- reducedDim(sim)
filter_rd <- rd[scell, ]
reducedDims(filter_sim) <- SimpleList(UMAP = filter_rd)
```

---

## 🔟 使用 K-means 重新构建更干净的轨迹

如果当前这批细胞仍然过于复杂，可以尝试在 UMAP 空间做一次 K-means 重新分群：

```r
set.seed(111)
cl <- kmeans(filter_rd, centers = 4)$cluster
colData(filter_sim)$kmeans <- cl
```

可视化 K-means 结果：

```r
mycolors <- brewer.pal(4, "Set1")

plt_k <- data.frame(
  filter_rd,
  kmeans_clusters = factor(cl, levels = sort(unique(cl)))
)

ggplot(plt_k, aes(UMAP.1, UMAP.2)) +
  geom_point(aes(color = kmeans_clusters), size = .5) +
  scale_color_manual(values = mycolors) +
  theme_bw() +
  theme(
    panel.grid = element_blank(),
    axis.text = element_blank(),
    axis.ticks = element_blank()
  ) +
  xlab("UMAP_1") +
  ylab("UMAP_2") +
  guides(color = guide_legend(override.aes = list(size = 5)))
```

然后重新跑一轮轨迹推断：

```r
filter_sim <- slingshot(
  filter_sim,
  clusterLabels = "kmeans",
  reducedDim = "UMAP",
  start.clus = "2",
  end.clus = "1"
)

plot(reducedDims(filter_sim)$UMAP, pch = 16, asp = 1)
lines(SlingshotDataSet(filter_sim), lwd = 2, col = mycolors)
```

如果觉得 K-means 不合适，也可以保留原始 `celltype` 再试一版：

```r
filter_sim <- slingshot(
  filter_sim,
  clusterLabels = "celltype",
  reducedDim = "UMAP",
  start.clus = "CD14+ Monocytes",
  end.clus = NULL
)
```

### 💡 这一步的核心原则

这里没有绝对标准答案，最重要的是：

> 轨迹结果应尽量符合生物学常识，而不是机械地套算法。

---

## 1️⃣1️⃣ 使用 tradeSeq 拟合 NB-GAM

### 11.1 准备 pseudotime 和 cellWeights

```r
counts <- filter_sim@assays@data$counts
crv <- SlingshotDataSet(filter_sim)

set.seed(111)
pseudotime <- slingPseudotime(crv, na = FALSE)
cellWeights <- slingCurveWeights(crv)
```

### 11.2 先用 evaluateK 选择 knots 数量

```r
set.seed(111)
icMat <- evaluateK(
  counts = counts,
  sds = crv,
  k = 3:10,
  nGenes = 500,
  verbose = TRUE
)
```

实践中常常根据图形拐点来选择 `nknots`。

在这套流程里，最终采用的是：

```r
nknots = 6
```

### 11.3 拟合模型

```r
system.time({
  sce <- fitGAM(
    counts = counts,
    pseudotime = pseudotime,
    cellWeights = cellWeights,
    nknots = 6,
    verbose = FALSE
  )
})

table(rowData(sce)$tradeSeq$converged)
```

这里 `converged` 很重要：

- `TRUE`：该基因模型收敛
- `FALSE`：该基因模型没有完全收敛

> ⚠️ `FALSE` 不代表该基因与轨迹无关，只表示模型在当前复杂度下没有稳定拟合成功。

这在细胞异质性较强或轨迹噪音较大时很常见。

---

## 1️⃣2️⃣ associationTest：寻找动态表达基因

```r
assoRes <- associationTest(sce)
head(assoRes)
```

核心输出一般包括：

- `waldStat`：Wald 统计量
- `df`：自由度
- `pvalue`：显著性
- `meanLogFC`：平均对数倍数变化

通常可以这样理解：

- `pvalue` 小：说明该基因在拟时序上存在显著变化
- `meanLogFC` 大：说明沿轨迹的表达变化幅度更明显

这一步适合筛选“轨迹动态基因”。

---

## 1️⃣3️⃣ startVsEndTest：寻找起点与终点相关基因

如果你更关心：

- 哪些基因更像起始状态标记物
- 哪些基因更像终末状态标记物

可以使用：

```r
startRes <- startVsEndTest(sce)
head(startRes)
```

然后按 Wald 统计量排序：

```r
oStart <- order(startRes$waldStat, decreasing = TRUE)
```

取最显著基因并画平滑曲线：

```r
sigGeneStart <- names(sce)[oStart[1]]
plotSmoothers(sce, counts, gene = sigGeneStart)
```

也可以直接查看单个基因的表达分布：

```r
plotGeneCount(crv, counts, gene = sigGeneStart)
plotGeneCount(crv, counts, gene = "GBP7")
```

---

## 1️⃣4️⃣ 用 ggplot2 美化 基因的拟时序表达图

### 14.1 构建绘图数据框

```r
coldata <- data.frame(
  celltype = sim@colData$celltype,
  plotcol = sim@colData$plotcol
)
rownames(coldata) <- colnames(sim)

filter_coldata <- coldata[colnames(sce), ]
filter_coldata$Pseudotime <- sce$crv$pseudotime.Lineage1

gene_wanted <- "GBP7"
top5_genes <- rownames(sce)[oStart[1:5]]
genes_use <- unique(c(top5_genes, gene_wanted))

gene_exp <- sce@assays@data$counts[genes_use, ]
gene_exp <- log2(gene_exp + 1) %>% t()

plt_data <- cbind(filter_coldata, gene_exp)
```

### 14.2 批量绘图

```r
getPalette <- colorRampPalette(brewer.pal(12, "Paired"))
mycolors <- getPalette(length(unique(plt_data$celltype)))

plt_list <- list()

for (gene in genes_use) {
  p <- ggplot(plt_data, aes(x = Pseudotime, y = .data[[gene]], color = celltype)) +
    geom_point(size = 0.6) +
    geom_smooth(se = FALSE, color = "orange") +
    theme_bw() +
    scale_color_manual(values = mycolors) +
    theme(legend.position = "none") +
    labs(y = gene)

  plt_list[[gene]] <- p
}

wrap_plots(plt_list)
```

### 14.3 单独导出带图例的 S100A9 图

```r
gene <- "S100A9"
p_test <- ggscatter(
  data = plt_data,
  x = "Pseudotime",
  y = gene,
  color = "celltype",
  size = 0.6
) +
  geom_smooth(se = FALSE, color = "orange") +
  theme_bw() +
  scale_color_manual(values = mycolors) +
  theme(legend.position = "right") +
  guides(color = guide_legend(
    ncol = 1,
    override.aes = list(size = 3)
  ))

p_test
```

这一类图尤其适合展示：

- 某一目标基因是否沿拟时序逐渐升高或降低
- 不同细胞类型在同一条拟时序上的表达分布差异
- 拟时序与细胞类型之间是否存在明显耦合关系

---

## 1️⃣5️⃣ 这套流程里的几个关键经验

### 15.1 不是所有细胞都适合直接做轨迹分析

如果细胞类型之间完全离散、没有连续过渡，或者混杂太严重，轨迹拟合往往会很牵强。

### 15.2 用于轨迹分析的分群，不一定越细越好

分群太细时，反而容易让轨迹重叠、弯曲，失去解释性。

### 15.3 tradeSeq 计算开销较大

先随机抽样 2000~3000 个细胞，是一个非常现实且有效的折中方案。

### 15.4 模型不收敛不代表基因没有意义

某些基因表达模式复杂，或者数据噪音较大时，NB-GAM 不能稳定收敛是常见现象。

### 15.5 感兴趣基因建议主动加入保留列表

如果像 `GBP7` 这样的目标基因并不一定属于默认高变基因集合，建议在一开始就手动加入 `genes_keep`。

---

## 1️⃣6️⃣ 小结

本文整理了一套从 Seurat 对象出发，进行单核细胞拟时序分析的完整流程，主要包括：

- 筛选目标细胞群
- 转换为 `SingleCellExperiment`
- 利用 `slingshot` 推断轨迹
- 使用 `tradeSeq` 挖掘动态基因
- 结合 `GBP7` 绘制拟时序表达变化图

如果用一句话概括这套方法，它更像是在回答：

> 单细胞数据中，这群细胞不只是“分成了几类”，而是“可能沿着怎样的连续轨迹发生变化”。

对于像单核细胞、巨噬细胞这类具有明显状态转换特征的群体，`slingshot + tradeSeq` 仍然是一套非常实用、可解释性较强的轨迹分析组合。

---

## 📌 参考命令小抄

```r
# 1. 读取对象并筛选细胞
scRNAsub <- subset(data, subset = cell_type %in% c("CD14+ Monocytes", "CD16+ Monocytes"))

# 2. 转换为 SingleCellExperiment
sim <- SingleCellExperiment(assays = List(counts = counts))
reducedDims(sim) <- SimpleList(UMAP = umap)
colData(sim)$celltype <- meta$cell_type

# 3. 轨迹推断
sim <- slingshot(sim, clusterLabels = "celltype", reducedDim = "UMAP", start.clus = "CD14+ Monocytes")

# 4. tradeSeq 分析
icMat <- evaluateK(counts = counts, sds = crv, k = 3:10, nGenes = 500)
sce <- fitGAM(counts = counts, pseudotime = pseudotime, cellWeights = cellWeights, nknots = 6)
assoRes <- associationTest(sce)
startRes <- startVsEndTest(sce)

# 5. 查看 GBP7
plotGeneCount(crv, counts, gene = "S100A9")
```

