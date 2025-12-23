# 项目2（6–12个月）：结构预测与“序列-结构”多模态建模（结构理解 + 可解释评测）
> 目标：从“会用结构文件”升级到“能围绕结构做可复现建模与分析”，覆盖 JD 的 structure prediction、multi-modal、biophysics/structural biology 的沟通能力。  
> 你不需要训练 AlphaFold 级别的模型；你需要的是：**结构数据工程 + 结构指标 + 多模态模型 + 失败分析**。

---

## 0. 6个月末交付清单
**交付物 A：结构数据工程与质量体系**
- 支持从 PDB / AlphaFold DB 获取结构（mmCIF/PDB）
- 统一清洗：链选择、缺失残基、替代构象、配体/离子处理、序列映射
- 结构质量指标：分辨率（实验）、pLDDT/PAE（AF）、clash/geometry（可选）
- 可复现的结构特征提取：
  - 二级结构、溶剂可及性（SASA）、接触图/距离图
  - 界面特征（PPI interface residues、buried surface area proxy）
  - 口袋特征（可选，留给项目3）

**交付物 B：一个多模态模型与系统评测**
- 输入：sequence embedding（来自项目1的 PLM 或现成 PLM） + structure features（图/几何）
- 输出（选一个主任务，另一个作附加任务）：
  1) ΔΔG/稳定性变化（突变层面）
  2) PPI 亲和力/界面热点预测（蛋白-蛋白）
  3) developability proxy（聚集/溶解性/表达相关）
- 评测：严格 split（按家族/蛋白去泄漏）+ 分层分析（结构区域、界面/核心/无序）

**交付物 C：结构可视化解释资产**
- 你能把模型输出（打分/不确定性）映射回结构，做成 “case study 卡片”
- 输出：每个案例 1 张结构图 + 5 行解释（适合面试与跨团队沟通）

---

## 1. 本项目要训练的核心能力
1) 结构数据生态：PDB vs AlphaFold、mmCIF、链/残基编号、缺失/插入等坑  
2) 结构特征工程：把 3D 变成 ML 可用特征（图、几何、界面）  
3) 多模态融合：sequence + structure + (assay metadata)  
4) 结构相关失败分析：什么时候结构信息帮忙、什么时候害人

---

## 2. 选题建议（选 1 主 1 副）
你需要一个“清晰可评测”的任务。以下按“可控性”排序：

### 主任务候选 1：突变导致的稳定性变化（ΔΔG proxy）
- 优点：数据相对多、定义清楚、适合结构/序列融合
- 输出解释：突变在核心/表面、是否破坏疏水核/盐桥

### 主任务候选 2：PPI 界面热点预测/亲和力变化
- 优点：更贴 biologics（抗体/蛋白 binder）
- 输出解释：界面残基、接触网络、界面几何

### 副任务：结构质量/置信度建模
- 把 AF 的 pLDDT/PAE 当作标签或辅助任务，学“哪里不可靠”

> 选择原则：先选你未来要工作的 biologics 场景（binder/抗体更偏 PPI）。

---

## 3. 技术路线总览
### 3.1 结构特征表示方式（建议从简单到复杂）
**Level 0（最快跑通）**
- 仅用序列 embedding（项目1）+ 简单结构摘要特征（SASA、二级结构比例、接触密度）

**Level 1（推荐主战）**
- 结构图：残基作为节点，边为距离阈值（<8Å）或 KNN
- 节点特征：AA type、二级结构、SASA、是否界面、AF 置信度等
- 模型：GNN（GraphSAGE/GAT）或轻量 geometric features + MLP

**Level 2（进阶，可选）**
- SE(3) 等变网络/几何深度学习（更难，但也更贴前沿）

### 3.2 多模态融合策略（可写成贡献点）
- concat（最简单）
- cross-attention（sequence tokens ↔ structure nodes）
- gating/fusion（让模型“决定什么时候信结构”） ← 很贴你的 JD

---

## 4. 6个月计划（按月里程碑）
### Month 1：结构数据与可视化基本功
**目标**
- 你能熟练：下载 PDB/AF 结构、在 ChimeraX 看链/界面/缺失
- 写好结构解析器：mmCIF/PDB → residue table（chain, res_id, coords, aa）

**任务清单**
1) 实现：结构读取 + 序列抽取 + chain 选择
2) 实现：residue-level table（每残基一个向量）
3) 实现：基础结构特征：二级结构、SASA（可先用简化工具/库）
4) 建一个结构质控报告：缺失比例、异常残基、长度不一致

**验收**
- 给任意蛋白，你能在 10 分钟内：找到结构来源（PDB/AF）、指出低置信区段、导出图

### Month 2：定义任务与数据集（最重要）
**目标**
- 锁定主任务数据集与拆分方案（防泄漏）
- 做 baseline：序列-only 与结构摘要特征

**任务清单**
1) 数据 schema：
   - entity（protein/complex）
   - variant（mutation list）
   - structure pointer（PDB/AF file）
   - label（ΔΔG/affinity/etc）
   - assay metadata（温度、pH、方法）
2) split：
   - 按 protein / family 分组 split
   - variant-level split 是不够的（会泄漏）
3) baseline：
   - PLM embedding + MLP
   - + 结构摘要特征
4) 生成首版报告

**验收**
- baseline 能跑通，且你能解释“结构摘要特征是否提升”

### Month 3：结构图建模（GNN 主模型）
**目标**
- 建一个 residue-graph 模型 + 与 PLM 融合
- 在严格 split 上稳定提升

**任务清单**
1) 构图：KNN 或距离阈值；支持缓存（避免每次重算）
2) 模型：GNN encoder + fusion head
3) 训练：稳定性（梯度、loss、早停）
4) 评测：分层分析
   - core vs surface vs interface vs disorder（用 pLDDT 或 SASA proxy）

**验收**
- 有主表 + 分层表，且能指出“结构在哪些区域贡献大”

### Month 4：可解释与案例驱动分析（面试/论文最加分）
**目标**
- 做 20 个 case study 卡片（每个：结构图 + 解释）
- 建立“失败模式库”：模型什么时候错、为什么错

**任务清单**
1) 对每个样本输出：残基重要性/贡献（简单可行的方法即可）
2) 在 ChimeraX 映射：按重要性上色
3) 总结 5 类失败模式：
   - 无序区
   - 构象变化
   - 界面诱导 fit
   - 数据噪声/条件不一致
   - 结构来源偏差（AF vs PDB）

**验收**
- 你能用结构图解释 5 个成功案例 + 5 个失败案例

### Month 5：把“多模态融合”做成方法点
**目标**
- 让模型学会“什么时候信结构、什么时候信序列”
- 做消融，形成可写贡献

**候选方法点**
- fusion gate：输入（protein embedding + structure summary）输出一个 gate，控制结构分支权重
- 不确定性：对低 pLDDT 区域降权或 dropout，提升鲁棒性

**验收**
- 在多个子集上更稳（尤其低质量结构/低置信区域）

### Month 6：工程化打磨 + 论文结构化写作
**目标**
- 一键跑全流程：prepare → train → eval → make_cases
- 报告写成“论文样式”

**任务清单**
1) 完整复现脚本与 README
2) 输出最终报告：
   - 主表
   - 分层结果
   - case study 附录
3) 准备投稿/内部分享：15–20 页 slide

**验收**
- 任何人拿到 repo 能复现你的结论

---

## 5. 风险与备选
- **结构特征太复杂** → 先用摘要特征 + 强 split + 强分析也能发（benchmark/tool）  
- **标签噪声大** → 增加条件建模（assay metadata）+ robust loss  
- **多模态收益不稳定** → 把“什么时候无效”写成结论也是贡献

---

## 6. 你可以如何写到简历里（模板）
- Built a multimodal sequence+structure modeling pipeline with leakage-controlled splits and residue-level graph features to predict <task>.
- Developed interpretability workflows mapping model attributions to 3D structures for actionable mutation hypotheses.
- Established a structure-quality-aware fusion strategy improving robustness across experimental and predicted structures.

---

## 7. 第一周最小执行清单
1) 选 3 个蛋白（你熟悉的领域即可）下载 PDB/AF 结构  
2) 写一个脚本：结构 → residue table（chain/res_id/aa/coords）  
3) 在 ChimeraX 打开并标注：缺失区域/低置信区域（pLDDT）  
4) 输出一张“结构数据质控报告”（哪怕只有长度/缺失/置信度分布）  
