# 单文件 HTML 规格文档 · 生成提示词

用来生成「双击就能看、带图、不需要任何网络」的技术设计文档。
参考成品：同目录 `DESIGN.html`。

---

## 一、提示词正文（直接复制）

```text
写一份单文件 HTML 技术规格文档，可直接双击打开，不依赖任何网络资源。

【形式】
- 单个 .html，内联 <style>；不用 CDN、外部字体、JS 框架、构建步骤
- 图用内联 SVG 手写（viewBox 统一 0 0 680 *），不要 mermaid、不要图片、不要 base64
- 一套暖中性色 + 一个主色：
  底 #FAFAF8 / 面板 #FFFFFF / 线 #E4E2DC / 正文 #1F1F1D / 次要 #5F5E5A / 主色 #185FA5
  辅助色：青 #0F6E56、浅蓝底 #E6F1FB、代码底 #F4F3EF
- 字体 "PingFang SC" 系统栈；正文 15px / 行高 1.75
- 标题 h1 32px、h2 20px、h3 16px，一律 font-weight 500
  （不要 600/700，中文粗标题会显脏）
- 容器 max-width 900px 居中；h2 上方留 56px 空白 + 一条 1px 下边框分节
- 只用五种结构化元素：
  表格（13.5px、表头 #F7F6F2、只有横线没有竖线）
  代码块（12.5px 等宽、圆角 10px、浅底 + 1px 边框）
  卡片 .card（白底、1px 边框、圆角 10px）
  提示块 .lead（左 3px 主色条 + 浅蓝底、右侧圆角）
  图容器 .fig（白底、圆角 14px、图注 12px 虚线分隔）
- 禁止：渐变、阴影、图标字体、大面积彩色铺底、超大圆角

【内容组织】
- 页头四层：大写小字 kicker → 一句 h1 → 两行定位说明 → meta 行（日期 / 规模数字 / 技术栈）
- 章节用编号标题，从「0 · 一句话定位」开始，编号写进 h2
- 每节先给结论，再给理由；理由用表格或卡片承载，不写成段落
- 能列成表的一律列成表；一段超过四行就改成列表或表

【口径】
- 只写最终结论。不写「上一版有 N 处问题」「本版修正了…」「附录 · 修订记录」
- 每处只给一个明确答案，不给「可以用 A 也可以用 B」
- 数字必须具体：几个容器、几行、几毫秒，不写「性能更好」
```

---

## 二、决定性三条

| 约束 | 效果 |
|---|---|
| 标题一律 `font-weight: 500` | 中文用 600/700 会显脏；500 在 PingFang 上有印刷品的克制感。**这一条对观感影响最大** |
| 只有横线，没有竖线，不用色块分区 | 表格去竖线、章节用下边框，页面自然干净；换成卡片网格立刻像 PPT |
| 不允许渐变、阴影、图标 | 出现任意一样，就从「规格文档」滑向「营销页」 |

---

## 三、尺寸速查

| 项 | 值 |
|---|---|
| 页面留白 | `padding: 56px 24px 96px` |
| 内容宽 | `max-width: 900px` |
| 正文 | 15px / 1.75 |
| 表格 | 13.5px，单元格 `padding: 9px 11px` |
| 代码块 | 12.5px / 1.7，`padding: 18px 20px` |
| 图注 | 12px，`#888780`，虚线顶边 |
| 章节间隔 | h2 `margin-top: 56px` |
| 窄屏 | 700px 以下收窄留白、h1 降到 26px |

---

## 四、SVG 手绘惯例

- 每个图一个 `<svg viewBox="0 0 680 H" role="img" style="font-family:inherit">`，H 按内容定，不自适应缩放
- 宽度统一 680，是为了和 900px 正文的视觉密度匹配；改宽要同时改字号
- 文字垂直居中用 `dominant-baseline="central"`，不要靠估算 y 坐标
- 节点是 `<rect rx="8">`，连线是 `<path>`，箭头用 `<defs><marker>` 定义一次复用
- **每个形状必须显式写 `fill`**，不要依赖 class 或继承，否则深色环境下可能变黑
- 图里字号只用 11 / 12 / 12.5 / 13 四档，不要出现 10 以下
- 颜色沿用正文那套变量值，主色强调关键节点，次要说明用 `#5F5E5A`

---

## 五、常见走偏

| 症候 | 原因 |
|---|---|
| 变成 PPT 观感 | 上了卡片网格、彩色标题条、投影 |
| 变成 README | 全是大段散文和嵌套列表，没有表和卡片 |
| 图很丑 | 用了 mermaid 自动布局，节点宽窄不一、连线穿越文字 |
| 手机上散架 | 固定像素宽度容器，或 SVG 没写 `width:100%; height:auto` |
| 需要联网才完整 | 引了 Google Fonts、CDN 上的 CSS/JS |

---

## 六、最小示例

### 6.1 整页骨架（可跑）

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>示例文档</title>
<style>
  :root {
    --bg:#FAFAF8; --panel:#FFF; --line:#E4E2DC; --ink:#1F1F1D; --ink2:#5F5E5A;
    --ink3:#888780; --blue:#185FA5; --teal:#0F6E56; --code-bg:#F4F3EF;
    --mono: ui-monospace, SFMono-Regular, Menlo, monospace;
  }
  * { box-sizing: border-box; }
  body { margin:0; padding:56px 24px 96px; background:var(--bg); color:var(--ink);
         font-family:"PingFang SC","Hiragino Sans GB",-apple-system,sans-serif;
         font-size:15px; line-height:1.75; -webkit-font-smoothing:antialiased; }
  .wrap { max-width:900px; margin:0 auto; }
  h1 { font-size:32px; font-weight:500; margin:0 0 12px; }
  h2 { font-size:20px; font-weight:500; margin:56px 0 16px; padding-bottom:10px;
       border-bottom:1px solid var(--line); }
  h3 { font-size:16px; font-weight:500; margin:32px 0 10px; }
  .meta { padding-top:16px; border-top:1px solid var(--line); font-size:13px; color:var(--ink3); }
  code { font-family:var(--mono); font-size:.875em; background:var(--code-bg);
         padding:2px 5px; border-radius:4px; color:#A32D2D; }
  .fig { background:var(--panel); border:1px solid var(--line); border-radius:14px;
         padding:20px 16px 12px; margin:24px 0; }
  .fig svg { display:block; width:100%; height:auto; }
  .cap { font-size:12px; color:var(--ink3); text-align:center; margin-top:12px;
         padding-top:10px; border-top:1px dashed var(--line); }
  table { width:100%; border-collapse:collapse; font-size:13.5px; }
  th, td { text-align:left; padding:9px 11px; border-bottom:1px solid var(--line); }
  th { font-weight:500; font-size:12.5px; color:var(--ink2); background:#F7F6F2; }
</style>
</head>
<body>
<div class="wrap">

  <h1>示例文档</h1>
  <p class="meta">2026-10-28 · 6 张表 · 单进程</p>

  <h2>1 · 架构</h2>
  <div class="fig">
    <!-- 6.2 的架构图粘在这里 -->
    <div class="cap">图 1 · 三层：人、控制面、执行面</div>
  </div>

  <h2>2 · 流程</h2>
  <div class="fig">
    <!-- 6.3 的流程图粘在这里 -->
    <div class="cap">图 2 · 失败回退到上一环</div>
  </div>

  <h2>3 · 状态</h2>
  <div class="fig">
    <!-- 6.4 的状态图粘在这里 -->
    <div class="cap">图 3 · 失败可重试</div>
  </div>

  <h2>4 · 对照表</h2>
  <table>
    <thead><tr><th style="width:24%">项</th><th style="width:30%">做法</th><th>理由</th></tr></thead>
    <tbody>
      <tr><td>下行</td><td>SSE</td><td>单向推送足够，不需要 WebSocket</td></tr>
      <tr><td>上行</td><td>HTTP POST</td><td>批量上报，一条一请求会把写锁排队放大</td></tr>
    </tbody>
  </table>

</div>
</body>
</html>
```

### 6.2 架构图

三层纵向堆叠：一层一个色系，跨层用中性灰箭头，层内用「白底 + 主色描边」的小块。

```html
<svg viewBox="0 0 680 268" role="img" xmlns="http://www.w3.org/2000/svg" style="font-family:inherit">
  <defs>
    <marker id="ar" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <rect x="270" y="12" width="140" height="40" rx="8" fill="#ffffff" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="340" y="32" text-anchor="middle" dominant-baseline="central" font-size="13" fill="#2C2C2A">人 · 看板</text>
  <line x1="340" y1="52" x2="340" y2="82" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#ar)"/>

  <rect x="40" y="88" width="600" height="84" rx="14" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="58" y="108" font-size="11.5" fill="#185FA5">控制面 · 单进程</text>

  <rect x="58" y="118" width="170" height="42" rx="8" fill="#ffffff" stroke="#185FA5" stroke-width="0.5"/>
  <text x="143" y="139" text-anchor="middle" dominant-baseline="central" font-size="12.5" font-weight="500" fill="#0C447C">issue API</text>

  <rect x="248" y="118" width="184" height="42" rx="8" fill="#ffffff" stroke="#185FA5" stroke-width="0.5"/>
  <text x="340" y="139" text-anchor="middle" dominant-baseline="central" font-size="12.5" font-weight="500" fill="#0C447C">stage driver</text>

  <rect x="452" y="118" width="170" height="42" rx="8" fill="#ffffff" stroke="#185FA5" stroke-width="0.5"/>
  <text x="537" y="139" text-anchor="middle" dominant-baseline="central" font-size="12.5" font-weight="500" fill="#0C447C">事件流 · SSE</text>

  <line x1="170" y1="172" x2="170" y2="212" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#ar)" marker-start="url(#ar)"/>
  <rect x="40" y="214" width="260" height="44" rx="8" fill="#F1EFE8" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="170" y="236" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#444441">SQLite · data/app.db · WAL</text>

  <line x1="490" y1="172" x2="490" y2="212" stroke="#0F6E56" stroke-width="1.5" marker-end="url(#ar)"/>
  <text x="498" y="200" font-size="11" fill="#5F5E5A">job 下行 / event 上行</text>
  <rect x="340" y="214" width="300" height="44" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
  <text x="490" y="236" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#085041">daemon · 你的机器</text>
</svg>
```

坐标不必精细到像素，但要守三条：**框宽取整、间距相等、文字居中**。三个内框依次是 `58 / 248 / 452`，间隔 20 —— 只要间隔统一，看起来就是齐的。

### 6.3 流程图（带回退边）

正向一条直线，回退用虚线拱到节点上方。节点里两行：中文名 + 类型。

```html
<svg viewBox="0 0 680 150" role="img" xmlns="http://www.w3.org/2000/svg" style="font-family:inherit">
  <defs>
    <marker id="af" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <rect x="40" y="80" width="96" height="48" rx="8" fill="#F1EFE8" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="88" y="98" text-anchor="middle" dominant-baseline="central" font-size="12" font-weight="500" fill="#2C2C2A">澄清</text>
  <text x="88" y="115" text-anchor="middle" dominant-baseline="central" font-size="11" fill="#5F5E5A">gate</text>

  <line x1="136" y1="104" x2="164" y2="104" stroke="#888780" stroke-width="1.5" marker-end="url(#af)"/>

  <rect x="166" y="80" width="96" height="48" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="214" y="98" text-anchor="middle" dominant-baseline="central" font-size="12" font-weight="500" fill="#0C447C">用例生成</text>
  <text x="214" y="115" text-anchor="middle" dominant-baseline="central" font-size="11" fill="#185FA5">agent</text>

  <line x1="262" y1="104" x2="290" y2="104" stroke="#888780" stroke-width="1.5" marker-end="url(#af)"/>

  <rect x="292" y="80" width="96" height="48" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="340" y="98" text-anchor="middle" dominant-baseline="central" font-size="12" font-weight="500" fill="#0C447C">脚本生成</text>
  <text x="340" y="115" text-anchor="middle" dominant-baseline="central" font-size="11" fill="#185FA5">agent</text>

  <line x1="388" y1="104" x2="416" y2="104" stroke="#888780" stroke-width="1.5" marker-end="url(#af)"/>

  <rect x="418" y="80" width="96" height="48" rx="8" fill="#F1EFE8" stroke="#5F5E5A" stroke-width="0.5"/>
  <text x="466" y="98" text-anchor="middle" dominant-baseline="central" font-size="12" font-weight="500" fill="#2C2C2A">审脚本</text>
  <text x="466" y="115" text-anchor="middle" dominant-baseline="central" font-size="11" fill="#5F5E5A">gate</text>

  <line x1="514" y1="104" x2="542" y2="104" stroke="#888780" stroke-width="1.5" marker-end="url(#af)"/>

  <rect x="544" y="80" width="96" height="48" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
  <text x="592" y="98" text-anchor="middle" dominant-baseline="central" font-size="12" font-weight="500" fill="#085041">运行</text>
  <text x="592" y="115" text-anchor="middle" dominant-baseline="central" font-size="11" fill="#0F6E56">job</text>

  <path d="M592 80 V56 Q592 46 582 46 H350 Q340 46 340 56 V80"
        fill="none" stroke="#888780" stroke-width="1.5" stroke-dasharray="4 3" marker-end="url(#af)"/>
  <text x="466" y="36" text-anchor="middle" font-size="11" fill="#888780">失败 · script_bug</text>
</svg>
```

回退边的写法是固定的：**从节点上沿出发 → 竖直上移 24 → 圆角拐弯 → 横向 → 再拐回目标上沿**。
`Q` 的半径用 10，虚线 `4 3`，颜色统一 `#888780` —— 视觉上退到背景，不抢主线。

### 6.4 状态图

横向一条主干，异常分支往下挂，往回指的虚线表示重试。

```html
<svg viewBox="0 0 680 186" role="img" xmlns="http://www.w3.org/2000/svg" style="font-family:inherit">
  <defs>
    <marker id="as" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <rect x="40" y="30" width="120" height="44" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="100" y="52" text-anchor="middle" dominant-baseline="central" font-size="13" fill="#0C447C">queued</text>
  <line x1="160" y1="52" x2="196" y2="52" stroke="#185FA5" stroke-width="1.5" marker-end="url(#as)"/>

  <rect x="200" y="30" width="120" height="44" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="260" y="52" text-anchor="middle" dominant-baseline="central" font-size="13" fill="#0C447C">claimed</text>
  <line x1="320" y1="52" x2="356" y2="52" stroke="#185FA5" stroke-width="1.5" marker-end="url(#as)"/>

  <rect x="360" y="30" width="120" height="44" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="420" y="52" text-anchor="middle" dominant-baseline="central" font-size="13" font-weight="500" fill="#0C447C">running</text>
  <line x1="480" y1="52" x2="516" y2="52" stroke="#185FA5" stroke-width="1.5" marker-end="url(#as)"/>

  <rect x="520" y="30" width="120" height="44" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
  <text x="580" y="52" text-anchor="middle" dominant-baseline="central" font-size="13" fill="#085041">succeeded</text>

  <line x1="420" y1="74" x2="420" y2="110" stroke="#A32D2D" stroke-width="1.5" marker-end="url(#as)"/>
  <rect x="360" y="112" width="120" height="44" rx="8" fill="#FCEBEB" stroke="#A32D2D" stroke-width="0.5" stroke-dasharray="4 3"/>
  <text x="420" y="134" text-anchor="middle" dominant-baseline="central" font-size="13" fill="#A32D2D">failed</text>

  <path d="M360 134 H270 Q260 134 260 124 V74"
        fill="none" stroke="#A32D2D" stroke-width="1.5" stroke-dasharray="4 3" marker-end="url(#as)"/>
  <text x="300" y="124" text-anchor="middle" font-size="11" fill="#888780">重试</text>
</svg>
```

异常状态用**红系**（`#A32D2D` / 底 `#FCEBEB`）并与正常路径区分线型：正常实线、异常或重试虚线。四个状态横向排开时宽度取 120、间隔 40，正好落在 680 里。

---

## 七、使用方式

把第一节整段贴给任意模型，再把你的素材（大纲、DDL、接口表、旧文档）附在后面，说一句：

> 按这个风格写一份《XXX》设计文档。

要把图交给我画，补一句「图按第六节的三种画法」—— 它比一句「画得好看点」有效得多。

如果要改视觉，只动第一节的色值和字号，别动「决定性三条」。
