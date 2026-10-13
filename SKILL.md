---
name: shop-feedback-cluster
description: 当中文电商运营需要把本地商品评论、客服或退换货反馈CSV整理为可追溯的问题聚类、需求卡和数量汇总，识别重复记录、匿名反馈与反证时使用。
---

# 评论与客服需求聚类

把已有导出的中文反馈变为可审阅需求卡。保留原话与记录ID，区分反馈条数和独立已知顾客数。只读输入，不收集在线评论、不联系顾客、不代替产品或售后负责人决定。

## 读取与计数

读取本地CSV/文本。最低字段是 feedback_id、text；customer_id、channel、SKU、日期、退货原因可缺失。保留来源文件和原始行号；缺 feedback_id 则按文件/行号分配稳定ID。

用可执行的CSV处理进行行数、完全重复和顾客数计算，不靠模型估算。对同一完整字段完全相同的重复行只计一次，保存重复行到保留行的映射。相同ID但文本/顾客等字段不同是ID冲突，全部保留并标待核实，不能覆盖。不同ID的同义反馈只聚类，不当数据重复删除；同一顾客再次报告计两条反馈、一个已知顾客。

input_rows 为实际数据行数（不含表头）；dedup_rows 为去除完全重复后的行数。空白customer_id为未知，不能分配虚构U号，也不能用“匿名1/匿名2”声称是两个自然人。

## 聚类与需求卡

按“顾客在什么情境下遇到什么障碍”归类，区分：
- 商品质量/参数误解；
- 页面或入口可发现性；
- 技术报错；
- 物流/售后规则冲突；
- 产品或内容改进请求。

同一关键词不等于相同问题；例如找不到规格入口与手机详情页加载报错是不同障碍。支持多意图时明确一条可进入多个簇，注明簇计数相加可能大于dedup_rows。

逐簇引用 feedback_ids、代表原话和实际已知顾客数。保留方向相反或限制方案的记录作为 counterevidence，说明它挑战的是哪个推论。对不同问题的意见不能硬凑成反证；没有反证用空数组。顾客说“需显示条件”和“不要堆满卖点”可能是两种呈现约束，不能直接宣布双方反对同一功能。

需求卡包含问题、证据、反证/未知、建议、验证动作及优先级理由。优先级是基于输入中的影响、频次和证据可靠性的复核建议；频次不是全体顾客比例。单条报错可建议优先复现，不能据此宣布普遍重大故障。金额、影响订单或销量没给则未知，不推算收入损失。以前taxonomy存在时沿用语义一致的ID，新增或拆分另列变化。

成本与实施难度只能在输入有工时/资源/技术条件等依据时作为事实进入优先级。没有成本资料，cost为unknown；不能把“报错”“页面改动”推成“低成本即可复现”“很容易修复”或确定工时。确有需要提出成本/难度判断时，单列为estimate，说明估计依据、假设和需验证项；不能把未评估的成本当已知排序理由。这条要求同时适用于priority_reason、next_step与报告中的建议措辞。

## 文件合同

在用户指定目录真实写入UTF-8 report.md、result.json；未指定用 outputs/shop-feedback-cluster/，不修改CSV输入。

report.md 包含原始/去重数量、去重映射、聚类表、逐卡证据与优先级理由、复核项和限制。

result.json 顶层包含：
- skill: "shop-feedback-cluster"
- source_ids: 实际读取的反馈ID数组
- input_rows、dedup_rows: 整数
- duplicate_rows: 每项含source_row、kept_row、feedback_id、reason
- clusters: 每项含 cluster_id、title、feedback_ids、unique_customers、unknown_customers、counterevidence、need、next_step、priority_reason；成本被用于判断时另列cost，写明fact/estimate/unknown、依据和假设
- data_quality: ID冲突、缺失字段和分类待审项
- limitations: 实际限制

unique_customers 是每簇非空customer_id的去重数量；unknown_customers 是该簇缺customer_id的去重反馈行数，字段名不代表已识别的独立匿名人数。counterevidence 是对象数组，每项包含 feedback_id、text、reason；引用可来自本簇或明确关联的另一簇。

## 交付核验

写后解析JSON，独立复算 input_rows、dedup_rows、各簇已知顾客数与匿名行数，核对全部有效反馈都有归属或明确待审、ID都来自输入。再逐句复核优先级和建议中的成本/工时/实施难度断言：无输入依据且未显式标为有假设的估计时，改为未知或删除。Markdown与JSON一致；不把原话主题整理结果说成已验证的市场需求或真实购买意愿。
