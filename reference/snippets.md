# 参考实现片段（通用化，需按目标游戏适配）

> 以下片段来自一次完整实战的可用代码，已做通用化处理（去除具体游戏信息）。
> 均为"骨架级"实现：可读、可移植，但不是开箱即用的工具。
> 依赖：`UnityPy`、`fontTools`、`freetype-py`、`Pillow`。

---

## 1. Mono 程序集：ldstr 扫描与 #US 堆重建

```python
import struct

class Assembly:
    """最小化 Mono 程序集解析：定位 #US 堆、扫描全部 ldstr。"""

    def __init__(self, data: bytes):
        self.data = data
        self._parse_pe_and_metadata()      # 定位 #US 偏移/大小、#~ 表
        self._collect_ldstr()              # 遍历方法体收集 ldstr

    # ---- PE / CLI / 元数据（结构细节见 ECMA-335） ----
    def _parse_pe_and_metadata(self):
        # e_lfanew → COFF → OptionalHeader → CLI header(RVA) → 元数据根 "BSJB"
        # → 流头: "#~"(表流, 提供 tables_stream 偏移与各表行数) / "#US"(堆)
        # → 用行大小规则跳过 0x00~0x06 等表，得到 MethodDef RVA 所在行
        ...  # 略：与通用 PE 解析一致；关键是得到 self.us_off, self.us_size,
             #          self.method_rvas（每个 MethodDef 的 RVA）

    # ---- 方法体线性扫描（tiny/fat 头） ----
    def _collect_ldstr(self):
        self.ldstr = []          # [(il内token的文件偏移, #US偏移)]
        for rva in self.method_rvas:
            off = self.rva2off(rva)
            b = self.data[off]
            if b & 3 == 2:                       # tiny
                code_size = b >> 2
                code = off + 1
            else:                                # fat
                hdr = struct.unpack_from('<HHII', self.data, off)
                code_size = hdr[3] >> 16
                code = off + (hdr[0] >> 12) * 4
            end = code + code_size
            p = code
            while p < end:
                op = self.data[p]
                if op == 0x72:                   # ldstr
                    self.ldstr.append((p + 1,
                                       struct.unpack_from('<I', self.data, p + 1)[0] & 0xFFFFFF))
                    p += 5
                    continue
                # 按 opcode 表推进：单字节/0xFE 前缀双字节/内联操作数长度
                # （switch 有 count*4 + 4 的内联数据，必须跳过）
                p += self.opcode_len(p)

    # ---- #US 字符串解码 ----
    def read_string(self, us_off: int) -> str:
        b = self.data
        p = self.us_off + us_off
        b0 = b[p]
        size = b0 & 0x7F if b0 & 0x80 == 0 else (
            struct.unpack_from('<H', b, p)[0] & 0x3FFF if b0 & 0xC0 == 0x80
            else struct.unpack_from('<I', b, p)[0] & 0x1FFFFFFF)
        p += 1 if b0 & 0x80 == 0 else (2 if b0 & 0xC0 == 0x80 else 4)
        return b[p:p + size * 2 - 2].decode('utf-16-le')   # 末尾 \x00 去除
```

重建与写回（保持文件大小不变）：

```python
def rebuild_heap(asm, mapping: dict[str, str]):
    """mapping: {旧字符串: 新字符串}；未命中的保持原样。返回 (新堆bytes, 新偏移表)。"""
    new_heap = bytearray()
    new_off = {}
    for s_old, off in asm.entries:                 # 按“旧偏移”稳定顺序
        s_new = mapping.get(s_old, s_old)
        new_off[s_old] = len(new_heap)
        u = s_new.encode('utf-16-le') + b'\x00'
        n = len(u) + 1
        if n < 0x80:
            new_heap += bytes([n])
        elif n < 0x4000:
            new_heap += struct.pack('<H', 0x8000 | n)
        else:
            new_heap += struct.pack('<I', 0xC0000000 | n)
        new_heap += u
    return bytes(new_heap), new_off

def apply(asm, mapping):
    heap, new_off = rebuild_heap(asm, mapping)
    out = bytearray(asm.data)
    out[asm.us_off: asm.us_off + len(heap)] = heap
    out[asm.us_off + len(heap): asm.us_off + asm.us_size] = b'\x00' * (asm.us_size - len(heap))
    for tok_off, us_off in asm.ldstr:
        old = asm.read_string(us_off)
        struct.pack_into('<I', out, tok_off, 0x70000000 | new_off[old])
    verify_and_write(asm.data, bytes(out), mapping, asm)
```

**校验骨架**（字节审计 + 全量重解析比对 + IL 全解码）：

```python
def verify_and_write(old: bytes, new: bytes, mapping, asm):
    # 1) 字节审计：差异只允许出现在 #US 堆区间与 ldstr 操作数
    # 2) 全量重解析：Assembly(new) → 逐条比对 ldstr 字符串 == 期望值
    # 3) 全部方法体 IL 重新线性解码 0 异常
    # 4) len(new) == len(old)
    ...
```

---

## 2. TMP_FontAsset 布局解析（无 typetree，启发式定位）

> 完整字段顺序见 `docs/tmp-font-atlas-baking.md` 第 1 节。
> 定位策略：**先用可靠锚点（字符串/PPtr 模式），再用"结构自洽"校验**。

```python
import re, struct

def parse_tmp(raw: bytes, pid: int) -> dict:
    # 字段锚点：m_Version 形如 "1.1.0"（长度前缀字符串）
    # 布局: [family][style][version][srcPPtr(12B)][mode i32]
    #       [glyphCount][glyphs*52][charCount][chars*16]
    #       [texCount][texPPtr*12][atlasIndex][multi u8][clear u8]
    #       [usedCount][usedRects*16][freeCount][freeRects*16]
    #       [fontInfo_Legacy ...][kerning][featureTable][fallback][creationSettings]
    #       [atlasWidth][atlasHeight][padding][renderMode]
    version_pos = raw.find(b'1.1.0')
    ...
    # atlasWidth 定位：在尾部搜索 (w == h) 且 w ∈ {512,1024,2048,4096,8192}
    #                 且其后是 0<pad<32 与已知 renderMode 的四元组
    ...
```

字形/字符记录读写（注意 atlasIndex 偏移 44）：

```python
GLYPH = '<I5f4ifii'      # index, 5 metrics, 4 rect, scale, atlasIndex, class = 52B
CHAR  = '<IIIf'          # const1, unicode, glyphIndex, scale = 16B

def shift_static_indexes(raw, info, shift=100000):
    g0 = info['off_glyph_count'] + 4
    c0 = info['off_char_count'] + 4
    for k in range(info['glyph_n']):
        p = g0 + k * 52
        struct.pack_into('<I', raw, p, struct.unpack_from('<I', raw, p)[0] + shift)
    for k in range(info['char_n']):
        p = c0 + k * 16 + 8                      # glyphIndex 字段
        struct.pack_into('<I', raw, p, struct.unpack_from('<I', raw, p)[0] + shift)
```

---

## 3. SDF 字形烘焙（FreeType + chamfer 距离 + 占用网格装箱）

```python
import io
import freetype
from fontTools.ttLib import TTFont

# 日文/中文路径在 Windows 上会让 freetype-py 打不开文件 → 从内存加载
face = freetype.Face(io.BytesIO(open(SUBSET_PATH, 'rb').read()))
face.set_pixel_sizes(0, POINT_SIZE)      # 1pt = 1px（72dpi）；用既有静态字形 advance 反推验证

def render(ch):
    face.load_char(ch, freetype.FT_LOAD_RENDER)
    g = face.glyph
    bm = g.bitmap                        # 8bit AA 墨迹位图
    ink = [[bm.buffer[j * bm.width + i] for i in range(bm.width)] for j in range(bm.rows)]
    return ink, g.bitmap_left, g.bitmap_top, g.advance.x / 64.0

def sdf_cell(ink, spread=6.6, margin=7):
    """单元 = 墨迹 + 2×margin；sd 内部为正；alpha = 128 + clamp(sd/spread)×127。

    spread/margin 必须从目标图集实测校准：抠一个已知静态字，
    看扫描线 0→255 渐变宽度（≈2×spread）与边缘值（128）。
    """
    ih, iw = len(ink), len(ink[0])
    cw, ch = iw + 2 * margin, ih + 2 * margin
    INF = 1e9
    d_in  = [[0.0]*cw for _ in range(ch)]
    d_out = [[0.0]*cw for _ in range(ch)]
    for j in range(ch):
        for i in range(cw):
            ii, jj = i - margin, j - margin
            inside = 0 <= ii < iw and 0 <= jj < ih and ink[jj][ii] >= 128
            (d_out if inside else d_in)[j][i] = INF
    D1, D2 = 1.0, 1.41421356
    def sweep(g):
        for j in range(ch):                          # 正向
            gr, up = g[j], (g[j-1] if j else None)
            for i in range(cw):
                d = gr[i]
                if d == 0: continue
                if i and gr[i-1] + D1 < d: d = gr[i-1] + D1
                if up:
                    if up[i] + D1 < d: d = up[i] + D1
                    if i and up[i-1] + D2 < d: d = up[i-1] + D2
                    if i+1 < cw and up[i+1] + D2 < d: d = up[i+1] + D2
                gr[i] = d
        for j in range(ch-1, -1, -1):                # 反向
            gr, dn = g[j], (g[j+1] if j+1 < ch else None)
            for i in range(cw-1, -1, -1):
                d = gr[i]
                if d == 0: continue
                if i+1 < cw and gr[i+1] + D1 < d: d = gr[i+1] + D1
                if dn:
                    if dn[i] + D1 < d: d = dn[i] + D1
                    if i+1 < cw and dn[i+1] + D2 < d: d = dn[i+1] + D2
                    if i and dn[i-1] + D2 < d: d = dn[i-1] + D2
                gr[i] = d
    sweep(d_in); sweep(d_out)
    rows = []
    for j in range(ch):
        row = bytearray(cw)
        for i in range(cw):
            s = d_out[j][i] - d_in[j][i]             # 内部为正
            row[i] = max(0, min(255, 128 + int(max(-1.0, min(1.0, s / spread)) * 127)))
        rows.append(bytes(row))
    return rows
```

**占用网格装箱**（序列化 free list 不可信——可能互相重叠、与 used 区重叠、虚报）：

```python
W = ATLAS_SIZE
grid = bytearray(W * W)
for (ux, uy, uw, uh) in static_used_rects:           # 静态字形区域标 1
    for j in range(uh):
        grid[(uy + j) * W + ux: (uy + j) * W + ux + uw] = b'\x01' * uw

scan = [0, 0]
def pack(w, h):
    """全局单调扫描，找第一块全空闲 w×h；返回 (x, y) 并标记。"""
    x, y = scan
    while y + h <= W:
        row = y * W
        if x + w > W:
            y, x = y + 1, 0
            continue
        if grid[row + x: row + x + w].count(0) != w:   # 首行快速过滤
            x += 16
            continue
        if all(grid[(y + j) * W + x: (y + j) * W + x + w].count(0) == w
               for j in range(1, h)):
            for j in range(h):
                grid[(y + j) * W + x: (y + j) * W + x + w] = b'\x01' * w
            scan[0], scan[1] = x + w, y
            return x, y
        x += 16
    return None
```

**字形/字符记录**（写入第二图集，atlasIndex=1）：

```python
def make_records(ch, ink, bl, bt, adv, rect, glyph_id, bake_offset=200000):
    cw, chh = len(ink[0]) + 2 * MARGIN, len(ink) + 2 * MARGIN
    glyph = struct.pack('<I5f4ifii', glyph_id + bake_offset,
                        float(cw), float(chh),
                        float(bl - MARGIN), float(bt + MARGIN), float(adv),
                        rect[0], rect[1], rect[2], rect[3],
                        1.0, 1, 0)                       # scale, atlasIndex=1, class=0
    char = struct.pack('<IIIf', 1, ord(ch), glyph_id + bake_offset, 1.0)
    return glyph, char
```

**像素写回（.resS，数据行自下而上）**：

```python
def write_cell(ress_file, stream_offset, atlas_w, rect, cell_rows_topdown):
    x, y, w, h = rect
    for k in range(h):
        ress_file.seek(stream_offset + (y + h - 1 - k) * atlas_w + x)
        ress_file.write(cell_rows_topdown[k])
```

**双图集**：字体资产里 `m_AtlasTextures` 数组 1→2（追加一张已无人引用的同规格
Texture2D 的 PPtr），新字形 `atlasIndex=1`，`m_AtlasPopulationMode=0`（Static）。
TMP 运行时对非主图集字形自动生成子网格材质。

---

## 4. 对象内容交换（引用重定向的零风险替代）

```python
def content_swap(env, src_pid, dst_pid):
    """把 dst 对象的数据复制进 src 对象槽位：外部引用（按 pathID 指向 src 的）
    自动获得 dst 的内容，场景/预制体文件零改动。"""
    objs = {o.path_id: o for o in env.objects}
    objs[src_pid].set_raw_data(objs[dst_pid].get_raw_data())

# 用法：字体资产 A 被 ~1000 个 UI 组件引用、其字形装不下中文；
#       字体资产 B 已具备全量字形 → content_swap(env, A, B)
#       同理交换两者的材质对象（材质里的 _MainTex 指向各自的图集纹理）。
```

> 为什么不做场景文件里的 PPtr 字节替换：小整数 pathID（2、3、162…）的 12 字节
> 模式与网格/顶点/颜色等二进制数据大量撞车，盲目替换会毁掉模型数据。
> 大而罕见的 pathID 风险较低，但仍不如内容交换干净。

---

## 5. 验证辅助

```python
def byte_diff_regions(old: bytes, new: bytes):
    """输出差异区间列表——用于审计'改动是否只发生在预期区域'。"""
    n = min(len(old), len(new))
    diffs = [i for i in range(n) if old[i] != new[i]]
    runs = []
    for i in diffs:
        if runs and i == runs[-1][1]:
            runs[-1][1] = i + 1
        else:
            runs.append([i, i + 1])
    return runs, len(old), len(new)

def extract_glyph_png(atlas_resS_path, stream_off, atlas_w, rect, out_png):
    """从图集抠出字形渲染 PNG（数据行自下而上 → 翻转显示）。人工目视校准/验收用。"""
    from PIL import Image
    x, y, w, h = rect
    f = open(atlas_resS_path, 'rb')
    rows = []
    for j in range(h):
        f.seek(stream_off + (y + j) * atlas_w + x)
        rows.append(bytes(f.read(w)))
    tile = Image.new('L', (w, h))
    tile.putdata([v for row in reversed(rows) for v in row])   # 数据行自下而上
    tile.resize((w * 4, h * 4), Image.NEAREST).save(out_png)

def assert_disjoint(static_ids, baked_ids):
    overlap = set(static_ids) & set(baked_ids)
    assert not overlap, f'字形索引冲突 {len(overlap)} 个'
    assert len(baked_ids) == len(set(baked_ids)), '烘焙字形索引重复'
```
