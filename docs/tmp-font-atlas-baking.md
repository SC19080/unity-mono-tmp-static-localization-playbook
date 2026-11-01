# 技巧 2：TMP 字形的静态烘焙（双图集 + 对象内容交换）

> 适用：Unity TextMeshPro（3.0.6 / Unity 2022.3 实测）字体资产为**静态烘焙日文图集**，
> 需要显示简体中文。核心结论：**放弃 TMP 的 Dynamic 运行时补字，离线把字形烘焙成
> 静态数据**。本文件记录最终验证过的完整做法与所有实测常量。

---

## 0. 为什么不走 Dynamic 运行时补字（重要结论）

TMP 的 `m_AtlasPopulationMode = Dynamic` 会在运行时把缺字光栅化进图集，
理论上是标准做法。但在目标游戏里实测：

1. 图集纹理不可写（`m_IsReadable=false`）时，FontEngine 的补字**静默失败**
   ——所有缺字显示为方块；
2. 把图集纹理设为可写（`m_IsReadable=true`）后，游戏**启动阶段直接崩溃**
   （Unity 崩溃弹窗），疑似与"可写 + .resS 流式大纹理"组合的加载路径有关；
3. 因此 Dynamic 路线在此游戏是死路。**静态烘焙**（原开发者自己的做法）是唯一可靠路线：
   所有字形离线生成，运行时零字体写入。

> 判定技巧：如果"静态表里有的字显示正常、静态表里没有的字全是方块"，
> 基本可以断定是运行时补字失败，而不是字体覆盖问题。

---

## 1. TMP_FontAsset 的序列化布局（3.0.6 / 2022.3 实测，无 typetree 文件）

MonoBehaviour 原始字节（大字段顺序，均为小端）：

```
[m_FamilyText str][m_StyleText str][m_Version str]
m_SourceFontFile_PPtr (fileID i32 + pathID i64)
m_AtlasPopulationMode i32        # 0=Static 1=Dynamic
m_GlyphTable    : i32 count + count × 52B
m_CharacterTable: i32 count + count × 16B
m_AtlasTextures : i32 count + count × 12B (PPtr<Texture2D>)
m_AtlasTextureIndex i32
m_IsMultiAtlasTexturesEnabled u8
m_ClearDynamicDataOnBuild u8
m_UsedGlyphRects: i32 count + count × 16B (x,y,w,h)
m_FreeGlyphRects: i32 count + count × 16B
m_FontInfo_Legacy ...（旧结构，跳过）
...（kerning / font feature table / fallback 表 / FontAssetCreationSettings /
    m_AtlasWidth i32 / m_AtlasHeight i32 / m_AtlasPadding i32 / m_AtlasRenderMode i32 ...）
```

**Glyph 记录（52 字节）**：

```
index u32
width, height, bearingX, bearingY, advance : f32 ×5   # metrics
rectX, rectY, rectW, rectH : i32 ×4                   # 图集内矩形
scale f32
atlasIndex i32        # 多图集时：该字形位于 m_AtlasTextures 的第几张贴图
class  i32
```

**Character 记录（16 字节）**：

```
const1 u32, unicode u32, glyphIndex u32, scale f32    # 实测首字段恒为 1
```

**关键语义（实测校准）**：

- `metrics.width/height` = **含 SDF 渐变边的单元尺寸**（≈ 墨迹 + 2×padding），
  不是裸墨迹；
- `bearingX/bearingY` = 该**单元**相对基线/原点的位置（bearingY 向上为正）；
- `advance` = 步进（像素，pointSize 以 72dpi 计，1pt = 1px）；
- `rect` 是该字形在图集纹理里的采样矩形。

> 以上布局建议先在目标文件上"自证"：解析若干已知字符（如 'A'、空格、常用汉字），
> 检查 metrics/rect/atlasIndex 的取值是否自洽（rect 在图集内、scale=1、rect≈metrics+1）。

---

## 2. 烘焙流水线

### 2.1 收集待烘焙字符

```
need = (程序集现版全部字符串 ∪ 程序集原版全部字符串 ∪ 场景/预制体烘焙文本)
       − 字体静态字符集（从资产里的字符集串解析，形如 "20-7E,3041-3093,..."）
       ∩ 子集字体 cmap     # 确保源字体真的有这个字形
```

同时把场景/预制体里烘焙文本（UTF-16/UTF-8 双格式扫描）的用字纳入覆盖统计。

### 2.2 渲染 + SDF 生成

- FreeType：`set_pixel_sizes(0, pointSize)`（**1pt = 1px，72dpi**，
  可用既有静态字形的 advance 反推验证），`FT_LOAD_RENDER` 得到 8bit AA 墨迹位图；
- 以墨迹为中心取单元：`cell = ink + 2×MARGIN`（MARGIN=7）；
- **chamfer SDF**（两遍扫描 + 1/√2 对角代价，纯 Python 足够）：
  对单元内每个 texel 计算 `sd = d(到外部) − d(到内部)`（内部为正）；
- 编码：`alpha = 128 + clamp(sd / spread, -1, 1) × 127`，
  实测原版图集 `spread ≈ 6.6px`（边缘 128、内部 255、外部 0）。
  **务必从既有字形扫描线拟合确认 spread**，不同资产可能不同。

### 2.3 装箱：不要相信序列化的 free list！

原版资产的 `m_FreeGlyphRects` 实测**互相重叠 7406 对、与 used 区重叠 18203 对、
且总量严重虚报**（声称 14.8M px 空闲，实际静态字形已占 97%）。
按它装箱会产出**叠字**（多个字形像素印进同一区域）。

正确做法：**占用网格**（`bytearray(W*H)`，静态 used rects 标记为 1），
全局单调扫描找第一块全空闲的 `w×h` 区域，放置后标记。
网格保证：与静态字形零重叠、与已放置字形零重叠。

> 容量核算：静态字形实际占用 = used rects 面积和 ÷ 图集面积。
> 实测原版 8192² 图集占用 **97%**，仅剩 1.8M px —— 装不下 597 个全尺寸中文字形
> （需要 ~6M px）。这正是需要**第二图集**的原因。

### 2.4 写像素：注意行序

Unity 纹理**数据行 0 = 图像底部**。字形单元的写入：

```
for k in range(cell_h):
    写到 数据行 (rect.y + cell_h - 1 - k) 的 [rect.x, rect.x + cell_w)
```

**校准方法**：先从图集抠一个已知静态字（如笔画复杂的常用汉字），
按"直接读取"与"上下翻转"各导出 PNG，人工确认哪个是正的；再校验 SDF 剖面
（沿中线打印像素值：外部 0 → 渐变 → 内部 255）。

像素写在 `.resS` 流式数据区：`offset = m_StreamData.offset + data_row × width + x`，
用 `r+b` 随机 seek 写入即可（块大小不变，无需重写整个 .resS）。

### 2.5 追加表记录 + 双图集

- 追加 597 条 Glyph 记录（`atlasIndex=1`）与 Character 记录；
- **字形 ID 空间分段**：静态字形 index 已 `+100000`（100001~107864），
  烘焙字形 index = 子集字体 glyphID `+200000`（200001~206700）。
  **两段绝不能重叠**：TMP 按 glyphIndex 在字典里查重并取**靠前的记录**，
  重叠时烘焙的中文会被静态表的日文字形顶掉（症状：部分字显示成别的字）；
- `m_AtlasTextures` 数组 1→2（追加一张已无人引用的旧图集纹理作为第二图集），
  新字形的 `atlasIndex=1`；像素写进该纹理自己的 .resS 区域；
- `m_AtlasPopulationMode` 设回 **Static**。

> 第二图集的选择：找一张**内容交换后已无人引用**的同尺寸同格式图集纹理
> （例如被内容交换覆盖掉的另一字体资产的旧图集）。
> 运行时 TMP 对多图集字形的非主图集字形会自动生成子网格与回退材质，属于标准功能。

---

## 3. 对象内容交换（不碰场景文件的引用重定向）

场景里有 ~1000 个组件引用字体资产 A（其原字形装不下中文），而字体资产 B
（另一字体）已具备全量字形。两种改法：

| 方案 | 风险 |
|---|---|
| 字节替换场景文件里的 PPtr | **小整数 pathID（2/3/162…）与网格/顶点等二进制数据大量撞车**，盲目替换会毁掉模型/贴图数据 |
| **对象内容交换** ✅ | 把 B 的数据整段复制进 A 的对象槽位（`set_raw_data`），pathID 不变，场景文件零改动 |

```
fontAssetSlotA.data ← fontAssetSlotB.data   # 字体：全量字形版本
materialSlotA.data ← materialSlotB.data     # 材质：指向全量图集的材质
```

注意：交换后多个字体资产共享同一材质/图集——在 **Static 模式**下没有运行时写入，
共享是安全的。副作用是被替换字体的原始样式消失（本例 UI 从黑体变衬线），
以及共享图集的容量按最高占用者计算。

---

## 4. 验证

1. **结构**：字形表↔字符表互引 0 缺失；glyphIndex 无重复；与静态 ID 空间 0 冲突；
   `atlasIndex ∈ {0,1}`；`m_AtlasTextures` 数量与实际纹理一致；
2. **覆盖**：目标语言全部字符 ∈ (静态字符集 ∪ 烘焙集)，缺口 = 0；
3. **目视**：从图集抠出抽样字形（含烘焙字形）渲染成 PNG——
   确认方向正确、形状正确、无叠字、SDF 渐变与原版一致；
4. **容量**：占用网格统计实际占用率 < 100%；
5. 资源完整性：对象表数量/pathID 一致、流式偏移在界内、未改动的文件与备份逐字节一致。

---

## 5. 环境坑

- **Windows 非 ASCII 路径**：freetype-py 打不开含日文/中文的路径 →
  `freetype.Face(io.BytesIO(open(path,'rb').read()))` 从内存加载；
- UnityPy 的**类型化保存**（read + 改字段 + save）对本例 Texture2D 是逐字节忠实的
  （实测仅目标 bool 变化），但**但凡能用 `set_raw_data` 原始字节编辑就不要用类型化保存**；
- `env.file.save()` 会重写整个 assets 文件：改前必须备份；改后必须重新解析回读；
- 字体子集用 fontTools 裁剪（含 GB2312 + 程序集现版/原版全部用字 + 场景烘焙文本用字），
  并遵守 OFL（附带许可证）。
