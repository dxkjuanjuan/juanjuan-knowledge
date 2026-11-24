# HTML 工程规范与质检

## 基本约束

- **单文件**：CSS、JS 全内联，零外链、零 CDN。用户可能断网，也可能直接发微信给人
- **中文字体栈**：`-apple-system,BlinkMacSystemFont,"PingFang SC","Hiragino Sans GB","Microsoft YaHei",sans-serif`
- **等宽字体栈**：`ui-monospace,SFMono-Regular,"SF Mono",Menlo,Consolas,monospace`
- **正文 15px / 行高 1.72**。打印时降到 10.5pt / 1.55
- **容器 max-width 940px**。比这个宽，A4 上会撑破

配色只用六个语义色，别自创：

```css
--ink:#1c2530;      /* 正文 */
--ink2:#4a5768;     /* 次要文字 */
--line:#dde3ec;     /* 边框 */
--blue:#1f4e79;     /* 术语、关键词 */
--red:#b3372c;      /* 考点、警告 */
--green:#1c6b47;    /* 正确、要点 */
```

---

## 打印样式：三条必须

### 1. 删掉所有入场动画（最容易致命）

```css
/* 会导致整本 PDF 变白纸 */
.paper{display:none;animation:fade .2s ease}
@keyframes fade{from{opacity:0}to{opacity:1}}
```

Chrome 在 `print-to-pdf` 时会重新触发动画，截图截在 opacity:0 那一帧。**症状**：PDF 页数正常但每页空白。**修法**：去掉动画，只用 `display:none/block`。

用之前验证：`pdftotext -f 2 -l 2 out.pdf -` 看有没有内容。有页数没文字，基本就是这个问题。

### 2. 分页规则

```css
@media print{
  @page{margin:13mm 12mm}
  body{font-size:10.5pt;line-height:1.55}
  nav.toc{display:none}
  .wrap{max-width:100%;padding:0}

  /* 多份试卷/多章时用这条，不要用 :first-of-type */
  .paper + .paper{break-before:page;page-break-before:always}

  /* 不许拆开的小块 */
  .def,.ex-h,.box,.q{break-inside:avoid}
  h2,h3{break-after:avoid}

  pre{font-size:9pt;background:#f7f9fc !important}
  *{-webkit-print-color-adjust:exact;print-color-adjust:exact}
}
```

`.ex-h`（例题的题目头）必须 `break-inside:avoid`，否则会出现"题目在上一页底部、代码在下一页开头"。

### 3. 双栏表的打印尺寸

```css
@media print{
  .ct td.c1{font-size:8pt;line-height:1.48;padding:1px 7px 1px 9px}
  .ct td.c2{font-size:8.2pt;line-height:1.5;padding:1px 9px 1px 7px}
  .ct tr{break-inside:avoid}
}
```

双栏表行高必须一致，否则左右对不齐。**不要**给 `td` 设 `padding` 上下值不同，靠 `vertical-align:top` 对齐即可。

---

## 表格列宽塌陷

现象：四列表格的表头被压成一行一个字。

```css
/* 病态 */
table.tb th,table.tb td{padding:5px 9px}

/* 修好 */
table.tb th,table.tb td{padding:5px 9px;min-width:4em}
table.tb th{white-space:nowrap}
```

窄屏（<640px）媒体查询里要改回来，否则手机横向溢出：

```css
@media (max-width:640px){
  body{overflow-x:hidden}
  table.tb th{white-space:normal}
  table.tb th,table.tb td{min-width:0;padding:4px 6px;font-size:12.4px}
  pre{font-size:11.6px;padding:9px 10px}
}
```

---

## 质检四关

### 第一关：标签平衡

```bash
python3 - <<'PY'
import re
s=open('out.html',encoding='utf-8').read()
for tag in ['section','div','table','tr','td','pre','ul','li','p','span','b','code','h2','h3']:
    o=len(re.findall(r'<%s[\s>]'%tag,s)); c=len(re.findall(r'</%s>'%tag,s))
    print(f"{tag}: {o}/{c} {'OK' if o==c else '<<< MISMATCH'}")
PY
```

`span` 和 `b` 允许少量不匹配（浏览器容错），但 `div`/`table`/`tr`/`td` 必须严格相等。

### 第二关：JS 语法

有 JS 就必须跑，模板字符串里一个反引号写错，整个脚本静默失效。

```bash
python3 - <<'PY'
import re
s=open('out.html',encoding='utf-8').read()
m=re.search(r'<script>(.*)</script>', s, re.S)
open('/tmp/check.js','w',encoding='utf-8').write(m.group(1))
PY
node --check /tmp/check.js && echo "JS OK"
```

### 第三关：截图目检

```bash
cp out.html /tmp/preview.html
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --no-sandbox --hide-scrollbars \
  --virtual-time-budget=2500 --window-size=1000,2600 \
  --screenshot=/tmp/shot.png "file:///tmp/preview.html"
```

要看的四件事：双栏表左右是否对齐、表格列是否被压窄、代码是否横向溢出、图谱三列是否等宽。

**为什么复制到 /tmp**：中文路径下 Chrome 偶尔读不到文件，报错静默。

### 第四关：打印样张

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --no-sandbox \
  --no-pdf-header-footer --virtual-time-budget=3000 \
  --print-to-pdf=out.pdf "file:///tmp/preview.html"

# 页数
python3 -c "import re;d=open('out.pdf','rb').read();print('pages:',len(re.findall(rb'/Type\s*/Page[^s]',d)))"

# 抽查 3 页转图
pdftoppm -png -r 60 -f 14 -l 16 out.pdf /tmp/pg
```

抽哪三页：**中间某页**（表格和双栏最密集处）、**例题跨页处**、**最后一页**（小结框是否被切断）。

---

## 判断溢出的正确姿势

`--window-size=414` 时 headless 的实际 `innerWidth` 可能是 **500**，截图会骗你。要判断是否真溢出，注入调试条读数值：

```javascript
var d=document.createElement("div");
d.style.cssText="position:fixed;top:0;left:0;z-index:9999;background:#000;color:#0f0;font:14px monospace";
d.textContent="inner="+window.innerWidth+" scrollW="+document.documentElement.scrollWidth;
document.body.appendChild(d);
```

`scrollW > inner` 才是真溢出。

---

## 常见返工原因

| 症状 | 原因 | 修法 |
|---|---|---|
| PDF 全白 | 入场动画 | 删 `animation` |
| PDF 里只有一份内容 | `.paper` 都是 `display:none` | 打印媒体里 `.paper{display:block}` |
| 表格列变一个字宽 | 无 `min-width` | `td{min-width:4em}` |
| 双栏表左右错位 | 左右 `padding` 上下不等 | 统一 `vertical-align:top` |
| 例题题目和代码分居两页 | `.ex-h` 没设 avoid | `break-inside:avoid` |
| 打印后背景色没了 | 未开 color-adjust | `*{-webkit-print-color-adjust:exact}` |
| 中文路径截图失败 | Chrome 读不到 | 复制到 /tmp 再截 |

---

## 交付前最后一遍

- 封面上的数字（页数/条数/题数）用脚本重算后替换
- `grep -c "undefined\|NaN"` 应为 0
- 确认原件未被修改（`ls -la` 看时间戳）
- 同时给出 `.html`（可看可打印）和 `.pdf`（直接打印）
