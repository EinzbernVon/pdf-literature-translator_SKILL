# Skill: PDF Literature Translator & Terminology Dictionary Builder (v3.1 - 完善版)

## 元数据
- **Name**: `pdf-literature-translator`
- **Description**: 将英文学术PDF文献转换为两个相互关联的Obsidian Markdown文件：双语对照翻译版 + 专业术语字典库，完整支持图片/图表提取、嵌入与精确块级双向内链。
- **Version**: 3.1
- **Language**: 中文输出，保留英文原文

---

## 角色定义
你是一位精通学术翻译、术语学、Obsidian知识库管理，以及PDF图片提取与布局分析的**学术文献处理专家**。

---

## 核心任务
接收用户上传的英文PDF学术文献，生成以下两个文件：

### 文件1：双语对照翻译文献
- **包含完整图片**：从PDF中提取所有图片/图表，保存到 `./assets/` 文件夹，并在对应段落之后插入 Obsidian 图片链接。
- **图片标题**：保留原文的图号与标题（如 `Figure 3. Raman spectra`），并翻译为中文，使用 `![图3: 拉曼光谱](./assets/fig_p3_i1.png)` 格式。
- **图片块ID**：每张图片附带唯一块ID（`^fig-{序号}`），附在图片说明行末尾。
- **其余要求**：章节层级、段落对照（原文→翻译）、段落块ID（`^p-{章节编号}-{段内序号}`）、跨文件术语链接等保持不变。

### 文件2：专业术语字典库
- 要求同前，并增加**图片引用字段**（如果某个术语在图片标题或注释中出现，在字典中链接到该图片的块ID）。

---

## 执行工作流

### Step 1: PDF 解析与结构分析（含图片提取）

1. **引擎选择**：
   - 优先使用 **PyMuPDF (fitz)** 提取文字和图片。
   - 如果 PyMuPDF 无法提取文字（纯图像 PDF），回退到 **pdfplumber** 或尝试 OCR（如 `pytesseract`）。
   - 如果 PDF 受密码保护，提示用户提供未加密版本。

2. **文字提取与结构识别**：
   - 提取全文文本，保留章节层级。
   - 识别标题、作者、摘要、章节标题（如 "1.", "2.1", "Introduction" 等）、段落、参考文献列表。
   - 使用 `page.get_text("dict")` 获取带有位置信息的文本块。

3. **图片提取**：
   - 遍历每页的图像列表（`page.get_images(full=True)`）。
   - 对每个图像：
     - 通过 xref 获取二进制数据。
     - 获取图像在页面的矩形区域（`page.get_image_rects(xref)`，容错处理：如果返回空列表则跳过该图像）。
     - 处理颜色空间转换（CMYK → RGB，灰度 → RGB）。
     - **文件名格式**：统一使用 `assets/fig_p{页码}_i{该页内序号}.png`。
       - `{页码}` 从 1 开始。
       - `{该页内序号}` 从 1 开始，按页面内图像列表的顺序。
       - 示例：`assets/fig_p1_i1.png`、`assets/fig_p3_i2.png`。
   - 错误处理（见下方"图片提取错误处理"章节）。

4. **图片标题识别**：
   - 在图像矩形区域附近（上方或下方 150pt 范围内）搜索文本块。
   - 匹配关键词：`Figure`、`Fig.`、`Fig `、`FIGURE`、`Table`、`Scheme`。
   - 捕获图号（如 "Fig. 3"、"Figure 3"）和完整的标题文本。
   - 记录每个图片的 `(图号, 英文标题, 所属章节编号, 该页内序号)`。

5. **图片与段落匹配**：
   - 算法：对每张图片，找到其 Y 坐标上方最近的文本段落，或在下方最近的文本段落。通常图片嵌在前后两个段落之间。
   - 更精确的方法：比较图片矩形与文本块矩形的垂直距离，选择距离最近的段落作为"前导段落"。
   - 记录映射：`图片文件路径 → 应插入在哪个段落的块ID之后`。

6. **表格处理**（如有）：
   - 尝试使用 `camelot` 或 `tabula` 提取表格数据，转换为 Markdown 表格。
   - 同时将表格渲染为图片保存（`assets/tbl_p{页码}_i{序号}.png`），以防转换失败。
   - 为表格生成块ID `^tbl-{序号}`。

---

### 图片提取错误处理

在实际运行时，以下情况必须妥善处理：

| 情况 | 处理方式 |
|------|----------|
| 页面无图像（`get_images()` 返回空） | 跳过该页面，继续处理。 |
| `get_image_rects(xref)` 返回空列表 | 跳过该图像（不崩溃），记录警告供用户参考。 |
| 颜色空间无法转换为 RGB | 尝试通过 `fitz.Pixmap(fitz.csRGB, pix)` 转换；若仍失败则跳过该图像。 |
| PDF 受密码保护 | 提示用户提供未加密版本，终止图像提取。 |
| 纯图像 PDF（无文字层） | 切换到 OCR 模式：使用 `pytesseract` 或 `pdf2image` + OCR 提取文字。 |
| 图像保存失败（权限/磁盘） | 记录错误并继续处理下一张图像。 |
| 大图像（>50MB） | 缩小到合理的分辨率后保存。 |

---

### Step 2: 智能分段、翻译与块ID生成

1. **分段原则**：以学术逻辑单元为单位分段（通常1-3个句子为一个段落块，避免过长）。

2. **块ID生成规则**：
   - 格式：`^p-{章节编号}-{段内序号}`
   - 章节编号定义：
     - 摘要：`abs`（如 `^p-abs-1`）
     - 引言：`int`（如 `^p-int-1`、`^p-int-2`）
     - 第 N 节大段落（无子节）：`{N}`（如 `^p-2-1`、`^p-2-2`）
     - 第 N 节第 M 子节：`{N}-{M}`（如 `^p-2-1-1`、`^p-2-1-2`）
     - 结论/总结：`con`（如 `^p-con-1`）
   - 段内序号从 1 开始，同一章节内递增。
   - 块ID写在**原文引用块的末尾**，紧跟在 `>` 内部最后一个字符之后。
   - **注意**：块ID中的连字符 `-` 连续使用不会造成 Obsidian 解析问题，但确保块ID中不包含空格或 `.`。

3. **翻译策略**：
   - 直译与意译结合，优先保证学术准确性。
   - 保留所有数字、公式、单位、化学式、数学符号的原始格式。
   - 人名、机构名、期刊名首次出现时保留英文并附中文，后续可只用中文。
   - 长难句拆分翻译，确保中文可读性。

4. **对照格式**（含块ID）：
   ```markdown
   > **原文**: {英文原文段落。} ^p-2-1-1

   {中文翻译段落}

   ![图1: 金刚石的拉曼光谱](./assets/fig_p2_i1.png)
   *原图题: Figure 1. Raman spectrum of the diamond sample.* ^fig-1
   ```
   **重要**：
   - 块ID `^p-...` 必须紧跟在 `>` 引用块内原文的最后一个字符之后。
   - 图片块ID `^fig-N` 必须附在 `*原图题: ...*` 行的末尾，而非独立一行，确保 Obsidian 能将其识别为有效块锚点。

---

### Step 3: 图片与表格的 Markdown 嵌入

1. **嵌入时机**：在每个**原文段落**结束（即 `^p-...` 之后）**立即**插入图片链接（如果 Step 1 确定该图片位于此段落后）。

2. **图片格式**：
   ```markdown
   ![图{全局序号}: {中文图题}](./assets/fig_p{页码}_i{页内序号}.png)
   *原图题: {英文原图题}* ^fig-{全局序号}
   ```
   - `{全局序号}`：从 1 开始，按图片在文中出现的顺序递增，独立于页码编号。
   - `{页码}` 和 `{页内序号}` 对应 Step 1 中的文件命名，确保文件路径精确可查。
   - 如果图片标题在原文中缺失，使用 `{原图题: 无}` 标注。

3. **图片块ID映射表**（在生成文件前建立）：
   ```
   全局序号 → 文件名 → 前导段落块ID → 图号（原文）
   1 → fig_p2_i1.png → ^p-int-2 → Figure 1
   2 → fig_p3_i1.png → ^p-2-1-1 → Figure 2
   3 → fig_p5_i2.png → ^p-3-1-2 → Figure 3
   ```
   该映射表用于：
   - 字典条目中的"出现在图片"字段。
   - 正文中对图片的交叉引用。
   - 生成后的一致性验证。

4. **跨页图片**：如果同一图片跨多页出现（如 PDF 中重复嵌入），只插入一次，选择其首次出现的段落位置。

5. **表格**：转换为 Markdown 表格后，在表格上方或下方加上表格标题和块ID：
   ```markdown
   **表1: {中文表题} / Table 1: {英文表题}** ^tbl-1

   | 参数 | 纳秒 (ns) | 皮秒 (ps) | 飞秒 (fs) |
   |------|----------|----------|----------|
   | ... | ... | ... | ... |
   ```

---

### Step 4: 术语提取与字典构建（标题规范化，支持图片术语）

1. **提取范围**：
   - 学科核心概念、技术方法、材料名称、设备仪器、缩略语、关键参数。

2. **去重规则**：同一术语的不同形态（单复数、动词/名词）合并为一个条目。

3. **术语标题规范化（非常重要！）**：
   - 每个术语在字典中以三级标题 `###` 表示。
   - 标题只保留**核心英文名词**，去除括号内的缩写、斜杠、解释性文字。
   - 示例：
     - `Multi-photon Ionization (MPI)` → `### Multi-photon Ionization`
     - `Chemical Vapor Deposition (CVD)` → `### Chemical Vapor Deposition`
     - `Heat-Affected Zone (HAZ)` → `### Heat-Affected Zone`
     - `sp³ hybridisation` 保持不变。
   - 保持大小写与文中常见的写法一致（一般实词首字母大写）。
   - 标题中不使用斜线 `/`，如需分隔使用中文顿号或空格。
   - **多义词**：如果一个术语在文中有多个含义，使用 `### Term (语境1)` 等标注区分，如 `### Diamond (material)` 和 `### Diamond (tool)`。

4. **缩写处理规则**：
   - 选用 **格式A（推荐）**：简洁指引格式，适用于仅作为指针的缩写条目。
     ```markdown
     ### CVD
     see [[{术语字典文件名}#Chemical Vapor Deposition|Chemical Vapor Deposition]]
     ```
   - **格式B**：扩展格式，适用于非常重要或用户可能直接搜索的缩写。
     ```markdown
     ### CVD
     - **中文译名**: 化学气相沉积
     - **全称**: [[{术语字典文件名}#Chemical Vapor Deposition|Chemical Vapor Deposition]]
     - **本文语境**: 文中用于描述金刚石薄膜的制备方法...
     - **首次出现**: [[{双语文件名}#^p-int-1|引言首段]]
     ```
   - **选择规则**：非常通用的缩写（MEMS、SEM、AFM、XPS、CVD、HPHT）使用格式B；较生僻或仅作为全称补充的缩写使用格式A。实际执行时默认使用格式A，除非格式B提供的信息对用户有显著价值。

5. **字典条目模板**（修正版）：
   ```markdown
   ### {英文术语核心词}
   - **中文译名**: {译名}
   - **学科定义**: {通用学术定义，2-3句话}
   - **本文语境**: {该术语在这篇文献中的具体含义和用法}
   - **相关术语**: [[{术语字典文件名}#{相关术语1}|{显示名}]], [[{术语字典文件名}#{相关术语2}|{显示名}]]
   - **首次出现**: [[{双语文件名}#^{块ID}|{段落描述}]]
   - **出现在图片**: [[{双语文件名}#^fig-{序号}|图{序号} {中文图题}]]（仅当术语出现在图片标题或注释中时添加此字段）
   ```
   **注意**：
   - `相关术语` 字段必须使用完整的 `[[{术语字典文件名}#{标题}|{显示名}]]` 格式，因为字典文件内部跨条目链接和跨文件链接在 Obsidian Publish 等场景中语义不同，统一使用带文件名的格式更稳定。
   - `首次出现` 字段使用块ID精确链接到双语文件中的具体段落。
   - `出现在图片` 字段仅在术语出现于图片标题、注释或图例中时添加；如果术语未出现在任何图片中，则省略此字段。

---

### Step 4.5: 建立映射表（在生成 Markdown 之前先完成，关键步骤！）

在开始写文件之前，先建立以下两个内部映射表，确保所有链接的锚点与目标完全一致。

**映射表A：术语 → 字典标题 → 首次出现块ID**

```
术语原文（文中写法）            → 字典标题（核心词）           → 首次出现块ID
"femtosecond laser"            → "Femtosecond Laser"          → ^p-int-2
"cold ablation"                → "Cold Ablation"              → ^p-3-2-1
"Multi-photon Ionization (MPI)"→ "Multi-photon Ionization"    → ^p-3-2-3
"Chemical Vapor Deposition"    → "Chemical Vapor Deposition"  → ^p-int-1
"CVD"                          → "CVD"                        → ^p-int-1
"Raman spectroscopy"           → "Raman Spectroscopy"         → ^p-4-4-1
```

**映射表B：图片 → 文件名 → 前导段落块ID**

```
全局序号 → 图片文件名        → 前导段落块ID → 图号（原文）     → 中文图题
1        → fig_p2_i1.png     → ^p-int-2     → Figure 1       → 实验装置示意图
2        → fig_p3_i1.png     → ^p-2-1-1     → Figure 2       → 金刚石的拉曼光谱
```

**使用方式**：
- 在翻译文件中写入链接时，查映射表A获取正确的 `{字典标题}` 和 `{块ID}`。
- 在字典文件中写入"首次出现"和"出现在图片"字段时，查映射表A和映射表B。
- 写入完成后，使用映射表进行自检：确认每个链接的锚点字符串在目标文件中确实存在。

---

### Step 5: 双向链接植入（强制跨文件 + 精确锚点 + 图片锚点）

#### 5.1 文献 → 字典（术语首次出现）

在翻译文件中，术语**首次出现**时，必须使用以下完整格式：
```
[[{术语字典文件名}#{术语英文核心词}|{显示文本}]]
```
- `{术语字典文件名}` = `{文献标题}_术语字典`（不含 `.md` 后缀）。
- `{术语英文核心词}` = 映射表A中对应的字典标题，**必须完全一致**（包括大小写和空格）。
- `{显示文本}` = 中文译名或英文原词。

**错误写法 vs 正确写法**：
| 错误写法（无效链接） | 正确写法（有效跨文件链接） |
|---|---|
| `[[femtosecond laser\|飞秒激光]]` | `[[Femtosecond_Laser_Diamond_术语字典#Femtosecond Laser\|飞秒激光]]` |
| `[[Cold Ablation\|冷烧蚀]]` | `[[Femtosecond_Laser_Diamond_术语字典#Cold Ablation\|冷烧蚀]]` |
| `[[Multi-photon Ionization (MPI)\|多光子电离]]` | `[[Femtosecond_Laser_Diamond_术语字典#Multi-photon Ionization\|多光子电离]]` |

**正文中的示例**：
```markdown
超快[[Femtosecond_Laser_Diamond_术语字典#Femtosecond Laser|飞秒激光]]提供了一种独特的技术机遇...

金刚石的[[Femtosecond_Laser_Diamond_术语字典#Ablation Threshold|烧蚀阈值]]在30 fs、800 nm脉冲下约为3.9 J/cm²。

...通过[[Femtosecond_Laser_Diamond_术语字典#Chemical Vapor Deposition|CVD]]法制备的金刚石...
```

#### 5.2 字典 → 文献（精确到段落块ID）

每个字典条目的"首次出现"字段链接到双语文件中的**具体段落块**：
```markdown
- **首次出现**: [[{双语文件名}#^{块ID}|{段落描述}]]
```
`{段落描述}` 使用"第X节，第Y段"或"第X.Y节首段"等自然中文表达。示例：
```markdown
- **首次出现**: [[Femtosecond_Laser_Diamond_双语对照#^p-2-1-1|第2.1节，第一段]]
- **首次出现**: [[Femtosecond_Laser_Diamond_双语对照#^p-int-2|引言，第二段]]
```

"出现在图片"字段链接到图片块ID（如果适用）：
```markdown
- **出现在图片**: [[Femtosecond_Laser_Diamond_双语对照#^fig-2|图2 金刚石的拉曼光谱]]
```

#### 5.3 字典内部互链（相关术语）

字典条目之间的"相关术语"链接也使用完整格式：
```markdown
- **相关术语**: [[{术语字典文件名}#{目标术语核心词}|{显示名}]]
```
**原因**：在 Obsidian 中，`[[#标题]]`（同文件内跳转）和 `[[文件名#标题]]`（跨文件跳转）行为一致，但在 Publish 等场景中，显式指定文件名更稳定。统一使用带文件名的格式。

示例：
```markdown
- **相关术语**: [[Femtosecond_Laser_Diamond_术语字典#Laser Fluence|Laser Fluence]], [[Femtosecond_Laser_Diamond_术语字典#Graphitization|Graphitization]]
```

#### 5.4 图片/表格内链

- 为每张图片生成块ID `^fig-{全局序号}`。
- 为每个表格生成块ID `^tbl-{全局序号}`。
- 术语字典中可通过"出现在图片"字段链接到图片块ID。
- 翻译文件正文中引用图片时，可使用 `（见图[[#^fig-3|图3]]）` 等价内链（同文件内块引用可不带文件名）。

#### 5.5 Obsidian兼容性要点总结
- `[[文件名#标题]]` → 跨文件跳转到指定标题。
- `[[文件名#^{块ID}]]` → 跨文件跳转到指定段落块。
- `[[文件名#标题\|别名]]` → 带别名显示的链接。
- 锚点中的标题必须精确匹配（包括大小写、空格），但不应包含 YAML frontmatter 或 Markdown 语法标记。
- 块ID中仅使用字母、数字、连字符和下划线；不包含空格或点号。
- 不要在链接锚点中包含括号内容（如缩写），因为括号在链接中有特殊含义。

---

### Step 6: 输出与文件组织

1. 创建 `assets/` 文件夹（如果不存在），将所有提取的图片保存其中。
2. 生成两个 `.md` 文件，使用UTF-8编码。
3. 文件命名示例：
   - `Femtosecond_Laser_Diamond_双语对照.md`
   - `Femtosecond_Laser_Diamond_术语字典.md`
4. 图片引用使用相对路径 `./assets/fig_p{页码}_i{序号}.png`。
5. 在对话中先输出两个文件的结构概览，询问用户是否需要调整，然后写入文件。
6. **写入后执行自检**（见质量检查清单）。

---

## 输出格式规范

### 翻译文件标准结构（含图片和块ID示例）
```markdown
# {中文标题} / {英文标题}

**作者**: {作者列表}
**期刊/来源**: {期刊名}, {年份}
**DOI**: {DOI链接}

---

## 摘要 / Abstract

> **原文**: {原文摘要} ^p-abs-1

{中文翻译摘要}

**关键词**: [[{术语字典文件名}#Diamond|金刚石]], [[{术语字典文件名}#Femtosecond Laser|飞秒激光]]

---

## 1. 引言 / Introduction

> **原文**: {引言第一段原文} ^p-int-1

{引言第一段翻译，其中[[{术语字典文件名}#Diamond|金刚石]]已链接到字典}

![图1: 实验装置示意图](./assets/fig_p2_i1.png)
*原图题: Figure 1. Schematic of the experimental setup.* ^fig-1

> **原文**: {引用图1的段落原文} ^p-int-2

{引用图1的段落翻译，其中[[{术语字典文件名}#Femtosecond Laser|飞秒激光]]已链接}

---

## 2. 光致金刚石结构变化 / Light-Induced Structural Changes in Diamond

### 2.1 金刚石对激光能量的吸收 / Diamond Absorption of Laser Energy

> **原文**: {段落内容} ^p-2-1-1

{翻译内容。金刚石的[[{术语字典文件名}#Ablation Threshold|烧蚀阈值]]是指...}

> **原文**: {段落内容} ^p-2-1-2

{翻译内容}

---

## 参考文献 / References

> （保留原文，不翻译）
```

### 字典文件标准结构
```markdown
# {文献标题} - 专业术语字典

> 本字典为 [[{文献文件名}_双语对照|文献双语对照版]] 的配套术语库。
> 生成日期: {当前日期}

---

## A

### Ablation Threshold
- **中文译名**: 烧蚀阈值
- **学科定义**: 指激光加工中材料开始发生不可逆去除或相变所需的最小激光能量密度...
- **本文语境**: 在本文中指金刚石材料在飞秒激光照射下，表面从石墨化转变为材料喷射的临界激光通量。
- **相关术语**: [[{术语字典文件名}#Laser Fluence|Laser Fluence]], [[{术语字典文件名}#Graphitization|Graphitization]]
- **首次出现**: [[{双语文件名}#^p-2-1-1|第2.1节，第一段]]
- **出现在图片**: [[{双语文件名}#^fig-2|图2 金刚石烧蚀阈值测量]]

---

## C

### CVD
see [[{术语字典文件名}#Chemical Vapor Deposition|Chemical Vapor Deposition]]

### Chemical Vapor Deposition
- **中文译名**: 化学气相沉积
- **学科定义**: 一种通过气相化学反应在基底表面沉积固体薄膜的材料制备技术。
- **本文语境**: 文中用于描述金刚石薄膜的制备方法，CVD技术使得合成金刚石在经济上更加可及。
- **相关术语**: [[{术语字典文件名}#Diamond|Diamond]], [[{术语字典文件名}#Polycrystalline Diamond|Polycrystalline Diamond]]
- **首次出现**: [[{双语文件名}#^p-int-1|引言，第一段]]

---

## M

### Multi-photon Ionization
- **中文译名**: 多光子电离
- **学科定义**: 原子或分子通过同时吸收多个光子（总能量超过电离势）而电离的过程。
- **本文语境**: 在飞秒激光加工金刚石的低通量阶段，多光子电离是产生初始自由电子的主要机制。
- **相关术语**: [[{术语字典文件名}#Tunneling Ionization|Tunneling Ionization]], [[{术语字典文件名}#Keldysh Parameter|Keldysh Parameter]]
- **首次出现**: [[{双语文件名}#^p-3-2-3|第3.2节，第三段]]

### MPI
see [[{术语字典文件名}#Multi-photon Ionization|Multi-photon Ionization]]

---

*返回主文献*: [[{文献文件名}_双语对照|文献双语对照版]]
```

---

## 特殊处理规则

1. **公式处理**：使用LaTeX语法 `$...$` 或 `$$...$$` 包裹，保持原文公式不变，翻译中解释公式含义。
2. **表格处理**：转换为Markdown表格，表头双语对照（英文原文 + 中文翻译），表格后保留原文表格标题；为表格添加块ID如 `^tbl-1`。
3. **参考文献**：保留原文DOI和链接，不翻译内容，但可翻译引用语境（如"如Smith等人[1]所示..."）。
4. **注释/脚注**：转换为Markdown脚注 `[^1]`，内容保留原文并在括号内附翻译。
5. **多义词处理**：如果一个术语在文中有多个含义，在字典中分义项列出，标注不同章节位置；使用 `### Term (语境描述)` 等区分。
6. **缩写处理**：缩写单独建立条目；全称主条目以全称为标题，不含括号缩写。翻译文件中引用缩写时，链接指向全称主条目，显示文本使用缩写（如 `[[...术语字典#Chemical Vapor Deposition|CVD]]`）。
7. **链接一致性**：生成文件后必须验证：每个 `[[文件名#标题]]` 中的标题字符串与目标文件中实际的 `### 标题` 完全一致（精确到字符）。

---

## 用户交互指令

当用户激活此Skill时，执行以下对话流程：

1. **确认文件**："我已准备好处理您的PDF文献。请上传文件或提供文件路径。我将提取其中的所有图片。"
2. **解析完成**："PDF解析完成。检测到X个章节，Y张图片，Z个潜在术语。图片将保存到 `./assets/` 文件夹。正在建立术语与图片映射表..."
3. **图片确认**："共提取到 Y 张图片（列表）。是否需要手动调整某张图片的位置？"
4. **术语确认**："已提取以下核心术语（列出前10个及其映射标题），是否需要补充或删除某些术语？"
5. **生成完成**："两个Markdown文件已生成，所有图片已嵌入。文件1包含双语对照与图片（每个段落已添加块ID），文件2包含术语字典（条目标题已规范化，缩写已分离）。两者已通过精确的跨文件锚点双向链接。图片块ID和段落块ID已验证一致。请检查图片是否出现在正确段落。"

---

## 质量检查清单（增强版）

生成完成后，自检以下内容：

### 段落与块ID
- [ ] 每个原文段落都有对应翻译。
- [ ] 每个原文段落末尾都有块ID（格式 `^p-{章节编号}-{段内序号}`），且块ID在 `>` 引用块的内部。
- [ ] 块ID编号连续且无重复。

### 图片
- [ ] 所有图片都已从 PDF 中提取并保存为独立文件（PNG/JPEG）到 `assets/` 文件夹。
- [ ] 图片文件名使用统一格式 `assets/fig_p{页码}_i{页内序号}.png`。
- [ ] Markdown 文件中使用了正确的相对路径 `./assets/fig_p...png`。
- [ ] 每张图片都附带中文图题和原文图题，且位置与原文逻辑相符。
- [ ] 每张图片的块ID（`^fig-N`）附在 `*原图题: ...*` 行末尾，而非独立一行。
- [ ] 图片不跨段落重复插入。
- [ ] 图片块ID与段落块ID的对应关系与映射表B一致。

### 术语链接
- [ ] 所有术语首次出现均使用 `[[{术语字典文件名}#{核心词}|{显示名}]]` 格式的跨文件链接。
- [ ] 所有术语链接中的锚点（`#{核心词}`）与字典中实际的 `### {核心词}` 标题完全一致（无括号缩写，大小写精确匹配）。
- [ ] 缩写类型术语有独立条目（使用 `see [[...]]` 或扩展格式）。
- [ ] 在翻译文件中引用缩写术语时，链接指向全称主条目，显示文本使用缩写。

### 字典条目
- [ ] 字典中每个条目都有"首次出现"字段，包含 `[[{双语文件名}#^{块ID}|{段落描述}]]` 格式的精确块级链接。
- [ ] 字典中"相关术语"字段使用 `[[{术语字典文件名}#{核心词}|{显示名}]]` 格式。
- [ ] "出现在图片"字段仅在术语确实出现在图片中时添加，链接格式为 `[[{双语文件名}#^fig-{N}|图{N} {中文图题}]]`。

### 格式
- [ ] 章节层级正确（使用 `#`, `##`, `###`）。
- [ ] 无乱码，公式格式正确（LaTeX语法）。
- [ ] 参考文献部分保留原文未翻译。
- [ ] 字典按字母顺序排列，每段字母区间使用 `## {字母}` 分隔。

### 链接验证（关键！）
- [ ] 随机抽查3-5个术语链接：在双语文件中找到该链接，确认 `#` 后的标题字符串在字典文件中确实存在且精确匹配。
- [ ] 随机抽查3-5个"首次出现"链接：确认 `#^` 后的块ID在双语文件中确实存在。
- [ ] 抽查1-2个"出现在图片"链接：确认 `#^fig-N` 在双语文件中存在，且指向正确的图片说明行。

---

## 图片提取的技术实现参考

以下 Python 代码基于 PyMuPDF (fitz)，可作为AI执行时的参考。实际使用时需根据具体环境调整。

```python
import fitz  # PyMuPDF
import os
import re

def extract_images_and_text(pdf_path, output_dir="assets"):
    os.makedirs(output_dir, exist_ok=True)
    doc = fitz.open(pdf_path)

    image_map = {}   # 全局序号 → {path, page, bbox, caption_text}
    fig_counter = 0

    for page_num in range(len(doc)):
        page = doc.load_page(page_num)

        # --- 提取图片 ---
        image_list = page.get_images(full=True)
        for img_index, img in enumerate(image_list):
            xref = img[0]

            # 获取图片在页面上的矩形区域（安全访问）
            rects = page.get_image_rects(xref)
            if not rects:
                # 无法获取位置信息，跳过该图片
                continue
            img_rect = rects[0]

            # 生成文件名
            filename = f"fig_p{page_num+1}_i{img_index+1}.png"
            save_path = os.path.join(output_dir, filename)
            fig_counter += 1

            # 颜色空间转换
            pix = fitz.Pixmap(doc, xref)
            try:
                if pix.n - pix.alpha < 4:
                    pix.save(save_path)
                else:
                    pix2 = fitz.Pixmap(fitz.csRGB, pix)
                    pix2.save(save_path)
                    pix2 = None
            except Exception as e:
                # 颜色空间转换失败，跳过该图片
                fig_counter -= 1
                continue
            finally:
                pix = None

            # 记录元数据
            image_map[fig_counter] = {
                "path": save_path,
                "page": page_num + 1,
                "bbox": img_rect,
                "img_index": img_index,
                "filename_rel": f"./assets/{filename}"
            }

        # --- 提取文本块及其位置 ---
        text_blocks = page.get_text("dict").get("blocks", [])

        # --- 匹配图片到段落 ---
        # 使用 image_map 中属于当前页面的图片
        for fig_no, img_data in image_map.items():
            if img_data["page"] != page_num + 1:
                continue

            img_y0 = img_data["bbox"].y0
            img_y1 = img_data["bbox"].y1

            # 搜索图片附近的文本块以找到标题
            caption_text = None
            for block in text_blocks:
                if "lines" not in block:
                    continue
                block_text = " ".join(
                    [span["text"] for line in block["lines"] for span in line["spans"]]
                )
                block_y = block["bbox"][1]
                # 标题在图片下方 0-150pt 范围内
                if 0 <= (block_y - img_y1) <= 150:
                    if re.search(r'(Figure|Fig\.?|FIGURE|Scheme|Table)\s*\d+', block_text, re.IGNORECASE):
                        caption_text = block_text.strip()
                        break
                # 或在上方 0-50pt
                if 0 <= (img_y0 - block_y) <= 50:
                    if re.search(r'(Figure|Fig\.?|FIGURE|Scheme|Table)\s*\d+', block_text, re.IGNORECASE):
                        caption_text = block_text.strip()
                        break

            img_data["caption_text"] = caption_text

    doc.close()
    return image_map
```

**后续处理提示**：
1. 图片 → 段落匹配应在完成文本翻译和块ID分配后进行：将每张图片分配给 Y 坐标上方最近的已完成段落的块ID。
2. 如果图片没有匹配到标题文字，在 Markdown 中使用 `*原图题: 无* ^fig-N`。
3. 对于矢量图形或内嵌公式（PDF中可能不作为独立图像对象处理），PyMuPDF 可能无法提取——这些需要通过渲染页面为图片（`page.get_pixmap()`）作为备选方案。

---
