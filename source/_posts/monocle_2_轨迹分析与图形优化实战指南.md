---
title: Monocle2轨迹分析
date: 2026-03-30 23:00:00
updated: 2026-03-30 23:00:00
tags:
  - 单细胞
  - Seurat
  - monocle2
  - 拟时序分析
categories:
  - 生信分析
  - 单细胞转录组
  - 轨迹分析
author: mice33
keywords: monocle2, Seurat, 单细胞轨迹分析,  拟时序
top: false
cover:
---

# Monocle2轨迹分析

参考资料：[单细胞之轨迹分析-2：monocle2 原理解读+实操 - 简书](https://www.jianshu.com/p/5d6fd4561bc0)



[TOC]

## Monocle2分析整体流程（结构化框架）

### Step 1：构建CellDataSet对象

```r
library(monocle)

convert_seurat_to_cds <- function(seurat_obj, assay="RNA", out_file=NULL) {
  # 提取矩阵
  expr_matrix <- as(as.matrix(seurat_obj@assays[["RNA"]]@counts),"sparseMatrix")
  #expr_matrix <- GetAssayData(object = seurat_obj, assay = assay, slot = "counts")
  #expr_matrix <- as(expr_matrix, "sparseMatrix")
  
  #表型数据
  pd <- seurat_obj@meta.data
  pd$cell_id <- rownames(pd)
  pd <- new("AnnotatedDataFrame", data = pd)
  
  #提取基因信息
  fd <- data.frame(
    gene_id = rownames(expr_matrix),
    gene_short_name = rownames(expr_matrix),
    row.names = rownames(expr_matrix)
  )
  fd <- new("AnnotatedDataFrame", data = fd)
  
  # 创建CDS
  cds <- newCellDataSet(
    expr_matrix,
    phenoData = pd,
    featureData = fd,
    lowerDetectionLimit = 0.5,
    expressionFamily = negbinomial.size()
  )
  
  # 保存结果
  if(!is.null(out_file)) {
    saveRDS(cds, file = out_file)
  }
  
  return(cds)
}

#cds <- convert_seurat_to_cds(pbmc, out_file = "my_cds.rds")
```

核心要点：
- 输入必须是 **counts或接近counts的数据**
- meta信息要完整（cluster / celltype）

#### 参数解读：

expressionFamily

![image-20260330131141297](C:\Users\mice\AppData\Roaming\Typora\typora-user-images\image-20260330131141297.png)

在Monocle2分析中，expressionFamily参数用于指定表达数据的统计分布假设。

对于原始UMI或read count数据，由于其具有离散性和过度离散（overdispersion）特征，通常采用负二项分布建模（negbinomial.size()）。相比negbinomial()，该方法在保证计算效率的同时具有较好的拟合性能，因此被广泛使用。

而对于经过归一化或对数转换的数据（如TPM/FPKM或Seurat的log-normalized表达矩阵），其分布更接近连续型（常近似对数正态分布），此时应使用高斯分布模型（gaussianff()），而不应继续采用负二项分布。

tobit()模型用于处理具有“截断（censored）”特征的连续表达数据，即当真实表达值低于某一检测阈值时被记录为0。这类模型曾用于早期非UMI单细胞数据（如FPKM/TPM），以解释大量零表达值的来源。然而，在当前以UMI计数为主的单细胞测序数据中，表达值本质为离散计数，负二项分布（negbinomial.size()）能够更合理地建模其统计特性。因此，tobit()在现代单细胞分析流程中已较少使用。

### Step 2：标准化与离散度估计

```r
cds <- estimateSizeFactors(cds)
cds <- estimateDispersions(cds)
```

作用：
- size factor：消除测序深度差异
- dispersion：用于筛选高变基因

### Step 3：过滤

一般来说，到这里之前Seurat里面已经完成了细胞过滤，这里可以省略。不过可以根据表达该基因的细胞数目过滤一下基因

```R
cds <- detectGenes(cds, min_expr = 0.1) ##统计某基因在多少个细胞中表达大于0.1
summary(fData(cds)$num_cells_expressed)

expressed_genes <- rownames(
  subset(fData(cds), num_cells_expressed >= 10)
) ##至少在10个细胞中表达大于上面设定的阈值
```

###  Step 4：细胞分类

不推荐用Monocle自带的做，应该在Seurat对象里面已经分清楚。

### Step 5：选择ordering genes（轨迹定义基因）

在 Monocle2 中，**ordering genes 的选择直接决定轨迹结构**，本质上是在回答：

> 👉 “用哪些基因去定义细胞状态变化？”

不同策略对应不同的生物学假设。

#### 🧠 方法分类

| 方法                   | 类型   | 核心思想           | 是否推荐 |
| ---------------------- | ------ | ------------------ | -------- |
| 差异基因（cluster）    | 无监督 | 细胞群差异驱动轨迹 | ⭐⭐⭐⭐     |
| 高变基因（dispersion） | 无监督 | 表达波动最大       | ⭐⭐⭐      |
| 发育marker             | 半监督 | 已知生物过程       | ⭐⭐⭐⭐     |
| 时间/状态差异基因      | 弱监督 | 拟时间相关         | ⭐⭐⭐⭐⭐    |

#### 方法1：差异表达基因（最常用 & 推荐）

理想状态下，我们希望尽可能少的使用正在研究的对象的先验知识，以免引入过多偏见。

##### 📌 核心逻辑

> 不同细胞群之间的差异基因
>  👉 最有可能驱动状态转变

```r
diff <- differentialGeneTest(
  cds,
  fullModelFormulaStr = "~celltype" ##差异分析变量，理论上可为pData(cds)的所有列名
)

deg <- rownames(
  subset(diff, qval < 0.01) ##过滤一下
)

##保存差异基因
write.table(deg, file = "monocle.DEG.xls", col.names = F, sep = "\t", quote = F)

ordering_genes <- rownames(deg)

##排序的基因一般2000左右比较合适，太多了的话可以top以下
ordering_genes <- rownames(deg)[order(deg$qval)][1:2000]
```

#### 方法2：高变基因（适用于一群细胞内部的轨迹分析）

即没有内部分群时

```R
disp_table <- dispersionTable(cds)

ordering_genes <- subset(
  disp_table,
  mean_expression >= 0.1 &  ##过滤防止噪声
  dispersion_empirical >= dispersion_fit
)$gene_id

```

dispersion_empirical表示基因表达的实际离散程度，而dispersion_fit表示在给定平均表达水平下模型预测的技术噪声水平。通过筛选dispersion_empirical大于dispersion_fit的基因，可以识别出表达波动显著高于预期的基因（即overdispersed genes），这些基因更可能反映真实的生物学异质性，而非测序噪声。

#### 其他方法分析

| 方法                             | 本质在选什么       | 信息类型          |
| -------------------------------- | ------------------ | ----------------- |
| `VariableFeatures`（Seurat）     | 方差最大的基因     | 技术+生物混合波动 |
| `FindAllMarkers`（Seurat）       | cluster差异基因    | 细胞群差异        |
| `dispersionTable`                | overdispersion基因 | 表达不稳定性      |
| `differentialGeneTest(~Cluster)` | 模型驱动差异       | 统计显著变化      |

不同基因选择方法本质上反映了不同的信息提取策略。Seurat的VariableFeatures和Monocle的dispersionTable侧重于识别表达波动较大的基因，但这些基因未必与细胞状态变化直接相关。FindAllMarkers则通过群间差异捕捉细胞类型特异性表达，而Monocle的differentialGeneTest通过广义线性模型在统计框架下识别与细胞分组显著相关的基因。

由于轨迹推断依赖于连续状态变化信号，基于差异表达（尤其是模型驱动的方法）通常比单纯基于表达波动的方法更为稳健。因此，推荐优先使用differentialGeneTest结合表达覆盖过滤来选择ordering genes。

### Step 6：降维与排序

```r
cds <- setOrderingFilter(cds, ordering_genes)
cds <- reduceDimension(cds, 
                       max_components = 2, ## 输出维度
                       method = "DDRTree")
cds <- orderCells(cds) ## 在树上排序
# cds <- orderCells(cds, root_state = 5)
```

关键点：
- DDRTree = Monocle2核心
- orderCells决定伪时间方向，使用root_state可以设置起点

### Step 7：拟时序结果可视化

#### 拟时序轨迹

```R
plot_cell_trajectory(cds, color_by = "Pseudotime", size = 1, show_backbone = TRUE)

##颜色对接ggsci
plot_cell_trajectory(cds, color_by = "Pseudotime", size = 1, show_backbone = TRUE) + scale_color_npg()
```

#### 细胞类型轨迹

```R
plot_cell_trajectory(cds, color_by = "celltype", size = 1, show_backbone = TRUE)

##可以画成树状图，更直观
p1 <- plot_cell_trajectory(cds, x=1, y=1,
                           color_by = "celltype")+
  theme(legend.position="none",panel.border = element_blank())+ ## 去掉p1的legend
  scale_color_npg()
p2 <- plot_complex_cell_trajectory(cds, x=1, y=1,
                                   color_by = "celltype")+
  scale_color_npg()+
  theme(legend.title = element_blank())
p1|p2

##沿时间轴的细胞密度图
pdata <- pData(cds)
ggplot(pdata, aes(Pseudotime, color= celltype, fill= celltype))+
  geom_density(bw = 0.5,size=1,alpha=0..5)+theme_classic2()
```

#### State轨迹

```R
plot_cell_trajectory(cds, color_by = "State", size = 1, show_backbone = TRUE)

##分面
plot_cell_trajectory(cds, color_by = "State") + facet_wrap("~State", nrow = 1)
```

### Step 8 ：指定基因可视化

```R
gene_use <- "SDCCAG8"
##直观变化--------------------------------
cds_subset <- cds[gene_use,]
p1 <- plot_genes_in_pseudotime(cds_subset,color_by = "State")
p2 <- plot_genes_in_pseudotime(cds_subset,color_by = "celltype")
p3 <- plot_genes_in_pseudotime(cds_subset,color_by = "Pseudotime")
p1|p2|p3

##其他展示方式-----------------------------
p1 <- plot_genes_jitter(cds_subset,grouping = "State",color_by = "State")
p2 <- plot_genes_violin(cds_subset,grouping = "State",color_by = "State")
p3 <- plot_genes_in_pseudotime(cds_subset,color_by = "State")
p1|p2|p3

##拟时序降维图-----------------------------
##标准美化版
library(monocle)
library(ggplot2)
library(viridis)

plot_gene_trajectory_replot <- function(cds_obj,
                                        gene_use,
                                        out_pdf = NULL,
                                        log_transform = TRUE,
                                        upper_quantile = 1,
                                        palette_option = "plasma",
                                        point_size = 1.2,
                                        point_alpha = 0.9,
                                        base_size = 14,
                                        plot_title = NULL) {
  
  if (!gene_use %in% rownames(cds_obj)) {
    stop(paste0(gene_use, " is not present in rownames(cds_obj)."))
  }
  
  # 1. 提取 Monocle 降维坐标
  coord_df <- as.data.frame(reducedDimS(cds_obj))
  coord_df <- t(coord_df)
  coord_df <- as.data.frame(coord_df)
  
  if (ncol(coord_df) < 2) {
    stop("reducedDimS(cds_obj) does not contain at least 2 dimensions.")
  }
  
  colnames(coord_df)[1:2] <- c("Dim1", "Dim2")
  coord_df$cell_id <- rownames(coord_df)
  
  # 2. 提取表达
  expr_vec <- as.numeric(exprs(cds_obj)[gene_use, coord_df$cell_id])
  
  if (log_transform) {
    expr_vec <- log1p(expr_vec)
  }
  
  upper_cut <- quantile(expr_vec, upper_quantile, na.rm = TRUE)
  expr_vec_plot <- pmin(expr_vec, upper_cut)
  
  coord_df$expr <- expr_vec_plot
  
  # 3. 关键：按表达从低到高排序，保证高表达最后画
  coord_df <- coord_df[order(coord_df$expr, decreasing = FALSE), ]
  
  # 4. 标题
  if (is.null(plot_title)) {
    plot_title <- paste0(gene_use, " log1p expression along trajectory")
  }
  
  # 5. 作图
  p <- ggplot(coord_df, aes(x = Dim1, y = Dim2, color = expr)) +
    geom_point(size = point_size, alpha = point_alpha) +
    scale_color_viridis_c(
      option = palette_option,
      name = paste0(gene_use, "\nexpression")
    ) +
    theme_classic(base_size = base_size) +
    theme(
      axis.line = element_line(color = "black", linewidth = 0.6),
      plot.title = element_text(hjust = 0.5)
    ) +
    labs(
      title = plot_title,
      x = "Component 1",
      y = "Component 2"
    )
  
  if (!is.null(out_pdf)) {
    pdf(out_pdf, width = 6, height = 5)
    print(p)
    dev.off()
  }
  
  return(p)
}

p <- plot_gene_trajectory_replot(
  cds_obj = cds,
  gene_use = "SDCCAG8",
  out_pdf = "data_trajectory_replot.pdf",
  log_transform = TRUE,
  upper_quantile = 0.99,
  palette_option = "plasma",  ##magma，plasma，inferno
  point_size = 1.3,
  point_alpha = 0.9
)

print(p)

# log1p(x) = log(1 + x)
```

### Step 9：寻找拟时序相关基因

#### 画热图

```R
#Time_diff <- differentialGeneTest(cds[ordering_genes,],cores = 1,
#                                  fullModelFormulaStr = "~sm.ns(Pseudotime)")
#这里是单独把排序基因提取出来做回归分析，如果不改则全做

Time_diff <- differentialGeneTest(cds, cores = 1,
                                  fullModelFormulaStr = "~sm.ns(Pseudotime)")

Time_diff <- Time_diff %>% filter(qval < 0.05) ##限制基因数目
write.csv(Time_diff,"Time_diff_all.csv",row.names = F)

Times_genes <- Time_diff %>% pull(gene_short_name)%>%as.character()
Times_genes <- Times_genes[1:500]  # 前500个显著基因

#p = #plot_pseudotime_heatmap(cds[Times_genes,],num_clusters=4,show_rownames=T,return_heatma#p=T)  ## 这里如果画不出来，报错的话，有可能是基因太多了
p = plot_pseudotime_heatmap(cds[Times_genes,],show_rownames=T,return_heatmap=T)## 不用num_clusters参数指定聚类数目
p

ggsave("Time_heatmapAll.pdf",p,width = 5, height = 10)
```

#### 单独提取某一个cluster

```R
p$tree_row

clusters <- cutree(p$tree_row, k=4) ##k值应当和前面热图的聚类数目保持一致
clustering <- data.frame(clusters)
clustering[,1] <- as.character(clustering[,1])
colnames(clustering) <- "Gene_Clusters"
table(clustering)

write.csv(clustering,"Time_clustering_all.csv",row.names = F)

##提取cluster6------------------
cluster6_genes <- clustering %>%
  dplyr::filter(Gene_Clusters == "6") %>%
  dplyr::pull(Gene_name)

length(cluster6_genes)  # 查看有多少基因
##富集分析--------------------------
library(org.Hs.eg.db)  # 人类示例，老鼠用 org.Mm.eg.db
library(clusterProfiler)
plot_KEGG <- function(ENTREZID,organism="hsa",qvalueCutoff=0.05){
  KEGG_res <- enrichKEGG(ENTREZID$ENTREZID,organism,pvalueCutoff = 1,qvalueCutoff = qvalueCutoff)
  
  KEGG_plot <- dotplot(KEGG_res)
  
  KEGG_res <- list(KEGG_res=KEGG_res,KEGG_plot=KEGG_plot)
  
  return(KEGG_res)
}
plot_GO <- function(ENTREZID,orgdb,ont="ALL",keyType="ENTREZID",qvalueCutoff=0.05){

  GO_res <- enrichGO(ENTREZID$ENTREZID,keyType = keyType,OrgDb = orgdb,ont=ont,readable = T,pvalueCutoff = 1,qvalueCutoff = 0.05)
  
  GO_res <-  clusterProfiler::simplify(GO_res,cutoff=0.7,select_fun=min)
  
  GO_plot <- dotplot(GO_res,split = "ONTOLOGY") + facet_wrap(.~ONTOLOGY,scales="free")
  
  GO_res <- list(GO_res=GO_res,GO_plot=GO_plot)
  
  
  return(GO_res)
}


# 转换 gene symbol → entrez
cluster6_entrez <- bitr(cluster6_genes, 
                        fromTyp = "SYMBOL",
                        toType = "ENTREZID",
                        OrgDb = org.Hs.eg.db)
length(cluster6_entrez$ENTREZID)  # 确认多少基因成功映射

# Cluster6 示例
GO6 <- plot_GO(cluster6_entrez, orgdb = org.Hs.eg.db,qvalueCutoff= 1)
KEGG6 <- plot_KEGG(cluster6_entrez, organism = "hsa",qvalueCutoff = 1)

# 导出 CSV
write.csv(as.data.frame(GO6$GO_res), "Cluster6_GO.csv", row.names = FALSE)
write.csv(as.data.frame(KEGG6$KEGG_res), "Cluster6_KEGG.csv", row.names = FALSE)

# 保存图片
if(!is.null(GO6$GO_plot)){
  ggsave("Cluster6_GO_dotplot.pdf", plot = GO6$GO_plot, width = 8, height = 6)
}

if(!is.null(KEGG6$KEGG_plot)){
  ggsave("Cluster6_KEGG_dotplot.pdf", plot = KEGG6$KEGG_plot, width = 8, height = 6)
}
```

### Step 10：分支分析

```R
plot_cell_trajectory(cds,color_by = "State")

##BEAM进行统计分析
#BEAM_res <- BEAM(cds, branch_point = 1, cores = 2) ##对所有基因排序，很慢
BEAM_res <- BEAM(cds[ordering_genes,], branch_point = 1, cores = 2)

BEAM_res <- BEAM_res[order(BEAM_res$qval),]
BEAM_res <- BEAM_res[,c("gene_short_name","pval","qval")]
head(BEAM_res)

write.csv(BEAM_res,"BEAM_res.csv",row.names = F)

plot_genes_branched_heatmap(cds[row.names(subset(BEAM_res,
                                                 qval< 1e-4)),],
                            branch_point = 1, ##绘制的是哪个分支
                            num_clusters = 4, ##分成几个cluster
                            cores = 1,
                            use_gene_short_name = T,
                            show_rownames = T)
##画出的图是以这个分叉路口向两边state发展的热图

##选前100个基因可视化--------------------------------
BEAM_genes <- BEAM_res %>%
  arrange(qval) %>%
  head(100) %>%
  pull(gene_short_name) %>%
  as.character()

p <- plot_genes_branched_heatmap(cds[BEAM_genes,],branch_point = 1,
                                 num_clusters = 3,show_rownames = T,return_heatmap = T)
ggsave("BEAM_heatmap.pdf",p$ph_res,width = 6.5,height = 10)
```

