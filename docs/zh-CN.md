# 在浏览器里做 PDF 工具：pdf-lib 与 pdfjs-dist 踩过的 7 个坑

> 在线演示：**[pdf.iceai.cloud](https://pdf.iceai.cloud)** — 免费、免注册，文件不上传。

用 `pdf-lib` + `pdfjs-dist` 做一个纯前端 PDF 工具站时，真正卡住我的地方都不在文档里。
每个坑至少花掉半天。记下来，省下一个人的时间。英文版见 [README.md](../README.md)。

---

## 1. `copyPages` 会静默丢掉 AcroForm 表单域

**现象** — 合并两个带表单的 PDF，页面内容都在，字段框也画出来了，但一打字就发现填不进去，
用 Acrobat 打开值也是空的。

**原因** — `copyPages` 复制的是页面内容流和页面上的注释（annotation）。表单字段定义在**文档级**
的 `/AcroForm` 字典里，不在页面上。复制过来的 widget 仍指向一个在新文档里并不存在的字段名，
于是渲染成一个「死框」。

**修法** — 自己重建表单字典，并且给不同源文件的字段加前缀避免重名：

```ts
import { PDFDocument, PDFName, PDFDict } from 'pdf-lib';

async function mergeKeepingForms(files: ArrayBuffer[]) {
  const out = await PDFDocument.create();
  const form = out.getForm();
  let n = 0;

  for (const bytes of files) {
    const src = await PDFDocument.load(bytes, { ignoreEncryption: true });
    const pages = await out.copyPages(src, src.getPageIndices());
    pages.forEach((p) => out.addPage(p));

    // 关键：手动把源文档字段「翻译」到目标文档，并加前缀
    for (const field of src.getForm().getFields()) {
      const renamed = `f${n++}_${field.getName()}`;
      if (form.getFieldMaybe(renamed)) continue;
      const type = field.constructor.name;
      if (type === 'PDFTextField') form.createTextField(renamed);
      else if (type === 'PDFCheckBox') form.createCheckBox(renamed);
    }
  }

  form.updateFieldAppearances();
  return out.save();
}
```

如果根本不需要可填表单，那就**把 widget 删掉**而不是重建 —— 一个能点但填不进去的框，
比一张拍平的页面更糟：

```ts
const annots = page.node.Annots();
if (annots) {
  for (let i = annots.size() - 1; i >= 0; i--) {
    const a = annots.lookup(i) as PDFDict;
    if (a.get(PDFName.of('Subtype'))?.toString() === '/Widget') annots.remove(i);
  }
}
```

---

## 2. 「压缩」是重新采样图片，不是重新编码文件

**现象** — 一个 12MB 的纯文字合同，调任何质量参数都几乎不变；一个 3MB 的扫描发票，
一压就糊成一片。

**原因** — PDF 的体积由两件完全不同的事决定：

| 内容类型 | 体积在哪 | 什么才有用 |
|---|---|---|
| 扫描件 / 图片页 | 页面 XObject 里的 DCTDecode(JPEG) 或 FlateDecode 位图 | 降采样 + 重新编码图片 |
| 文字 / 矢量页 | 嵌入字体子集（`FontFile2` / `FontFile3`） | 基本无解，再子集化也只有个位数百分比 |

重新编码整个文件不会让它变小。必须遍历 XObject 树，替换图片流。

```ts
page.node.Resources()?.lookup(PDFName.of('XObject'))?.entries().forEach(([key, ref]) => {
  const stream = out.context.lookup(ref);
  if (stream.dict.get(PDFName.of('Subtype'))?.toString() !== '/Image') return;
  const filter = stream.dict.get(PDFName.of('Filter'))?.toString();
  // DCTDecode = 已经是 JPEG；FlateDecode = 无损位图。两种都要降采样后重压
});
```

两个实践经验：

- 别把 `save({ useObjectStreams: true })` 当压缩用，最多省几个百分点。
- 让用户看到预估结果再决定。预估 = 原始体积 − 被移除的图片字节 + 重编码后的字节，
  精度足够给人参考，也能阻止用户拿纯文字 PDF 反复试。

---

## 3. `pdfjs-dist` 的 worker 必须同源

**现象** — dev 正常，打包上线后报
`Failed to construct 'Worker': Script at 'https://cdn.../pdf.worker.min.mjs' cannot be accessed from origin`。
更坏的情况是它静默退回主线程，解析大文件时页面直接假死。

**原因** — `new Worker()` 只接受同源脚本（module worker 也不例外）。
`GlobalWorkerOptions.workerSrc` 指向 CDN 是最常见的错因。

**修法** — 把 worker 打进自己的静态资源：

```ts
import * as pdfjs from 'pdfjs-dist';
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url'; // Vite

pdfjs.GlobalWorkerOptions.workerSrc = workerUrl;
```

webpack 用 `new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url).toString()`。
一定要在**构建产物**里确认这个请求是 200 且同源 —— 这里 404 会退化成主线程解析，
只有处理 40MB 文件时才会暴露。

---

## 4. 大文件直接打死标签页

**现象** — 页面卡住，然后 Safari 提示「此网页因为占用大量内存已被重新载入」。

**原因** — 文件本身在内存里一份（`File`），转 `ArrayBuffer` 又一份，解析成对象图再一份，
每渲染一页再加一个 canvas。150MB 扫描件峰值轻松超过 1.5GB，手机上必挂。

**修法** — 四件事都要做：

```ts
// a) 分页渲染，渲染完立刻清理
const page = await doc.getPage(i);
await page.render({ canvasContext: ctx, viewport }).promise;
page.cleanup();

// b) 用完销毁，释放 worker 侧缓存
doc.destroy();

// c) 重活丢进 Worker，主线程只负责 UI，否则解析期间任何交互都没响应

// d) 先看大小，超阈值明确拒绝，而不是让它慢慢 OOM
const LIMIT_MB = 80;
if (file.size > LIMIT_MB * 1048576) {
  throw new Error(`文件 ${(file.size / 1048576).toFixed(0)} MB，超过浏览器端上限 ${LIMIT_MB} MB`);
}
```

(d) 对产品体验的意义比看起来大：一句「太大了，原因是这个」远好过一次崩溃，
而且它诚实地承认了浏览器能做什么、不能做什么。

---

## 5. `splitTextToSize` 对中文断行是错的

**现象** — `drawText` 画中文时要么整段溢出边界，要么在莫名其妙的位置换行。

**原因** — `splitTextToSize` 按空格断行。中文没有空格，于是整段被当成一个「词」。
而且它不处理避头尾规则（标点不能出现在行首）。

**修法** — 按测量宽度逐字断行，加两条断行规则：

```ts
function wrapCJK(text: string, font: PDFFont, size: number, maxWidth: number): string[] {
  const lines: string[] = [];
  let line = '';
  for (const ch of text) {
    if (ch === '\n') { lines.push(line); line = ''; continue; }
    const next = line + ch;
    if (font.widthOfTextAtSize(next, size) > maxWidth && line) {
      if (/[，。、；：？！）》」』]/.test(ch)) {     // 避头点：标点挤回上一行
        lines.push(line + ch); line = '';
      } else {
        lines.push(line); line = ch;
      }
    } else line = next;
  }
  if (line) lines.push(line);
  return lines;
}
```

`widthOfTextAtSize` 是唯一可靠的度量。不要假设中文字形宽度正好是 `1em` —— 大多数字体都不是，
误差累积得很快。

---

## 6. iOS Safari 没有 `showSaveFilePicker`，blob URL 还会漏内存

**现象** — 桌面能下载，iOS Safari 点了没反应；或者保存几次之后页面自己刷新了。

**原因** — File System Access API 只有 Chromium 有。iOS 上必须回退到 `<a download>` + blob URL。
而 blob URL 在 revoke 之前会一直钉住那块内存 —— 一个没回收的 100MB blob，就是 Safari
会通过杀掉你的标签页来回收的 100MB。

```ts
async function saveBlob(blob: Blob, filename: string) {
  if ('showSaveFilePicker' in window) {
    try {
      const handle = await (window as any).showSaveFilePicker({
        suggestedName: filename,
        types: [{ description: 'PDF', accept: { 'application/pdf': ['.pdf'] } }],
      });
      const w = await handle.createWritable();
      await w.write(blob);
      await w.close();
      return;
    } catch (e) {
      if ((e as DOMException).name === 'AbortError') return; // 用户取消，不要弹错
    }
  }

  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = filename; a.rel = 'noopener';
  document.body.appendChild(a);
  a.click();
  a.remove();
  // 不要立刻 revoke：部分浏览器此时下载还没真正开始，会得到 0 字节文件
  setTimeout(() => URL.revokeObjectURL(url), 60_000);
}
```

那 60 秒不是 hack，它是「下载下来的文件是 0 字节」这个 bug 的修法。

---

## 7. 加密 PDF 抛的错五花八门

**现象** — 用户拖进一个带密码的 PDF，有时是 `EncryptedPDFError`，有时是
`Failed to parse PDF document`，有时是 pako 的 inflate 报错 —— 取决于它是怎么加密的。

**原因** — 加 `ignoreEncryption: true` 能加载，但拿到的文档流解不开；不加就抛
`EncryptedPDFError`。还有一种「只有权限密码、没有打开密码」的情况（限制打印/复制），
它能正常加载，但你不能假装那些限制不存在。

**修法** — 先分类，再给出可操作的提示：

```ts
async function loadPdf(bytes: ArrayBuffer) {
  try {
    return await PDFDocument.load(bytes);
  } catch (e: any) {
    const msg = String(e?.message ?? e);
    if (/encrypt/i.test(msg) || e?.name === 'EncryptedPDFError')
      throw new Error('这个 PDF 有打开密码，请输入密码后重试');
    if (/Invalid PDF|Failed to parse/i.test(msg))
      throw new Error('文件不是有效的 PDF，或者已损坏');
    if (/password/i.test(msg))
      throw new Error('密码不正确');
    throw new Error('无法读取这个文件，可以试试用其他工具打开确认是否正常');
  }
}
```

永远不要把库的原始报错直接抛给用户。`EncryptedPDFError` 用户无法处理，
「有打开密码，请输入」可以。

---

## 附：`page.getSize()` 不包含旋转

页面带 `/Rotate 90` 时，`getSize()` 返回的仍是未旋转的 media box。竖版页面被显示成横版，
你算出来的宽高是反的，任何基于它的排版（页边距、图片位置、水印坐标）都会整体转 90°。

```ts
const rotate = page.getRotation().angle % 360;   // 0 | 90 | 180 | 270
const { width, height } = page.getSize();
const visualW = rotate === 90 || rotate === 270 ? height : width;
const visualH = rotate === 90 || rotate === 270 ? width : height;
```

---

## 许可

MIT。有新坑欢迎提 issue —— 能绊倒你的，就该进这份清单。
