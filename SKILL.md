---
name: sf
description: 在网页中使用 SF Symbols 图标。当用户提到「SF」「SF Symbol」「SF图标」「系统图标」「苹果图标」，或需要在 HTML/网页中放图标时自动触发。调用本地 sfsym 工具生成内联 SVG，支持全部 6000+ SF Symbols 图标库。做网页时凡是需要图标都用这个。用法：说出图标名 + 字号即可。
---

# SF Symbols 网页图标工具

## 这个 skill 是什么

把 Apple SF Symbols 库(6000+ 系统图标)转成**即用的内联 SVG**,塞进网页里。产出的 SVG 用 `fill="currentColor"`,颜色完全由外层 CSS `color` 属性控制,天然适配 light/dark 模式。

**做网页时,所有图标都走这个 skill**——不用 emoji(跨平台渲染不一致)、不用自画 SVG(成本高)、不用图标字体库(还要额外加载)。

## 什么时候自动触发

用户对话里出现以下信号时,Claude 自动调用本 skill:

| 信号 | 例子 |
|------|------|
| 点名 SF / SF Symbol | "用 SF 的 chevron.left" |
| 说"苹果图标 / 系统图标" | "加个苹果风格的搜索图标" |
| 给具体 symbol 名 | "magnifyingglass 13pt" / "加个 xmark" |
| 网页里要图标但没指定库 | "顶部加个返回按钮" / "加个关闭图标" |

## 怎么用(对 Claude 说)

**最简单格式:** `<图标名> <字号>`

| 输入 | 产出 |
|------|------|
| "用 magnifyingglass 13pt" | 13pt 搜索图标 SVG,28×28 BOX |
| "加 chevron.left 10pt" | 10pt 返回箭头,20×20 BOX |
| "关闭按钮,用 SF" | xmark 图标,默认 13pt |

**不确定图标名?** 说场景,Claude 查表给建议。常用名见本文末尾「常用图标速查」。

**默认字号:** 13pt(Figma 标准 Button 1)。需要小号写 "10pt"。其他尺寸能用但不是预设,非必要别改。

**颜色?** 不用管。SVG 里是 `currentColor`,你外层 CSS 写 `color: var(--xxx)` 就行。

## Step 0:首次使用自检(每次触发 skill 前必跑)

本 skill 有三个系统依赖,缺任一都跑不了。**首次触发时跑一次自检,缺啥报缺啥**,不要直接跑 Python 然后抛一堆堆栈:

```bash
# 三行检查,任一失败就停住,输出安装命令给用户
test -f "/Applications/SF Symbols.app/Contents/Resources/Fonts/SFSymbolsFallback.otf" \
  || { echo "✗ 缺 SF Symbols.app → 下载 https://developer.apple.com/sf-symbols/"; exit 1; }
command -v sfsym >/dev/null \
  || { echo "✗ 缺 sfsym CLI → brew install yapstudios/tap/sfsym"; exit 1; }
python3 -c "import fontTools, AppKit" 2>/dev/null \
  || { echo "✗ 缺 Python 依赖 → pip3 install fonttools pyobjc-framework-Cocoa"; exit 1; }
echo "✓ 依赖齐全"
```

失败时:**只输出缺失项和安装命令,不继续跑生成逻辑**。让用户装完再喊。

全通过 = 进入下面的「标准工作流」。同一 session 第二次触发可以跳过自检(已知环境就绪)。

---

## 标准工作流：字体 → 转曲 → BOX 居中

这是经过验证的正确方式（对标 Figma 规格）：

1. **字号 13pt，line-height 22px，font-weight 510**（Figma 标准参数）
2. **BOX = 28×28px**（Figma Button 1 容器）
3. 从 `SFSymbolsFallback.otf` 提取路径，**在 wght=510 处实例化**变量字体
4. 按正确排版指标居中放入 BOX
5. 输出即用 SVG，无字体依赖，`currentColor` 颜色可控

```python
import subprocess, re
from fontTools.ttLib import TTFont
from fontTools.varLib.instancer import instantiateVariableFont
from fontTools.pens.svgPathPen import SVGPathPen

FONT = "/Applications/SF Symbols.app/Contents/Resources/Fonts/SFSymbolsFallback.otf"

def symbol_name_to_codepoint(name):
    """symbol 名 → codepoint，用 sfsym + canvas 像素比对查找"""
    import AppKit, re as _re
    from fontTools.ttLib import TTFont as _TTFont
    from fontTools.pens.svgPathPen import SVGPathPen as _Pen
    font = _TTFont(FONT)
    cmap = font.getBestCmap()
    gs = font.getGlyphSet()
    scale_ref = (60-8)/1841.0

    img = AppKit.NSImage.imageWithSystemSymbolName_accessibilityDescription_(name, None)
    if not img:
        raise ValueError(f"Symbol not found: {name}")
    rep = AppKit.NSBitmapImageRep.alloc().initWithBitmapDataPlanes_pixelsWide_pixelsHigh_bitsPerSample_samplesPerPixel_hasAlpha_isPlanar_colorSpaceName_bytesPerRow_bitsPerPixel_(
        None, 60, 60, 8, 4, True, False, AppKit.NSDeviceRGBColorSpace, 0, 0)
    ctx = AppKit.NSGraphicsContext.graphicsContextWithBitmapImageRep_(rep)
    AppKit.NSGraphicsContext.setCurrentContext_(ctx)
    AppKit.NSColor.whiteColor().set(); AppKit.NSRectFill(AppKit.NSMakeRect(0,0,60,60))
    img.drawInRect_(AppKit.NSMakeRect(6,6,48,48))
    raw = bytes(rep.bitmapData()[:60*60*4])
    ref = bytes(raw[i] for i in range(0, len(raw), 4))
    ref_dark = sum(1 for v in ref if v < 200)

    def render_g(gname):
        pen = _Pen(gs); gs[gname].draw(pen); d = pen.getCommands()
        if not d: return None
        tokens = _re.findall(r'[MCLZcqz]|[-\d.]+(?:e[-+]?\d+)?', d)
        path = AppKit.NSBezierPath.bezierPath()
        i = 0
        while i < len(tokens):
            cmd = tokens[i]; i += 1
            try:
                if cmd=='M': x,y=float(tokens[i]),float(tokens[i+1]);i+=2;path.moveToPoint_(AppKit.NSMakePoint(x,y))
                elif cmd=='L': x,y=float(tokens[i]),float(tokens[i+1]);i+=2;path.lineToPoint_(AppKit.NSMakePoint(x,y))
                elif cmd=='C':
                    x1,y1=float(tokens[i]),float(tokens[i+1]);i+=2
                    x2,y2=float(tokens[i]),float(tokens[i+1]);i+=2
                    x3,y3=float(tokens[i]),float(tokens[i+1]);i+=2
                    path.curveToPoint_controlPoint1_controlPoint2_(AppKit.NSMakePoint(x3,y3),AppKit.NSMakePoint(x1,y1),AppKit.NSMakePoint(x2,y2))
                elif cmd in ('Z','z'): path.closePath()
            except: break
        tx = AppKit.NSAffineTransform.transform()
        tx.translateXBy_yBy_(4.0, 4.0+199*scale_ref); tx.scaleBy_(scale_ref)
        path.transformUsingAffineTransform_(tx)
        r2 = AppKit.NSBitmapImageRep.alloc().initWithBitmapDataPlanes_pixelsWide_pixelsHigh_bitsPerSample_samplesPerPixel_hasAlpha_isPlanar_colorSpaceName_bytesPerRow_bitsPerPixel_(
            None,60,60,8,4,True,False,AppKit.NSDeviceRGBColorSpace,0,0)
        c2 = AppKit.NSGraphicsContext.graphicsContextWithBitmapImageRep_(r2)
        AppKit.NSGraphicsContext.setCurrentContext_(c2)
        AppKit.NSColor.whiteColor().set();AppKit.NSRectFill(AppKit.NSMakeRect(0,0,60,60))
        AppKit.NSColor.blackColor().set();path.fill()
        raw2 = bytes(r2.bitmapData()[:60*60*4])
        return bytes(raw2[i] for i in range(0, len(raw2), 4))

    candidates = [(cp, n) for cp, n in cmap.items() if 0x100000 <= cp <= 0x10FFFF and '.medium' in str(n)]
    best_score, best_cp = float('inf'), None
    for cp, gname in candidates:
        try:
            ch = render_g(gname)
            if ch is None: continue
            dark = sum(1 for v in ch if v < 200)
            if dark < ref_dark*0.4 or dark > ref_dark*2.5: continue
            score = sum(abs(int(a)-int(b)) for a,b in zip(ref, ch))
            if score < best_score: best_score = score; best_cp = cp
        except: continue
    if best_cp is None:
        raise ValueError(f"Codepoint not found for: {name}")
    return best_cp


def sf_to_svg(codepoint_or_name, pt=13, line_height=22, box=28, weight=510):
    """
    默认值即 Figma 标准：13pt / weight 510 / line-height 22 / BOX 28
    只有用户明确要求时才修改对应参数。
    codepoint_or_name: int codepoint 或 str symbol 名（如 'xmark'）
    """
    # 支持直接传 symbol 名
    if isinstance(codepoint_or_name, str):
        codepoint = symbol_name_to_codepoint(codepoint_or_name)
    else:
        codepoint = codepoint_or_name

    if not FONT or not __import__('os').path.exists(FONT):
        raise FileNotFoundError(f"SF Symbols app not found at: {FONT}")

    font = TTFont(FONT)
    inst = instantiateVariableFont(font, {"wght": weight})
    cmap = inst.getBestCmap()
    gs = inst.getGlyphSet()
    upm = inst['head'].unitsPerEm
    asc = inst['hhea'].ascent
    desc = inst['hhea'].descent
    # scale 来自 Figma 转曲实测（circle.lefthalf.filled 圆形直径 / 2066 UPM）
    # 13pt/wght=510 → circle=16.75px → scale=16.75/2066=0.008107
    # 10pt/wght=590 → circle=13px    → scale=13/2066=0.006291
    SCALE_MAP = {(13,510): 16.75/2066, (10,590): 13/2066}
    scale = SCALE_MAP.get((pt, weight), pt / upm)

    gname = cmap.get(codepoint)
    if not gname:
        raise ValueError(f"Codepoint U+{codepoint:X} not found in font")

    adv = inst['hmtx'].metrics[gname][0]
    pen = SVGPathPen(gs)
    gs[gname].draw(pen)
    d = pen.getCommands()
    if not d:
        raise ValueError(f"No path data for U+{codepoint:X}")

    tx = (box - adv * scale) / 2
    natural_lh = (asc - desc) * scale
    half_leading = (line_height - natural_lh) / 2
    line_box_top = (box - line_height) / 2
    ty = line_box_top + half_leading + asc * scale

    return (
        f'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {box} {box}" width="{box}" height="{box}">'
        f'<path transform="translate({tx:.4f},{ty:.4f}) scale({scale:.6f},-{scale:.6f})" '
        f'fill="currentColor" d="{d}"/>'
        f'</svg>'
    )


# 批量版本：多图标一次性生成（10× 性能提升）
# 每次 instantiateVariableFont 要 3-5 秒，批量时只调用一次
_INSTANCE_CACHE = {}

def sf_batch(codepoints_or_names, pt=13, line_height=22, box=28, weight=510):
    """
    一次实例化字体，批量生成多个图标 SVG。
    处理 10+ 个图标时必用这个，不要循环调用 sf_to_svg。
    返回 {name_or_cp: svg_string} 字典。
    """
    import os
    if not os.path.exists(FONT):
        raise FileNotFoundError(f"SF Symbols app not found at: {FONT}")

    # 缓存实例化结果（weight 相同就复用）
    if weight not in _INSTANCE_CACHE:
        _INSTANCE_CACHE[weight] = instantiateVariableFont(TTFont(FONT), {"wght": weight})
    inst = _INSTANCE_CACHE[weight]

    cmap = inst.getBestCmap()
    gs = inst.getGlyphSet()
    upm = inst['head'].unitsPerEm
    asc = inst['hhea'].ascent
    desc = inst['hhea'].descent
    SCALE_MAP = {(13,510): 16.75/2066, (10,590): 13/2066}
    scale = SCALE_MAP.get((pt, weight), pt / upm)
    natural_lh = (asc - desc) * scale
    half_leading = (line_height - natural_lh) / 2
    ty = (box - line_height)/2 + half_leading + asc * scale

    results = {}
    for item in codepoints_or_names:
        cp = symbol_name_to_codepoint(item) if isinstance(item, str) else item
        gname = cmap.get(cp)
        if not gname: continue
        adv = inst['hmtx'].metrics[gname][0]
        pen = SVGPathPen(gs)
        gs[gname].draw(pen)
        d = pen.getCommands()
        if not d: continue
        tx = (box - adv * scale) / 2
        results[item] = (
            f'<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {box} {box}" width="{box}" height="{box}">'
            f'<path transform="translate({tx:.4f},{ty:.4f}) scale({scale:.6f},-{scale:.6f})" '
            f'fill="currentColor" d="{d}"/></svg>'
        )
    return results
```

**使用建议：**
- 1-2 个图标 → `sf_to_svg()`
- 3 个以上 → `sf_batch()`（性能差 5-10 倍）

---

## Token 节省原则（重要,违反就浪费）

**每个 SVG path d 字符串 2000+ 字符。贴到对话里一次 = 一次浪费。**

### ✅ 正确做法

1. **脚本直接写文件,不回显到对话**
   ```python
   # 生成 SVG 后,用 pathlib 直接替换源文件,不 print、不贴给用户看
   import pathlib, re
   p = pathlib.Path('src/components/IconPanel.jsx')
   s = p.read_text()
   s = s.replace(OLD_SVG, new_svg, 1)
   p.write_text(s)
   print(f'replaced')   # 只报动作,不报内容
   ```

2. **改尺寸不重跑脚本,只改 SVG 属性**
   SF Symbols SVG 是矢量,整体缩放只需改 `width` / `height`,path d 一字不改。
   ```
   10pt → 8pt:width="20" → width="16"(等比缩)
   ```
   **不要**重跑 Python 脚本生成新 SVG —— 结果一样,白花 token。

3. **多图标用 `sf_batch()`**(已有,再次强调):一次实例化字体,批量输出。不要循环调 `sf_to_svg`(每次 3-5 秒 + 重复 font 加载)。

### ❌ 反模式(真实踩坑)

- 在对话里贴完整 SVG 三次(10pt 版 → bbox 归一化 14px 版 → 回到 skill 标准 20px 版),每次 2000+ 字符。应该:直接 Edit 文件,只汇报"改好了"。
- 用户只要尺寸变化(20→16),脚本从头生成:纯浪费。应该:单个 `s.replace('width="20"', 'width="16"')`。
- 同一 skill 跑 5 次只为看 SVG 结果:应该一次写入文件,看浏览器效果。

### 视觉大小不一致是 SF Symbols 的**设计**,不是 bug

不同 symbol 的 glyph advance 不同(`photo.badge.exclamationmark` 比 `figure.stand` 宽)。按 SKILL 标准 BOX 居中后,**视觉宽度会有差异**,这是 Apple 设计的 optical balance。

**不要用 bbox 归一化"修正"** — 那会偏离 SF Symbols 标准渲染。要视觉齐,换视觉重量接近的图标(两个都单 glyph,不混 badge)。

---

## 默认参数（除非用户明确要求，否则不改）

两套预设，按场景选择：

### 13pt 标准（默认）
| 参数 | 值 |
|------|----|
| `font-size` | 13px |
| `font-weight` | 510 |
| `line-height` | 22px |
| BOX | 28×28px |

### 10pt 小号
| 参数 | 值 |
|------|----|
| `font-size` | 10px |
| `font-weight` | 590 |
| `line-height` | 22px |
| BOX | 按需（Symbol 元素 16×22px） |

调用时传对应预设：
```python
# 13pt 标准（Figma 转曲验证：circle = 16.75px，scale = 16.75/2066 = 0.008107）
sf_to_svg(cp, pt=13, line_height=22, box=28, weight=510)

# 10pt 小号（Figma 转曲验证：circle = 13px，scale = 13/2066 = 0.006291）
sf_to_svg(cp, pt=10, line_height=22, box=20, weight=590)
```

> **Scale 计算方式**：`scale = Figma转曲后圆形尺寸 / circle_glyph_UPM(≈2066)`
> 不用 `pt/upm`，用 Figma 实测值作为地面真值。

---

## 为什么是 SFSymbolsFallback.otf，不是 SF Pro

- SF Pro（SFNS.ttf）只有 ss01-ss09，**没有 ss16**，不含 SF Symbol 字形
- 这些 PUA 字符（U+100000+）只存在于 `SFSymbolsFallback.otf`
- 浏览器渲染时也是通过系统字体级联降级到这个字体

---

## 为什么要实例化变量字体

`SFSymbolsFallback.otf` 是 CFF2 变量字体，weight 轴范围 1-1000，默认 400。
直接提取路径是 wght=400（细），必须先在 **wght=510** 处实例化才能拿到正确粗细。

```python
from fontTools.varLib.instancer import instantiateVariableFont
inst = instantiateVariableFont(font, {"wght": 510})
```

---

## sfsym 命令行（快速预览用，不用于生产）

```bash
/opt/homebrew/bin/sfsym export <symbol-name> --size <pt> -f svg -o - \
  | sed 's/fill="#000000"/fill="currentColor"/g'
```

---

## 颜色 / Light·Dark 适配

SVG 路径统一用 `fill="currentColor"`，颜色由外层 CSS `color` 属性控制。

**写入前必须检查：**
1. 当前场景是否需要 Light/Dark 切换？
2. 图标落地的背景色是什么？（深色背景 → 用亮色 token，浅色背景 → 用深色 token）

**常用 token 对照（HyperOS 设计系统）：**

| 场景 | CSS 变量 | Dark 值 | Light 值 |
|------|---------|---------|---------|
| 主要图标 | `var(--hyperos_color_on_surface)` | `rgba(255,255,255,0.88)` | `rgba(0,0,0,0.85)` |
| 次要图标 | `var(--hyperos_color_on_surface_secondary)` | `rgba(255,255,255,0.60)` | `rgba(0,0,0,0.60)` |
| 三级图标 | `var(--hyperos_color_on_surface_tertiary)` | `rgba(255,255,255,0.40)` | `rgba(0,0,0,0.40)` |
| 强调色图标 | `var(--hyperos_color_primary)` | `#3482ff` | `#007aff` |

**Shell 工具（sp 变量体系）：**

| 场景 | CSS 变量 |
|------|---------|
| 默认按钮图标 | `var(--sp-fg-2)` |
| Active 状态 | `var(--sp-accent)` |

Figma 规格里 `color: #1A1A1A` 是 Light 模式值，实际写代码要替换为对应 CSS 变量，不要硬编码颜色。

---

## 搜索图标名

```bash
/opt/homebrew/bin/sfsym list --search <关键词>
```

---

## 常用图标速查

| 场景 | Symbol 名 |
|------|-----------|
| 搜索 | `magnifyingglass` |
| 主页 | `house` / `house.fill` |
| 设置 | `gearshape` / `gearshape.fill` |
| 用户 | `person` / `person.fill` |
| 添加 | `plus` / `plus.circle` |
| 删除 | `trash` / `xmark` |
| 返回 | `chevron.left` |
| 前进 | `chevron.right` |
| 分享/导出 | `square.and.arrow.up` |
| 收藏 | `star` / `heart` |
| 通知 | `bell` / `bell.fill` |
| 相机 | `camera` |
| 菜单 | `line.3.horizontal` |
| 关闭 | `xmark` / `xmark.circle` |
| 确认 | `checkmark` / `checkmark.circle` |
| 警告 | `exclamationmark.triangle` |
| 编辑 | `pencil` / `square.and.pencil` |
| 锁 | `lock` / `lock.fill` |
| 下载 | `arrow.down.to.line` |
| 上传 | `arrow.up.from.line` |
