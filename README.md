# browser-pdf-notes

> Live demo: **[pdf.iceai.cloud](https://pdf.iceai.cloud)** — free, no signup, files never leave your browser.
> (merge / split / compress / PDF↔image run entirely client-side; Word conversion and encryption need a server)

Seven things that broke while building a PDF toolbox that runs in the browser with
`pdf-lib` + `pdfjs-dist`. None of them are documented clearly in either library, and each one
cost me at least half a day. Written down so the next person doesn't.

中文版见 [docs/zh-CN.md](docs/zh-CN.md)。

---

## 1. `copyPages` silently drops AcroForm fields

**Symptom** — merge two PDFs that contain form fields. The pages come over fine, the fields'
widgets are still visible on the page, but typing into them does nothing and the values are
gone in Acrobat.

**Why** — `copyPages` copies the page content stream and the page's annotations. Form fields
live in the *document-level* `/AcroForm` dictionary, not on the page. The copied widgets still
point at field names that no longer exist in the destination document, so they render as dead
boxes.

**Fix** — rebuild the form dictionary yourself, with a per-document prefix so field names from
different source files don't collide:

```ts
import { PDFDocument, PDFName, PDFDict, PDFString } from 'pdf-lib';

async function mergeKeepingForms(files: ArrayBuffer[]) {
  const out = await PDFDocument.create();
  const form = out.getForm();
  let n = 0;

  for (const bytes of files) {
    const src = await PDFDocument.load(bytes, { ignoreEncryption: true });
    const pages = await out.copyPages(src, src.getPageIndices());
    pages.forEach((p) => out.addPage(p));

    // 关键：手动把源文档的字段「翻译」到目标文档，并加前缀
    const srcForm = src.getForm();
    for (const field of srcForm.getFields()) {
      const name = field.getName();
      const renamed = `f${n++}_${name}`;
      const existing = form.getFieldMaybe(renamed);
      if (existing) continue;
      // 按字段类型重建：文本 / 复选 / 单选
      const type = field.constructor.name;
      if (type === 'PDFTextField') {
        form.createTextField(renamed);
      } else if (type === 'PDFCheckBox') {
        form.createCheckBox(renamed);
      }
      // 只想保留「看得见、填不了」的话，宁可不要 copyPages 带来的死 widget：
      // 提前删掉页面上的 widget 注释，避免用户以为能填
    }
  }

  form.updateFieldAppearances();
  return out.save();
}
```

If you don't need editable forms at all, strip the widgets instead of rebuilding — a dead
widget is worse than a flattened page, because it looks interactive.

```ts
// 删掉页面上所有 Widget 注释（表单控件框），让它们不再显示为可填
const annots = page.node.Annots();
if (annots) {
  for (let i = annots.size() - 1; i >= 0; i--) {
    const a = annots.lookup(i) as PDFDict;
    if (a.get(PDFName.of('Subtype'))?.toString() === '/Widget') annots.remove(i);
  }
}
```

---

## 2. "Compress" means "resample the images", not "re-encode the file"

**Symptom** — a naive quality slider does nothing to a 12 MB text-heavy contract, and destroys
a 3 MB scanned invoice.

**Why** — the size of a PDF is dominated by two different things depending on content:

| Content | What occupies the bytes | What actually helps |
|---|---|---|
| Scanned / image pages | DCTDecode (JPEG) or FlateDecode bitmaps inside page XObjects | downsampling + recompressing images |
| Text / vector pages | embedded font subsets (`FontFile2` / `FontFile3`) | almost nothing — subsetting further gives single-digit % |

Re-encoding the file does not shrink it. You have to walk the XObject tree and replace the
image streams.

**Fix** — a rough shape that gives 60–85% reduction on scans and leaves the text layer intact:

```ts
// 1) 遍历每页的 XObject，找出图片
// 2) 解码 → 按目标 DPI 降采样 → 用较低质量重新编码成 JPEG
// 3) 把新流写回，注意同步更新 /Width /Height /Filter /Length
page.node.Resources()?.lookup(PDFName.of('XObject'))?.entries().forEach(([key, ref]) => {
  const stream = out.context.lookup(ref);
  const subtype = stream.dict.get(PDFName.of('Subtype'))?.toString();
  if (subtype !== '/Image') return;

  const filter = stream.dict.get(PDFName.of('Filter'))?.toString();
  // DCTDecode = 已经就是 JPEG；FlateDecode = 无损位图，两种都要降采样
  // ...
});
```

Two practical notes:

- Do **not** just do `PDFDocument.load().save({ useObjectStreams: true })` and call it
  compression. It saves a few percent at best.
- Show the user the estimated result before they commit. Predicting size = baseline size −
  (removed image bytes) + (re-encoded image bytes) is accurate enough to be useful, and it
  stops people from hammering a text PDF expecting magic.

---

## 3. `pdfjs-dist` worker must be same-origin

**Symptom** — works in dev, fails in production build with
`Failed to construct 'Worker': Script at 'https://cdn.../pdf.worker.min.mjs' cannot be accessed
from origin 'https://yoursite.com'`. Or, worse, it silently falls back to the main thread and
freezes the tab.

**Why** — `new Worker()` only accepts same-origin scripts (module workers have no exception to
this). `GlobalWorkerOptions.workerSrc` pointing at a CDN is the single most common cause.

**Fix** — ship the worker with your own assets and reference it with a bundler-resolved URL:

```ts
import * as pdfjs from 'pdfjs-dist';
// Vite：?url 会把文件复制到 assets 并给出最终 URL
import workerUrl from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

pdfjs.GlobalWorkerOptions.workerSrc = workerUrl;
```

For webpack, `new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url).toString()`.
Whatever you use, **verify in the built output** that the request to the worker is 200 and
same-origin — a 404 here degrades to main-thread parsing and you'll only notice on a 40 MB file.

---

## 4. Large files kill the tab: `arrayBuffer()` on 100 MB+ is fatal

**Symptom** — the page hangs, then Safari shows "this webpage was reloaded because it was using
significant memory".

**Why** — the file is already in memory once as a `File`, again as an `ArrayBuffer`, again as
the parsed document object graph, plus a canvas per rendered page. A 150 MB scan can easily peak
above 1.5 GB. On mobile that is guaranteed death.

**Fix** — four things, all of them necessary:

```ts
// a) 分页渲染，渲染完立刻清理
const page = await doc.getPage(i);
const viewport = page.getViewport({ scale });
await page.render({ canvasContext: ctx, viewport }).promise;
page.cleanup();

// b) 解析时不要保留原始 buffer
doc.destroy();   // 页面全部渲染/提取完成后释放 worker 侧缓存

// c) 重活丢进 Worker，主线程只做 UI
//    否则解析期间任何交互都无响应，用户以为卡死了

// d) 先检查大小，超阈值就明确拒绝，而不是让它慢慢 OOM
const LIMIT_MB = 80;
if (file.size > LIMIT_MB * 1024 * 1024) {
  throw new Error(`文件 ${(file.size / 1048576).toFixed(0)} MB，超过浏览器端处理上限 ${LIMIT_MB} MB`);
}
```

Point (d) matters for product quality more than it looks. A clear "too big, here's why" beats a
tab crash, and it's honest about what a browser can do.

---

## 5. `splitTextToSize` is wrong for CJK

**Symptom** — Chinese text drawn with `page.drawText` either overflows the margin in one long
line, or breaks at bizarre places. `splitTextToSize` appears to treat a whole Chinese paragraph
as a single "word".

**Why** — `splitTextToSize` wraps on whitespace. Chinese has no whitespace, so there's nothing
to split on. The library also can't apply CJK line-breaking rules (no leading punctuation at
line end, no trailing at line start).

**Fix** — wrap by measured width, character by character, with a couple of line-break rules:

```ts
function wrapCJK(text: string, font: PDFFont, size: number, maxWidth: number): string[] {
  const lines: string[] = [];
  let line = '';

  for (const ch of text) {
    if (ch === '\n') { lines.push(line); line = ''; continue; }
    const next = line + ch;
    if (font.widthOfTextAtSize(next, size) > maxWidth && line) {
      // 避头点：标点不能出现在行首，把它挤回上一行
      if (/[，。、；：？！）》」』]/.test(ch)) {
        lines.push(line + ch);
        line = '';
      } else {
        lines.push(line);
        line = ch;
      }
    } else {
      line = next;
    }
  }
  if (line) lines.push(line);
  return lines;
}
```

`widthOfTextAtSize` is the only reliable measurement — don't try to assume CJK glyphs are
exactly `1em` wide, they aren't for most fonts, and the error accumulates fast.

---

## 6. iOS Safari: no `showSaveFilePicker`, and blob URLs leak memory

**Symptom** — download works on desktop, silently does nothing on iOS Safari. Or: after a few
saves the page reloads on its own.

**Why** — `showSaveFilePicker` (File System Access API) is Chromium-only. On iOS you must fall
back to an `<a download>` with a blob URL. And blob URLs pin their underlying memory until
revoked — a 100 MB blob that is never revoked is 100 MB that Safari will reclaim by killing your
tab.

**Fix** —

```ts
async function saveBlob(blob: Blob, filename: string) {
  // Chromium：优先用原生保存对话框，体验好且不占内存
  if ('showSaveFilePicker' in window) {
    try {
      const handle = await (window as any).showSaveFilePicker({
        suggestedName: filename,
        types: [{ description: 'PDF', accept: { 'application/pdf': ['.pdf'] } }],
      });
      const writable = await handle.createWritable();
      await writable.write(blob);
      await writable.close();
      return;
    } catch (e) {
      if ((e as DOMException).name === 'AbortError') return; // 用户取消，不要弹错
      // 其他错误走 fallback
    }
  }

  // Safari / Firefox：a[download] + blob URL，用完立刻撤销
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.rel = 'noopener';
  document.body.appendChild(a);
  a.click();
  a.remove();
  // 不要立刻 revoke：部分浏览器此时下载还没真正开始，会得到空文件
  setTimeout(() => URL.revokeObjectURL(url), 60_000);
}
```

The 60-second delay is not a hack, it's the fix for "downloaded file is 0 bytes" on Safari.

---

## 7. `PDFDocument.load` on an encrypted PDF throws — with several different errors

**Symptom** — user drops a password-protected PDF and sees `EncryptedPDFError: Input document to
PDFDocument.load is encrypted` — or a generic `Failed to parse PDF document`, or a pako inflate
error, depending on how it was encrypted.

**Why** — `ignoreEncryption: true` lets loading proceed but you get a document whose streams
can't be decoded. Without it, you get `EncryptedPDFError`. Encryption with an owner password only
(no user password, i.e. "restricted printing/copying") is a third case, and it loads fine but you
must not pretend the restrictions don't exist.

**Fix** — classify before doing anything else, and always show something actionable:

```ts
async function loadPdf(bytes: ArrayBuffer) {
  try {
    return await PDFDocument.load(bytes);
  } catch (e: any) {
    const msg = String(e?.message ?? e);
    if (/encrypt/i.test(msg) || e?.name === 'EncryptedPDFError') {
      // 需要用户输入密码；提示里说明「打开密码」和「权限密码」是两回事
      throw new Error('这个 PDF 有打开密码，请输入密码后重试');
    }
    if (/Invalid PDF|Failed to parse/i.test(msg)) {
      throw new Error('文件不是有效的 PDF，或者已损坏');
    }
    if (/password/i.test(msg)) {
      throw new Error('密码不正确');
    }
    throw new Error('无法读取这个文件，可以试试用其他工具打开确认是否正常');
  }
}
```

Never surface the raw library message. Users cannot act on `EncryptedPDFError`, but they can act
on "there's an open password, enter it".

---

## Bonus: `page.getSize()` ignores rotation

If a page has `/Rotate 90`, `getSize()` still returns the unrotated media box. A portrait page
displayed as landscape will give you swapped width/height, and any layout you compute
(margins, image placement, watermark position) will be wrong — rotated 90°.

```ts
const rotate = page.getRotation().angle % 360;   // 0 | 90 | 180 | 270
const { width, height } = page.getSize();
const visualW = rotate === 90 || rotate === 270 ? height : width;
const visualH = rotate === 90 || rotate === 270 ? width : height;
```

---

## Licence

MIT. Corrections and additional pitfalls are welcome as issues — if it tripped you up, it
belongs in this list.
