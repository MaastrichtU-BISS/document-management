<template>
  <div class="du-shell">
    <div class="du-header">
      <span class="du-eyebrow">document corpus</span>
      <h3 class="du-title">Upload documents</h3>
    </div>

    <div class="du-dropzone" :class="{ 'du-dropzone--over': dragOver }"
      @dragover.prevent="dragOver = true"
      @dragleave="dragOver = false"
      @drop.prevent="handleDrop"
    >
      <span class="du-icon">⤓</span>
      <p class="du-drop-text">Drop files here or <label class="du-file-label">browse<input type="file" :accept="acceptAttr" multiple @change="handleFiles" /></label></p>
      <span class="du-drop-hint">{{ acceptLabel }} · {{ sizeLabel }} · parsed in-browser</span>
    </div>

    <!-- PDF page picker: shown one queued PDF at a time -->
    <div v-if="pdf" class="du-pdf">
      <div class="du-pdf-head">
        <span class="mono">{{ pdf.name }} — select pages</span>
        <span class="du-pdf-actions">
          <button type="button" class="du-btn" @click="toggleAllPages">{{ allSelected ? 'None' : 'All' }}</button>
          <button type="button" class="du-btn du-btn--primary" :disabled="!selectedPages.length" @click="confirmPdf">Add {{ selectedPages.length }} page(s)</button>
          <button type="button" class="du-btn" @click="skipPdf">Skip</button>
        </span>
      </div>
      <div class="du-pdf-grid">
        <label v-for="p in pdf.thumbs" :key="p.page" class="du-thumb" :class="{ 'du-thumb--on': pageSet.has(p.page) }">
          <img :src="p.src" :alt="'page ' + p.page" />
          <span class="du-thumb-cap"><input type="checkbox" :checked="pageSet.has(p.page)" @change="togglePage(p.page)" /> {{ p.page }}</span>
        </label>
      </div>
    </div>

    <p v-if="busy" class="du-status mono">Reading {{ busy }} file(s)…</p>
    <p v-else-if="lastAdded" class="du-status mono">Added {{ lastAdded }} document(s) to the corpus.</p>
    <p v-if="rejected.length" class="du-status du-status--error mono">
      Skipped {{ rejected.length }} file(s): {{ rejected.join(', ') }}
    </p>
  </div>
</template>

<script setup>
import { ref, computed, markRaw } from 'vue';

// Config threaded from legalblocks.config.json (see pages/documents.vue). Nothing
// about storage lives here — the widget only turns files into document records and
// emits them; the page persists them through the artifact bus.
const props = defineProps({
  accept: { type: String, default: 'txt,md,html,pdf,docx' },
  maxSizeMb: { type: [Number, String], default: 0 },
  enablePdf: { type: [Boolean, String], default: true },
  enableDocx: { type: [Boolean, String], default: true },
});
const emit = defineEmits(['add']);
const dragOver = ref(false);

const truthy = (v) => v === true || v === 'on' || v === 'true' || v === 1 || v === '1';
const pdfOn = computed(() => truthy(props.enablePdf));
const docxOn = computed(() => truthy(props.enableDocx));

const extensions = computed(() =>
  props.accept.split(',').map((s) => s.trim().toLowerCase()).filter(Boolean)
);
const acceptAttr = computed(() => extensions.value.map((e) => '.' + e).join(','));
const acceptLabel = computed(() => extensions.value.map((e) => e.toUpperCase()).join(', '));

const maxBytes = computed(() => {
  const mb = Number(props.maxSizeMb);
  return Number.isFinite(mb) && mb > 0 ? mb * 1024 * 1024 : 0;
});
const sizeLabel = computed(() =>
  maxBytes.value ? `max ${Number(props.maxSizeMb)} MB/file` : 'no size limit'
);

const busy = ref(0);
const lastAdded = ref(0);
const rejected = ref([]);

const MIME = {
  txt: 'text/plain', md: 'text/markdown', html: 'text/html',
  pdf: 'application/pdf',
  docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
};
const extOf = (name) => (name.split('.').pop() || '').toLowerCase();

// --- lazy CDN parsers (generated apps only depend on nuxt) --------------------
let _pdfjs = null;
async function getPdfjs() {
  if (!_pdfjs) {
    _pdfjs = await import(/* @vite-ignore */ 'https://esm.sh/pdfjs-dist@4');
    _pdfjs.GlobalWorkerOptions.workerSrc = `https://cdn.jsdelivr.net/npm/pdfjs-dist@${_pdfjs.version}/build/pdf.worker.min.mjs`;
  }
  return _pdfjs;
}
async function getMammoth() {
  const m = await import(/* @vite-ignore */ 'https://esm.sh/mammoth@1');
  return m.default ?? m;
}

async function sha1Hex(text) {
  try {
    const buf = await crypto.subtle.digest('SHA-1', new TextEncoder().encode(text));
    return [...new Uint8Array(buf)].map((b) => b.toString(16).padStart(2, '0')).join('');
  } catch {
    return '';
  }
}

function makeRecord(name, ext, full_text) {
  return {
    id: 'doc-' + Date.now().toString(36) + '-' + Math.random().toString(36).slice(2, 6),
    name,
    source: 'local_upload',
    mime: MIME[ext] || 'application/octet-stream',
    size: new Blob([full_text]).size,
    full_text,
    created_at: new Date().toISOString(),
  };
}

async function emitDocs(records) {
  const docs = records.filter((d) => d && d.full_text && d.full_text.trim());
  for (const d of docs) d.hash = await sha1Hex(d.full_text);
  if (docs.length) {
    lastAdded.value = docs.length;
    emit('add', docs);
  }
}

// --- PDF page-picker state ----------------------------------------------------
const pdf = ref(null); // { name, doc, thumbs:[{page,src}] }
const pdfQueue = ref([]);
const pageSet = ref(new Set());
const selectedPages = computed(() => [...pageSet.value].sort((a, b) => a - b));
const allSelected = computed(() => pdf.value && pageSet.value.size === pdf.value.doc.numPages);

function togglePage(p) {
  const s = new Set(pageSet.value);
  s.has(p) ? s.delete(p) : s.add(p);
  pageSet.value = s;
}
function toggleAllPages() {
  if (!pdf.value) return;
  pageSet.value = allSelected.value
    ? new Set()
    : new Set(Array.from({ length: pdf.value.doc.numPages }, (_, i) => i + 1));
}

async function openNextPdf() {
  pdf.value = null;
  const file = pdfQueue.value.shift();
  if (!file) return;
  try {
    const pdfjs = await getPdfjs();
    const doc = await pdfjs.getDocument({ data: await file.arrayBuffer() }).promise;
    const thumbs = [];
    for (let n = 1; n <= doc.numPages; n++) {
      const page = await doc.getPage(n);
      const viewport = page.getViewport({ scale: 0.3 });
      const canvas = document.createElement('canvas');
      canvas.width = viewport.width;
      canvas.height = viewport.height;
      await page.render({ canvasContext: canvas.getContext('2d'), viewport }).promise;
      thumbs.push({ page: n, src: canvas.toDataURL('image/jpeg', 0.7) });
    }
    pageSet.value = new Set(thumbs.map((t) => t.page)); // all selected by default
    // markRaw: pdf.js's PDFDocumentProxy uses private class fields (#s) that throw
    // when accessed through a Vue reactive Proxy, so it must never be made reactive.
    pdf.value = { name: file.name, doc: markRaw(doc), thumbs };
  } catch {
    rejected.value.push(`${file.name} (unreadable PDF)`);
    await openNextPdf();
  }
}

async function extractPageText(doc, n) {
  const page = await doc.getPage(n);
  const content = await page.getTextContent();
  return content.items
    .filter((it) => 'str' in it)
    .map((it) => it.str + (it.hasEOL ? '\n' : ''))
    .join('')
    .replace(/ {2,}/g, ' ')
    .trim();
}

async function confirmPdf() {
  const { name, doc } = pdf.value;
  const pages = selectedPages.value;
  pdf.value = null;
  busy.value = 1;
  const parts = [];
  for (const n of pages) parts.push(`— Page ${n} —\n${await extractPageText(doc, n)}`);
  busy.value = 0;
  await emitDocs([makeRecord(name, 'pdf', parts.join('\n\n'))]);
  await openNextPdf();
}
async function skipPdf() {
  pdf.value = null;
  await openNextPdf();
}

// --- ingestion ----------------------------------------------------------------
async function readOne(file) {
  const ext = extOf(file.name);
  if (ext === 'docx') {
    const mammoth = await getMammoth();
    const { value } = await mammoth.extractRawText({ arrayBuffer: await file.arrayBuffer() });
    return makeRecord(file.name, ext, value ?? '');
  }
  // txt / md / html and anything else readable as text
  return makeRecord(file.name, ext, await file.text());
}

async function ingest(fileList) {
  const all = Array.from(fileList ?? []);
  if (!all.length) return;
  rejected.value = [];

  // Size + type gating first, before reading anything.
  const accepted = [];
  for (const f of all) {
    const ext = extOf(f.name);
    if (extensions.value.length && !extensions.value.includes(ext)) {
      rejected.value.push(`${f.name} (.${ext} not allowed)`);
    } else if ((ext === 'pdf' && !pdfOn.value) || (ext === 'docx' && !docxOn.value)) {
      rejected.value.push(`${f.name} (${ext.toUpperCase()} disabled)`);
    } else if (maxBytes.value && f.size > maxBytes.value) {
      rejected.value.push(`${f.name} (${(f.size / 1024 / 1024).toFixed(1)} MB)`);
    } else {
      accepted.push(f);
    }
  }

  // PDFs go through the page picker; the rest are read straight away.
  const pdfs = accepted.filter((f) => extOf(f.name) === 'pdf');
  const rest = accepted.filter((f) => extOf(f.name) !== 'pdf');

  if (rest.length) {
    busy.value = rest.length;
    const records = [];
    for (const f of rest) {
      try {
        records.push(await readOne(f));
      } catch {
        rejected.value.push(`${f.name} (parse failed)`);
      }
    }
    busy.value = 0;
    await emitDocs(records);
  }

  if (pdfs.length) {
    pdfQueue.value.push(...pdfs);
    if (!pdf.value) await openNextPdf();
  }
}

const handleFiles = (e) => {
  ingest(e.target.files);
  e.target.value = '';
};
const handleDrop = (e) => {
  dragOver.value = false;
  ingest(e.dataTransfer?.files);
};
</script>

<style scoped>
.du-shell { font-family: system-ui, sans-serif; padding: 20px 0; display: flex; flex-direction: column; gap: 14px; }
.du-eyebrow { font-family: monospace; font-size: 10px; font-weight: 500; letter-spacing: 0.14em; text-transform: uppercase; color: #8C8579; }
.du-title { font-size: 16px; font-weight: 600; margin: 4px 0 0; color: #1A1714; }

.du-dropzone {
  border: 2px dashed #E5DFD3; border-radius: 12px; padding: 28px;
  display: flex; flex-direction: column; align-items: center; gap: 8px;
  background: #FBF8F2; transition: border-color 0.15s, background 0.15s; cursor: pointer;
}
.du-dropzone--over { border-color: #D4421E; background: #FDE8E0; }
.du-icon { font-size: 26px; }
.du-drop-text { font-size: 14px; color: #4A453E; margin: 0; }
.du-file-label { color: #D4421E; font-weight: 600; cursor: pointer; }
.du-file-label input { display: none; }
.du-drop-hint { font-size: 11px; color: #8C8579; font-family: monospace; text-align: center; }

.du-pdf { border: 1px solid #E5DFD3; border-radius: 12px; padding: 14px; background: #fff; display: flex; flex-direction: column; gap: 12px; }
.du-pdf-head { display: flex; justify-content: space-between; align-items: center; gap: 12px; font-size: 12px; color: #4A453E; }
.du-pdf-actions { display: flex; gap: 6px; }
.du-btn { font-size: 11px; padding: 4px 10px; border: 1px solid #E5DFD3; border-radius: 6px; background: #FBF8F2; color: #4A453E; cursor: pointer; }
.du-btn--primary { background: #D4421E; border-color: #D4421E; color: #fff; }
.du-btn:disabled { opacity: 0.5; cursor: default; }
.du-pdf-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(96px, 1fr)); gap: 10px; max-height: 300px; overflow-y: auto; }
.du-thumb { display: flex; flex-direction: column; align-items: center; gap: 4px; border: 2px solid transparent; border-radius: 8px; padding: 4px; cursor: pointer; }
.du-thumb--on { border-color: #D4421E; background: #FDE8E0; }
.du-thumb img { width: 100%; border: 1px solid #E5DFD3; border-radius: 4px; }
.du-thumb-cap { font-size: 10px; font-family: monospace; color: #4A453E; display: flex; align-items: center; gap: 4px; }

.du-status { font-size: 11px; color: #5F7A3A; margin: 0; }
.du-status--error { color: #D4421E; }
</style>
