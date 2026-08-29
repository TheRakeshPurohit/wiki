# 图形语法（Grammar of Graphics）在 AI 驱动可视化中的应用与选型

## 一、核心理论：图形语法是什么

《图形语法》（The Grammar of Graphics）由统计学家 Leland Wilkinson 于 1999 年提出，把数据可视化解构为一组**通用、可重用**的组件，而非一系列固定的图表类型。

### 传统图表库 vs 图形语法

- **传统思维（ECharts、Chart.js、ApexCharts）**：图表是固定的死结构（折线图、饼图、柱状图）。遇到非标的复合交互需求，须单独写复杂绘制逻辑，或查阅庞大配置字典（Options Mapping）。
- **图形语法（Vega-Lite、AntV G2/F2）**：所有图表在底层都由相同的核心组件组合而成。通过抽象链式调用或声明式结构，开发者像搭积木一样自由发明新图表。

### 核心积木组件

```
[原始数据 Data] →度量 Scale→ [视觉通道 Aesthetics] →几何标记 Geom→ →坐标系 Coord→ [最终图表]
```

- **度量（Scale）**：连接数据空间与屏幕空间——把输入域（Domain，如销量 0~10000）映射到输出范围（Range，如像素 0~500px 或颜色数组）。
- **几何标记（Geometry / Geom）**：数据在屏幕上的物理形状。常见 Interval（柱状）、Line（线）、Point（点）、Area（面积）等。
- **视觉通道（Aesthetics / Attr）**：把数据字段分配给可见属性——position（位置）、color（颜色）、size（粗细/大小）、shape（形状）。
- **坐标系（Coordinate / Coord）**：几何标记铺开的网格系统——Rect（直角）、Polar（极坐标）。

**颠覆性逻辑（以饼图为例）**：在图形语法里没有"饼图"这种独立图表。饼图 = `Interval`（柱状标记）+ `Polar`（极坐标系）。在直角坐标下把柱子堆叠、再把坐标系拧成圆（极坐标），视觉上就变成饼图或南丁格尔玫瑰图。

## 二、为什么图形语法天然适合 AI 代码生成

在 LLM 自动出图（AI Data Agent）场景，遵循图形语法的库生成准确率大幅超越传统配置驱动引擎：

1. **高维度语义逻辑，消除幻觉**：传统库依赖深层嵌套的庞大 JSON 对象，AI 易在深层嵌套中拼错配置项或丢括号。图形语法是链式或扁平声明——AI 只需理解"将销量映射到高度（`.position('month*sales')`）"的语义映射，理解与输出都轻松。
2. **可预测性，支持"发明"图表**：面对未见过的非标图表，传统库无能为力；图形语法下 AI 像写 Prompt 一样拼装积木，代码即可运行：
   ```js
   chart.point().position('x*y').shape('heart').size('heat').color('sentiment'); // 自动生成"情感爱心散点图"
   ```
3. **数据与视觉完全解耦**：AI 只需两步推理——分析数据特征（连续型/离散型）→ 生成语法通道。这条思考链（Chain of Thought）高度契合大模型的底层逻辑。

## 三、轻量开源方案选型：Vega-Lite vs AntV F2

针对"开源免费、体积小（Bundle 敏感）、适配 AI 生成"的要求：

| 维度 | Vega-Lite（国际标准） | AntV F2（国产移动端） |
|------|-----------------------|----------------------|
| 开发背景 | 华盛顿大学交互数据实验室（微软 PowerBI 底层） | 阿里巴巴蚂蚁金服团队 |
| 开源协议 | BSD | MIT |
| 核心形态 | 纯 JSON 声明式（Spec） | JavaScript 链式调用 / 轻量声明 |
| 体积 | 数百 KB，只做编译、不含繁重渲染 | 极小，专为移动端 / 小程序瘦身 |
| AI 预训练友好度 | 强——主流大模型预训练已吞入大量 Vega-Lite 语料 | 高——中文社区友好，适合国内模型做 Few-shot |
| 最佳场景 | Web 桌面 SaaS、Python 数据中台、AI Data Agent 后端交付 | 微信/支付宝小程序、移动端 H5 运营页、手机端周报看板 |

## 四、被淘汰的竞品与边界

- **ECharts / AntV G2 原版**：含完整渲染引擎（如 ZRender），体积通常 MB 级，对包体积敏感的太重。
- **Chart.js**：体积小，但属配置字典驱动，AI 极易盲猜参数出错。
- **D3.js**：底层操作工具，无图表抽象，AI 需写上百行 DOM 操作，出错率极高。

### ApexCharts：典型的"模版 + 配置字典"流派（非图形语法）

ApexCharts 是一套优秀的**图表模版与配置字典驱动**的传统库。其配置也是巨大的 options JSON，表面与 Vega-Lite 的声明式 JSON 相似，但底层架构与 AI 适配度有本质区别。

**语义区别：调用模版 vs 自由组装**
- ApexCharts：显式指定 `chart.type: 'bar'`，再查该类型下怎么配颜色、怎么配粗细——厚重的配置字典。
- 图形语法：没有柱状/折线的概念，只有 `interval` / `line` 等几何标记；换坐标系配置图表就自行变异，所有图表都是临时组合。

**数据格式区别：行列分离 vs 结构化记录**
- ApexCharts：Series + Labels 架构，数据拆成一列 X（categories）和一列 Y（data）；AI 改图需写复杂算法把原始数据转置、对齐、裁剪，出错率高。
- 图形语法：只吃扁平化 JSON 数组（Tidy Data），无需预处理，AI 只调整 encoding 映射关系。

**横向对比（ApexCharts 7.x vs Vega-Lite）**

| 维度 | ApexCharts | Vega-Lite |
|------|-----------|-----------|
| 颜值与动画 | 默认 UI 现代、动效细腻 | 朴素，学术界/科学计算风格，需调校 |
| 功能经引 | 18+ 常见图表开箱即用 | 靠 mark+encoding 自由拼装，理论无限、常见要自配 |
| 渲染后端 | 纯 SVG（≥万点卡顿） | Canvas/SVG 两栖，海量数据可切 Canvas |
| 包体积 | 曾经近 1MB，新版支持 Tree-shaking | 数百 KB，无沉重渲染包 |
| AI 友好度 | 中——常规图表可，非标复合图表因配置冲突幻觉 | 高——天然契合推理链，国际主流大模型御用底层 |

## 五、工业级落地架构（Vega-Lite 为例）

推荐"AI 只吐 JSON Spec、前端纯干渲染"的安全架构——规避 `eval()` 执行动态 JS。

**AI 侧的结构化 Prompt（模板）**：

> 你是一个数据可视化专家，请严格按 Vega-Lite 的 Spec 规范生成图表 JSON：`[数据源]` … `[图形语法要求]` 用 bar / 将 date 映射 X 轴(nominal) / revenue 映射 Y 轴(quantitative) / region 映射颜色且堆叠 … `[输出约束]` 只输出纯净 JSON，不做 markdown 包裹，不含解释文字。

**AI 稳定输出（纯净 JSON Spec）**：

```json
{
  "data": { "values": [ { "date": "2026-01", "revenue": 120, "region": "North" } ] },
  "mark": "bar",
  "encoding": {
    "x": { "field": "date", "type": "nominal" },
    "y": { "field": "revenue", "type": "quantitative" },
    "color": { "field": "region", "type": "nominal" }
  }
}
```

**前端消费与渲染**：引入轻量 Vega/Vega-Lite 编译分发层，`vegaEmbed('#vis', aiGeneratedSpec)` 即可渲染，天然规避 eval 执行动态代码的安全风险。

## 六、总结与选型落地建议

- **人类手写、追求颜值/动画/开箱即用的管理系统** → 坚持 ApexCharts，用 7.x 核心按需引入控制体积。
- **AI 代理主导、自动解析数据生成不可预测自定义图表的系统** → 切换到 Vega-Lite。

> 注：上文体积、版本号、功能列举基于 2026 年的库版本，具体以官方文档与装机实测为准；引用来源未逐一核验。