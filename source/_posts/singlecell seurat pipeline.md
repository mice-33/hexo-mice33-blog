---
title: 从GEO原始矩阵到细胞注释：一套可复用的Seurat单细胞分析流程
date: 2026-03-23 11:10:00
updated: 2026-03-23 11:10:00
tags:
  - 单细胞
  - Seurat
  - Harmony
  - scDblFinder
  - R语言
  - 生物信息学
categories:
  - 生信分析
  - 单细胞转录组
description: 以GSE182244为例，系统整理一套从原始表达矩阵读入、metadata标准化、质量控制、动态过滤、双细胞去除、聚类、Harmony批次校正到细胞类型注释的Seurat分析流程。
keywords: Seurat, scRNA-seq, Harmony, scDblFinder, GSE182244, PBMC
cover:
---

# 从GEO原始矩阵到细胞注释：一套可复用的Seurat单细胞分析流程

最近在整理自己的单细胞分析代码时，我越来越倾向于把流程拆成两个目标同时推进：

1. **分析结果要能跑通**，也就是从原始矩阵一路走到可解释的聚类和注释；
2. **流程结构要可复用**，不同项目只改数据输入和少量参数，就能快速迁移。

这篇文章就以 **GSE182244 PBMC 数据** 为例，系统梳理一套比较完整的 Seurat 分析流程。整套代码覆盖了：

- 项目目录初始化
- 原始表达矩阵与 metadata 读入
- metadata 统一命名与标准化
- 基础质量控制（mt/rb/hb）
- 按样本动态阈值过滤低质量细胞
- `scDblFinder` 双细胞识别
- 标准 Seurat 聚类
- Harmony 批次效应校正
- marker 可视化与细胞类型注释

如果你也想把自己的单细胞代码从“能跑”整理成“能复用”，这套思路会非常实用。

---

## 1. 流程设计思路：先搭框架，再跑分析

我现在比较推荐的写法，不是直接从 `Read10X()` 或 `CreateSeuratObject()` 开始往下堆，而是先把整个项目管理框架搭起来。

### 1.1 为什么先做目录初始化

单细胞项目很容易越做越乱，特别是当你同时输出：

- 原始对象
- QC 后对象
- 过滤后对象
- 双细胞识别后对象
- Harmony 校正后对象
- metadata 表格
- marker 表格
- UMAP / DotPlot / QC 图

如果这些文件都混在一个目录里，后面维护成本会非常高。

因此，这份流程的第一步就是封装 `init_project()`：

```r
init_project <- function(project_id, base_dir = ".") {
  project_dir <- file.path(base_dir, project_id)
  dirs <- c("raw", "meta", "qs", "tables", "plots", "temp")

  dir.create(project_dir, recursive = TRUE, showWarnings = FALSE)
  invisible(lapply(file.path(project_dir, dirs), dir.create,
                   recursive = TRUE, showWarnings = FALSE))

  list(
    project_id = project_id,
    project_dir = project_dir,
    raw_dir = file.path(project_dir, "raw"),
    meta_dir = file.path(project_dir, "meta"),
    qs_dir = file.path(project_dir, "qs"),
    tables_dir = file.path(project_dir, "tables"),
    plots_dir = file.path(project_dir, "plots"),
    temp_dir = file.path(project_dir, "temp")
  )
}
```

### 1.2 目录结构为什么值得标准化

这一步最大的价值不在“创建文件夹”，而在于**统一项目约定**。

例如：

- 原始数据永远放 `raw/`
- 参数记录永远放 `meta/`
- 中间对象永远放 `qs/`
- 统计表永远放 `tables/`
- 可视化输出永远放 `plots/`

这样后面你换数据集时，只需要换 `project_id` 和输入文件，整个分析骨架基本不用重写。

---

## 2. 用 GSE182244 作为示例项目

这份流程以 `GSE182244` 为例，先把项目参数记录下来：

```r
project_id <- "GSE182244"
paths <- init_project(project_id, base_dir = "projects")

params <- list(
  project_id = "GSE182244",
  species = "Homo sapiens",
  tissue = "PBMC",
  source = "GEO",
  expr_file = "GSE182244_pbmc.integr.exprmatrix.txt.gz",
  meta_file = "GSE182244_pbmc.integr.metadata.txt.gz"
)
```

这里我很喜欢再加一步：把参数写入文本文件。

```r
save_params <- function(param_file, params) {
  writeLines(
    paste(names(params), unlist(params), sep = ": "),
    con = param_file
  )
}
```

这样做有两个好处：

- 后面回看项目时，一眼就知道这套结果对应的是哪份数据；
- 如果将来做多个项目并行，参数记录非常利于追踪。

---

## 3. 原始表达矩阵与 metadata 的读入

这一部分其实最容易“因数据而异”。很多 GEO 项目不是标准 10x 输出，因此不能简单套 `Read10X()`，而要按具体文件结构处理。

这份代码采用的是：

- 表达矩阵用 `data.table::fread()` 读入；
- metadata 通过 `scan()` 先读表头，再用 `read.table()` 读正文。

```r
expr <- data.table::fread(expr_file, data.table = FALSE)
genes <- expr[[1]]
mat <- as.matrix(expr[, -1, drop = FALSE])
rownames(mat) <- genes
storage.mode(mat) <- "numeric"
colnames(mat) <- sub("\\.([0-9]+)$", "-\\1", colnames(mat))
```

这里有一个非常实用的小细节：

```r
colnames(mat) <- sub("\\.([0-9]+)$", "-\\1", colnames(mat))
```

很多 GEO 表达矩阵里的细胞条形码格式和 metadata 里的细胞名不完全一致，比如：

- 矩阵里是 `AAAC... .1`
- metadata 里是 `AAAC...-1`

如果不统一，后面就会卡在矩阵列名和 metadata 行名无法匹配的问题上。

### 3.1 交集细胞这一步一定要做

```r
common_cells <- intersect(colnames(mat), rownames(meta))
mat <- mat[, common_cells, drop = FALSE]
meta <- meta[common_cells, , drop = FALSE]
```

这是我现在几乎每个项目都会保留的一步。

因为只要表达矩阵和 metadata 有一边多了一些细胞，后面 `CreateSeuratObject()` 虽然不一定立即报错，但分析对象会留下隐患。最稳妥的做法就是：

> 只保留两边都存在的细胞，保证 `colnames(mat)` 与 `rownames(meta)` 完全一致。

检查方式也很直接：

```r
identical(colnames(mat), rownames(meta))
```

---

## 4. 创建 Seurat 对象前，先把 metadata 做标准化

很多初学者在这里容易忽略一个问题：

> 真正决定后续流程是否顺畅的，不只是表达矩阵，还有 metadata 是否统一。

我现在比较倾向于把 metadata 明确收敛到一组固定字段：

- `project_id`
- `sample_id`
- `group`
- `batch`
- `nCount_RNA`
- `nFeature_RNA`
- `percent.mt`
- `percent.rb`
- `percent.hb`

这样做的好处是：

- 后续 QC 函数可以复用；
- 多个数据集整合时字段名不乱；
- pseudobulk、CellChat、DESeq2 等下游分析更容易衔接。

### 4.1 统一 sample_id 和 group 的写法

这份代码里用了两套映射：

```r
sample_map <- c(
  "HD1" = "GSM5525599",
  "HD2" = "GSM5525600",
  "HD3" = "GSM5525601",
  "GPP1_1" = "GSM5525596",
  "GPP1_2" = "GSM5525597",
  "GPP2" = "GSM5525598"
)

group_map <- c(
  "GPP" = "MPO-deficient",
  "Healthy" = "Healthy"
)
```

这是非常值得保留的习惯。

原因很简单：原始数据里的样本名和分组名往往并不适合直接进入正式分析。

比如：

- `HD1/HD2/HD3` 更适合映射到真实 GSM 编号；
- `GPP` 这种缩写更适合统一成可解释的疾病标签。

### 4.2 创建 Seurat 对象

在 metadata 整理完之后，再去建对象：

```r
rownames(mat) <- make.unique(rownames(mat))
mat <- Matrix(mat, sparse = TRUE)

seu <- CreateSeuratObject(
  counts = mat,
  project = project_id,
  meta.data = meta_std,
  min.cells = 0,
  min.features = 0
)
```

这里还有两个很关键的小点：

#### （1）重复基因名先处理

```r
rownames(mat) <- make.unique(rownames(mat))
```

否则后面很多函数会直接报错。

#### （2）尽快转成稀疏矩阵

```r
mat <- Matrix(mat, sparse = TRUE)
```

单细胞数据大部分是稀疏的，越早转成 sparse matrix，内存压力越小。

---

## 5. 基础质量控制：把 mt、rb、hb 一次性补齐

这份代码里封装了一个我很喜欢的 QC 函数：`add_basic_qc()`。

核心思想是：

- 如果对象里已经有 `percent.mt/percent.rb/percent.hb`，就直接用；
- 如果没有，就自动计算；
- 同时导出 QC 表格和常见 QC 图。

### 5.1 先准备基因集合

```r
mt_genes <- grep("^MT-", rownames(seu), value = TRUE)
rb_genes <- grep("^RPS|^RPL", rownames(seu), value = TRUE)
hb_genes <- grep("^HB[ABDEGMQZ]", rownames(seu), value = TRUE)
hb_genes <- setdiff(hb_genes, c("HBP1"))
```

这里对血红蛋白基因还专门排除了 `HBP1`，这是个很细但很实用的处理：

> 单纯正则匹配 `^HB` 容易误抓一些并不应该算进血红蛋白污染的基因。

### 5.2 质量控制函数

```r
seu <- add_basic_qc(
  seu,
  mt_genes = mt_genes,
  rb_genes = rb_genes,
  hb_genes = hb_genes,
  paths = paths
)
```

运行后，它会输出：

- `qc_metrics.csv`
- `qc_violin.pdf`
- `qc_scatter.pdf`

这一步的价值在于：

1. 数据是否整体偏低质量，一眼就能看出来；
2. 后面定过滤阈值时，更有依据；
3. 所有图表都有固定输出位置，方便追踪项目。

---

## 6. 过滤低质量细胞：不要只用固定阈值

很多教程会直接写：

```r
subset(seu, subset = nFeature_RNA > 200 & percent.mt < 20)
```

当然这种写法可以用，但当样本间差异比较大时，固定阈值往往太粗糙。

这份代码更进一步，封装了 `filter_pbmc_cells()`，特点是：

- 仍然保留全局默认阈值；
- 但同时按样本计算动态阈值；
- 最终阈值取“默认阈值”和“动态阈值”的组合。

### 6.1 为什么按样本算动态阈值

不同样本在这些指标上可能天然不同：

- 文库深度
- 检测到的基因数
- 线粒体比例

如果所有样本都硬套一刀切标准，很可能：

- 某些样本被过度过滤；
- 某些样本又保留了过多异常细胞。

### 6.2 动态阈值的核心逻辑

代码里用了两种统计策略：

#### 对 `nFeature_RNA` 和 `nCount_RNA`
先取 log10，再按 IQR 规则算上下界：

```r
lower <- 10^(q[1] - 1.5 * iqr)
upper <- 10^(q[2] + 1.5 * iqr)
```

#### 对 `percent.mt`
只取上界：

```r
q[2] + 1.5 * iqr
```

最终又和默认阈值合并：

```r
threshold_df$min_features_final <- pmax(min_features, threshold_df$min_features_dyn, na.rm = TRUE)
threshold_df$max_features_final <- pmin(max_features, threshold_df$max_features_dyn, na.rm = TRUE)
threshold_df$max_counts_final   <- pmin(max_counts, threshold_df$max_counts_dyn, na.rm = TRUE)
threshold_df$max_mt_final       <- pmin(max_mt, threshold_df$max_mt_dyn, na.rm = TRUE)
```

这种策略我个人很喜欢，因为它避免了两个极端：

- 既不是完全死板的固定阈值；
- 也不是完全放任动态阈值无限飘。

### 6.3 过滤结果也要存档

函数最后除了返回过滤后的对象，还会输出：

- `filter_thresholds.csv`
- `cell_filter_decisions.csv`
- `filter_summary.csv`

这对后期复盘特别重要。因为你不只是知道“过滤了”，还知道：

> 每个样本到底用了什么阈值、哪些细胞被删掉、总体删掉多少。

---

## 7. 双细胞识别：按样本跑 `scDblFinder`

在我自己的实践里，双细胞识别最好不要粗暴地对整个对象一把跑完，尤其是在多样本场景下。

这份代码使用的是按样本循环运行 `scDblFinder`：

```r
run_scDblFinder_per_sample <- function(seu, sample_col = "sample_id", seed = 123) {
  ...
}
```

### 7.1 为什么按样本跑

因为双细胞率通常与这些因素相关：

- 每个样本的细胞数
- 上机条件
- 文库复杂度
- 混样方式

如果把所有样本混在一起跑，容易让某些样本的双细胞识别偏移。

### 7.2 这段代码里有一个很实用的保护逻辑

```r
if (ncol(sub) < 100) {
  sub$scDblFinder.score <- NA_real_
  sub$doublet_status <- "singlet"
  out_list[[sid]] <- sub
  next
}
```

如果某个样本细胞数太少，就不强行做双细胞识别。这是非常合理的：

> 小样本本来就不适合稳定估计双细胞结构，硬跑反而容易引入噪音。

### 7.3 识别结果的输出

运行后可以直接检查：

```r
table(seu_df$doublet_status)
table(seu_df$sample_id, seu_df$doublet_status)
```

然后保留 singlet：

```r
seu_singlet <- subset(seu_df, subset = doublet_status == "singlet")
```

并且把：

- 带 doublet score 的对象
- 去除 doublet 后的对象

都保存下来，后面回头查也非常方便。

---

## 8. 标准 Seurat 聚类流程

去除双细胞后，就进入比较经典的 Seurat 标准流程：

```r
seu_singlet <- NormalizeData(seu_singlet, verbose = FALSE)
seu_singlet <- FindVariableFeatures(seu_singlet, selection.method = "vst", nfeatures = 2000, verbose = FALSE)
seu_singlet <- ScaleData(seu_singlet, verbose = FALSE)
seu_singlet <- RunPCA(seu_singlet, npcs = 30, verbose = FALSE)
seu_singlet <- FindNeighbors(seu_singlet, dims = 1:30, verbose = FALSE)
seu_singlet <- FindClusters(seu_singlet, resolution = 0.5, verbose = FALSE)
seu_singlet <- RunUMAP(seu_singlet, dims = 1:30, verbose = FALSE)
```

这是一个非常标准的分析链条。

如果要做得更“工程化”一点，我建议保留以下输出：

- `DimPlot(group.by = "sample_id")`
- `DimPlot(group.by = "group")`
- `DimPlot(label = TRUE)`

因为这三张图可以快速回答三个问题：

1. 样本有没有明显批次分离；
2. 分组有没有初步结构差异；
3. 聚类本身是否分得清楚。

---

## 9. Harmony 批次效应校正

在多样本项目中，聚类前后的批次问题几乎一定要看。

这份流程使用的是 Harmony：

```r
seu_harmony <- harmony::RunHarmony(
  object = seu_singlet,
  group.by.vars = "group",
  reduction.use = "pca",
  dims.use = 1:30
)
```

然后基于 Harmony 低维空间继续：

```r
seu_harmony <- FindNeighbors(seu_harmony, reduction = "harmony", dims = 1:30)
seu_harmony <- FindClusters(seu_harmony, resolution = 0.5)
seu_harmony <- RunUMAP(seu_harmony, reduction = "harmony", dims = 1:30)
```

### 9.1 这里有一个值得注意的点

代码里写的是：

```r
group.by.vars = "group"
```

也就是说，当前校正目标是 `group`。

这个设置在实际项目里要根据目标来决定：

- 如果你只是想先去掉样本批次，很多时候更常见的是 `sample_id` 或 `batch`；
- 如果你这里的 `group` 更像技术批次或混样来源，也可以这样做；
- 但如果 `group` 本身代表真正的生物学分组，是否用它做校正变量，需要非常谨慎。

换句话说，这段代码的框架没有问题，但**校正变量一定要回到具体研究问题里判断**。

### 9.2 矫正前后一定要对比看

代码里保留了前后对比图：

- 未校正版 UMAP
- Harmony 后 UMAP

这个习惯非常重要。

因为批次校正不是“跑完就一定更好”，而是要确认：

- 样本间混合是不是更合理了；
- 真正的生物学结构有没有被抹掉。

---

## 10. Marker 设计与注释思路

这一部分我认为是整套代码里非常有“博客价值”的地方。

因为它不是把注释写死在某一步，而是显式构建了 marker 字典。

```r
marker_list <- list(
  "T cell" = c("PTPRC", "CD3D", "CD3E", "TRAC", "IL7R", "LTB"),
  "NK cell" = c("PTPRC", "NKG7", "GNLY", "KLRD1", "PRF1", "CTSW"),
  "B cell" = c("PTPRC", "MS4A1", "CD79A", "CD79B", "CD74", "HLA-DRA"),
  "Plasma cell" = c("PTPRC", "MZB1", "JCHAIN", "XBP1", "SDC1", "IGKC"),
  "Monocyte" = c("PTPRC", "LYZ", "LST1", "FCN1", "S100A8", "S100A9"),
  "Dendritic cell" = c("PTPRC", "FCER1A", "CST3", "CLEC10A", "HLA-DPA1", "HLA-DPB1"),
  "Platelet" = c("PPBP", "PF4", "SDPR", "NRGN"),
  "Epithelia" = c("EPCAM", "KRT8", "KRT18", "KRT19", "KRT17"),
  "Stromal" = c("PECAM1", "VWF", "COL1A1", "COL1A2", "DCN", "LUM")
)
```

### 10.1 为什么我喜欢“marker 字典”这种写法

因为它把注释过程拆成了两个层次：

#### 第一层：你主观上认为应该有哪些大类细胞

比如：

- T cell
- NK cell
- Monocyte
- B cell
- Dendritic cell

#### 第二层：再通过 DotPlot 和 FindAllMarkers 去验证

这样注释不是黑箱，而是：

> 先有生物学预期，再用数据支持这个预期。

### 10.2 自定义 DotPlot 包装函数也很实用

代码里封装了 `my_DotPlot()`：

```r
my_DotPlot <- function(DotPlot_obj){
  DotPlot_obj$data %>%
    ggplot(aes(x = id, y = features.plot, color = avg.exp.scaled)) +
    geom_point(aes(size = pct.exp)) +
    scale_color_gradientn(colours = rev(RColorBrewer::brewer.pal(11, "RdBu"))) +
    theme_test() +
    theme(text = element_text(size = 16)) +
    labs(x = "", y = "")
}
```

优点是：

- 统一风格；
- 更容易调字体和颜色；
- 以后不同项目可以直接复用。

---

## 11. 用 FindAllMarkers 作为注释辅助证据

完成初步聚类后，这份代码还会跑：

```r
markers_harmony <- FindAllMarkers(
  seu_harmony,
  only.pos = TRUE,
  min.pct = 0.25,
  logfc.threshold = 0.25
)
```

并导出 marker 表：

```r
write.csv(
  markers_harmony,
  file.path(paths$tables_dir, paste0(project_id, ".harmony_cluster_markers.csv")),
  row.names = FALSE
)
```

然后再取每个 cluster 的 top10 marker：

```r
top10_markers <- markers_harmony %>%
  group_by(cluster) %>%
  slice_max(order_by = avg_log2FC, n = 10) %>%
  ungroup()
```

这一步非常重要，因为它能让你的注释不只是“凭经验看 DotPlot”，而是同时参考：

- 经典 marker
- cluster-specific marker

这两套证据放在一起，注释就会稳很多。

---

## 12. 最终注释与对象保存

这份流程最后采用了显式 cluster 对 cell type 的映射：

```r
cluster_map <- c(
  "0"  = "T cell",
  "1"  = "T cell",
  "2"  = "NK cell",
  "3"  = "T cell",
  "4"  = "Monocyte",
  "5"  = "Monocyte",
  "6"  = "B cell",
  "7"  = "Monocyte",
  "8"  = "B cell",
  "9"  = "NK cell",
  "10" = "T cell",
  "11" = "T cell",
  "12" = "Unknown",
  "13" = "Platelet",
  "14" = "Dendritic cell",
  "15" = "Dendritic cell",
  "16" = "Unknown"
)
```

然后写入：

```r
seu_harmony$celltype_l1 <- unname(cluster_map[as.character(seu_harmony$seurat_clusters)])
```

### 12.1 为什么建议保留 `Unknown`

很多人做注释时喜欢强行给每个 cluster 一个名字，但我现在越来越倾向于：

> 看不准就保留 `Unknown`。

因为：

- 某些 cluster 可能是低质量残留；
- 某些 cluster 可能需要更细粒度 marker 才能拆；
- 某些 cluster 可能本身就是混合群。

与其误注释，不如先诚实保留。

### 12.2 为什么保存多个阶段对象很重要

这份代码会把不同阶段的对象都存到 `qs/` 目录，比如：

- raw
- qc
- filtered
- singlet
- clustered
- harmony_annotated

这种做法最大的好处是：

后面如果你要返工某一步，不需要从头再跑。

例如：

- 想改 QC 阈值，就从 `qc.qs` 往后接；
- 想改双细胞去除，就从 `filtered.qs` 往后接；
- 想改注释，就直接载入 `harmony_group.qs`。

这对大型项目节省时间非常明显。

---

## 13. 这套流程适合哪些场景

我觉得这套分析框架尤其适合下面几类场景：

### 13.1 GEO 公共数据复现

因为它对“非标准 10x 文件格式”的兼容性更好，特别适合：

- 表达矩阵和 metadata 分开下载；
- 细胞命名格式需要手动对齐；
- metadata 列名杂乱，需要统一重命名。

### 13.2 多项目并行分析

因为项目目录、文件命名和保存习惯都是标准化的，所以多个项目可以共用一套框架。

### 13.3 后续还要接下游分析

比如：

- CellChat
- pseudobulk + DESeq2
- module score
- 差异表达
- 轨迹分析

只要前面的 metadata 和对象保存规范了，下游衔接会顺很多。

---

## 14. 我觉得还可以继续优化的地方

虽然这套代码已经很完整，但从“长期复用”的角度，我觉得还有几处可以继续升级。

### 14.1 把参数彻底配置化

比如把这些内容放到单独配置文件：

- 物种
- 组织类型
- 过滤阈值
- marker 字典
- cluster 注释映射
- Harmony 校正变量

这样整套流程就更接近“半自动管线”。

### 14.2 注释部分可引入参考映射

现在是基于 marker + 人工判断做一级注释，这很适合作为基础版本。

后面还可以引入：

- SingleR
- Azimuth
- scRNA reference mapping

把人工 marker 注释和参考图谱注释结合起来。

### 14.3 Harmony 的校正变量要更明确

前面提到，`group.by.vars = "group"` 这一步一定要根据研究问题谨慎设置。最好把：

- `sample_id`
- `batch`
- `group`

这几种方案都可视化对比一下，再定最终策略。

---

## 15. 小结

如果只用一句话来总结这套流程，我会这么说：

> 这不是一份“把 Seurat 常规命令串起来”的脚本，而是一套从项目管理、metadata 标准化、QC、动态过滤、双细胞识别、批次校正到注释保存都尽量规范化的单细胞分析骨架。

它最大的价值不是某一步用了哪个函数，而是把这些关键问题都提前想清楚了：

- 文件怎么组织
- metadata 怎么统一
- QC 怎么留痕
- 双细胞怎么按样本处理
- 批次校正怎么接入
- 注释怎么既有 marker 支撑，又能可视化展示
- 中间结果怎么保存，方便返工

如果你正在整理自己的单细胞代码，我非常建议把“脚本能跑”再往前推进一步，升级成“流程可复用”。

---

## 附：这套流程的最小主线

最后附一版我理解下的核心主线，方便快速浏览。

```r
# 1. 初始化项目
paths <- init_project(project_id, base_dir = "projects")

# 2. 读入表达矩阵与metadata
# 3. 统一metadata并创建Seurat对象

# 4. 计算QC指标
seu <- add_basic_qc(seu, mt_genes, rb_genes, hb_genes, paths)

# 5. 过滤低质量细胞
seu_filtered <- filter_pbmc_cells(seu, paths = paths)

# 6. 双细胞识别
seu_df <- run_scDblFinder_per_sample(seu_filtered, sample_col = "sample_id")
seu_singlet <- subset(seu_df, subset = doublet_status == "singlet")

# 7. 标准聚类
seu_singlet <- NormalizeData(seu_singlet)
seu_singlet <- FindVariableFeatures(seu_singlet)
seu_singlet <- ScaleData(seu_singlet)
seu_singlet <- RunPCA(seu_singlet)
seu_singlet <- FindNeighbors(seu_singlet, dims = 1:30)
seu_singlet <- FindClusters(seu_singlet, resolution = 0.5)
seu_singlet <- RunUMAP(seu_singlet, dims = 1:30)

# 8. Harmony校正
seu_harmony <- RunHarmony(seu_singlet, group.by.vars = "group")

# 9. marker与注释
markers_harmony <- FindAllMarkers(seu_harmony)
seu_harmony$celltype_l1 <- unname(cluster_map[as.character(seu_harmony$seurat_clusters)])
```

如果后面我把这套流程进一步整理成函数化模板或 Snakemake / targets 版本，我再继续补一篇。
