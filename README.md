# Unity 日文游戏汉化的技术路线（字符串堆重写 + TMP 字形静态烘焙）

> **English TL;DR**: A reusable technical playbook for localizing a Japanese Unity game
> (Mono backend + TextMeshPro) into Chinese, entirely via offline file patching:
> (1) in-place rebuilding of the .NET `#US` string heap with `ldstr` token remapping,
> and (2) baking CJK glyphs as **static** SDF entries into TextMeshPro font atlases
> (a second atlas + glyph-ID-space separation + object content swap), because TMP's
> runtime "Dynamic" glyph population silently failed / crashed in the target game.
> Hardened by a full verification harness. Written so that a future AI agent (or human)
> can repeat the process on a similar title. See [docs/](docs/).

**语言**：中文为主（关键术语附英文）。

---

## 这份仓库解决什么问题

对一款 **Mono 后端 + TextMeshPro (3.0.6) + Unity 2022.3** 的日文 Unity 游戏做简体汉化，
在不重打包、不修改游戏代码逻辑的前提下：

1. **文本**：台词/界面文本是编译进 `Assembly-CSharp.dll` 的字符串常量，
   不是运行时读外部文件 → 需要**原地重建 .NET 元数据 `#US` 字符串堆**并重映射全部
   `ldstr` 指令操作数。
2. **字形**：TMP 字体图集是静态烘焙的日文字符集（占用 97%），简体字大量缺失 →
   需要**把中文字形离线烘焙为静态 SDF 字形**写进图集与字形表。
3. **验证**：所有改动必须有程序化回读校验 + 抽样目视验证，避免"能启动但显示错乱"。

最终效果：游戏可正常启动，全部文本显示为中文，原日文字形不受影响。

---

## 目录结构

```
README.md                        本文
docs/mono-string-patching.md     技巧 1：程序集字符串堆原地重写（含校验方法）
docs/tmp-font-atlas-baking.md    技巧 2：TMP 字形静态烘焙（双图集 + 内容交换）
docs/failure-cases.md            失败案例复盘：症状 → 根因 → 修复 → 教训
reference/snippets.md            关键算法参考实现片段（通用化伪代码/Python）
```

依赖（Python 3）：`UnityPy`、`fontTools`、`freetype-py`、`Pillow`。

---

## 总体流水线（最终验证过的顺序）

```
0. 备份所有将被修改的文件（*.bak）
1. 提取文本：反汇编 IL 收集 ldstr → 导出「原文→译文」Markdown 表（人工/AI 翻译）
2. 写回文本：重建 #US 堆 + 重映射 ldstr → 回读校验（详见 docs/mono-string-patching.md）
3. 字形覆盖分析：收集"程序集全部字符串的用字"减去"字体图集已有字符" = 待烘焙集合
4. 静态烘焙：FreeType 渲染 → SDF → 装箱 → 写图集像素 → 追加字形/字符表记录
   （详见 docs/tmp-font-atlas-baking.md）
5. 引用重定向：不改场景文件，用「对象内容交换」让旧引用获得新字形覆盖
6. 总校验：资源对象完整性 / IL 全量解码 / 字形表互引 / 覆盖率 100% / 抽样渲染目视
7. 进游戏逐界面验收
```

---

## 核心经验速查（血泪教训，细节见 docs/）

| # | 教训 | 一句话方案 |
|---|---|---|
| 1 | 台词在程序集字符串堆里，别找外部文件 | 解析 CLI 元数据，原地重建 `#US` 堆 |
| 2 | TMP 的 Dynamic 运行时补字不可依赖 | 补字需要可写图集纹理；本例中设置 Read/Write 后**启动即崩**，不可写则**静默失败** → 走静态烘焙 |
| 3 | 字形 ID 空间必须分段错开 | 静态字形 ID +100000、烘焙字形 ID +200000；TMP 按 glyphIndex 查重取靠前记录，重叠区间会导致中文显示成随机日文字形 |
| 4 | 序列化的 free rect list 不可信 | 它互相重叠、还与 used 区重叠（原版打包工具的过时记账）；按它装箱产出**叠字** → 用占用网格自管 |
| 5 | 原图集可能物理装不下新字形 | 实测静态占用 97%；解决：把一张**已无人引用的旧图集纹理**改造成第二图集（`m_AtlasTextures` 追加 + `glyph.atlasIndex=1`） |
| 6 | 图集像素**数据行自下而上**（row0=图像底部） | 写像素前先抠一个已知字渲染成 PNG 校准方向 |
| 7 | SDF 编码需实测校准 | 边缘=128、内部=255、外部=0、渐变半宽≈图集 padding；从既有字形扫描线拟合 |
| 8 | 想改 ~1000 处场景引用？别改场景文件 | 把"目标对象的数据"复制进"被引用的旧对象槽位"（pathID 不变），零外部引用改动 |
| 9 | 小整数 pathID 的盲目字节替换会毁掉网格数据 | pathID 2/3 这类小整数与二进制数据大量撞车；替换只对大而罕见的 pathID 安全 |
| 10 | 一切改动必须幂等 + 全量回读校验 | 重复运行结果一致；回读时把产物当新文件完整重新解析比对 |

---

## 验证清单（发布前全绿）

- [ ] 程序集：文件大小不变；除字符串堆与 ldstr 操作数外**零字节差异**
- [ ] 程序集：全部方法体 IL 可精确解码（0 异常）
- [ ] 程序集：回读后逐条比对全部 `ldstr` 指向的字符串 = 期望值
- [ ] 资源：对象表数量/pathID 集合一致；流式数据（.resS）偏移全部在界内
- [ ] 字体：字形表↔字符表互引 0 缺失；glyphIndex 无重复；与静态 ID 空间 0 冲突
- [ ] 字体：`m_AtlasPopulationMode` 为 Static；atlasIndex 合法；双图集纹理齐备
- [ ] 覆盖率：目标语言全部字符 ∈ (静态字符集 ∪ 烘焙集)，缺口 = 0
- [ ] 目视：从图集抠出抽样字形渲染成 PNG 人工确认（方向/形状/渐变）
- [ ] 译文表：占位符 `{0}` 保留、换行风格一致、逻辑标记未被翻译

---

## 授权与合规注意

- 字体：使用 OFL 授权字体（如思源宋体/Noto Serif SC）并按 OFL 要求附带许可证与声明；
  子集化（subsetting）后在衍生字体说明中注明。
- 不要在公开仓库中包含：游戏原始文本内容、译文内容、游戏截图、游戏内专有资产、
  以及任何可用于定位具体游戏的信息。仓库只保留**方法与通用代码**。
- 本文/仓库仅描述对**本地合法拥有的游戏副本**做个人汉化修改的技术过程。

---

## License

文档与参考代码：MIT（或按需）。字体遵循其自身授权（OFL 等）。
