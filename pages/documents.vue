<template>
  <div class="lb-page page-documents">
    <div class="page-header">
      <div class="eyebrow">{{ screenCopy.eyebrow ?? 'document corpus' }}</div>
      <h1 class="serif page-title">{{ screenCopy.title ?? 'Documents' }}</h1>
    </div>

    <DocumentManagement
      :accept="allowedTypes"
      :max-size-mb="maxFileSizeMb"
      :enable-pdf="enablePdf"
      :enable-docx="enableDocx"
      @add="onAdd"
    />

    <div v-if="documents.length" class="doc-toolbar">
      <input v-model="query" class="doc-search mono" type="search" placeholder="Search by name…" />
      <button type="button" class="doc-danger" @click="clearAll">Delete all ({{ documents.length }})</button>
    </div>

    <div v-if="filtered.length" class="doc-table">
      <div class="doc-table-head">
        <span>Reference</span><span>Source</span><span>Size</span><span>Added</span><span>Actions</span>
      </div>
      <div v-for="d in filtered" :key="d.id" class="doc-row">
        <span class="doc-name">{{ d.name }}</span>
        <span class="mono doc-dim">{{ d.source }}</span>
        <span class="mono doc-dim">{{ prettySize(d.size) }}</span>
        <span class="mono doc-dim">{{ prettyDate(d.created_at) }}</span>
        <span class="doc-actions">
          <button type="button" class="doc-link" @click="view(d)">View</button>
          <NuxtLink :to="{ path: '/annotate', query: { doc: d.id } }" class="doc-link">Annotate</NuxtLink>
          <button type="button" class="doc-link doc-link--danger" @click="removeOne(d.id)">Delete</button>
        </span>
      </div>
    </div>
    <p v-else-if="documents.length" class="doc-note mono">No documents match “{{ query }}”.</p>
    <p v-else class="doc-note mono">No documents yet — upload files above to build the corpus.</p>

    <!-- Viewer -->
    <div v-if="viewing" class="doc-viewer" @click.self="viewing = null">
      <div class="doc-viewer-panel">
        <div class="doc-viewer-head">
          <span class="doc-name">{{ viewing.name }}</span>
          <button type="button" class="doc-link" @click="viewing = null">Close ✕</button>
        </div>
        <div v-if="viewingHtml" class="doc-viewer-body" v-html="viewingHtml"></div>
        <pre v-else class="doc-viewer-body mono">{{ viewing.full_text }}</pre>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import DocumentManagement from '../components/DocumentManagement.vue';
import { useArtifact, listArtifacts, removeArtifact } from '@legalblocks/module-runtime';
import config from '../legalblocks.config.json';

// Config threaded from the builder. `documents_endpoint` (an artifact_endpoint bound
// to the "document" prefix) decides whether these documents live in
// browser storage or a database — this page's code is identical either way.
const P = config.params?.documentManagement ?? {};
const allowedTypes = P.allowed_types ?? 'txt,md,html,pdf,doc,docx';
const maxFileSizeMb = P.max_file_size_mb ?? '10';
const enablePdf = P.enable_pdf ?? 'on';
const enableDocx = P.enable_docx ?? 'on';
const screenCopy = config.screens?.find((s) => s.type === 'list')?.copy ?? {};

// One artifact per document under this prefix; the list screen enumerates them.
const PREFIX = 'document.';
const docKey = (id) => PREFIX + id;

const documents = ref([]);
const query = ref('');
const viewing = ref(null);
const viewingHtml = ref('');

const filtered = computed(() => {
  const q = query.value.trim().toLowerCase();
  return q ? documents.value.filter((d) => d.name.toLowerCase().includes(q)) : documents.value;
});

// --- persistence (all through the bus; adapter resolved from config) ----------
function envelope(record) {
  return {
    artifact_type: 'document',
    schema_version: '2.0.0',
    payload: record,
    lineage: { produced_by_package: 'document-management', produced_by_module: 'documentManagement' },
  };
}

async function persist(record) {
  await useArtifact(docKey(record.id)).write(envelope(record)).catch(() => {});
}

async function refresh() {
  let entries = [];
  try { entries = await listArtifacts(PREFIX); } catch { entries = []; }
  documents.value = entries
    .map((e) => e.artifact?.payload)
    .filter(Boolean)
    .sort((a, b) => (a.created_at ?? '').localeCompare(b.created_at ?? ''));
  await rebuildAggregate();
}

// Backward-compatibility shim: the annotate screen still reads a single
// `document-set`. We derive it from the per-document store and expose
// every field a consumer might want: `segments` (AnnotationUI), `text`/`full_text`
// (unused today; kept for a future text-extraction integration). Dropped once
// consumers read by prefix.
function segmentText(text) {
  const paras = (text ?? '').split(/\n\s*\n/).map((s) => s.trim()).filter(Boolean);
  if (paras.length > 1) return paras;
  const lines = (text ?? '').split(/\n/).map((s) => s.trim()).filter(Boolean);
  return lines.length ? lines : [(text ?? '').trim()].filter(Boolean);
}

async function rebuildAggregate() {
  const docs = documents.value.map((d) => ({
    id: d.id,
    ref: d.name,
    full_text: d.full_text,
    text: d.full_text,
    segments: segmentText(d.full_text),
  }));
  await useArtifact('document-set')
    .write({ artifact_type: 'document-set', schema_version: '1.0.0', payload: { documents: docs } })
    .catch(() => {});
}

// --- actions ------------------------------------------------------------------
async function onAdd(records) {
  for (const r of records) await persist(r);
  await refresh();
}
async function removeOne(id) {
  await removeArtifact(docKey(id)).catch(() => {});
  await refresh();
}
async function clearAll() {
  for (const d of documents.value) await removeArtifact(docKey(d.id)).catch(() => {});
  await refresh();
}

async function view(d) {
  viewing.value = d;
  viewingHtml.value = '';
  if (d.mime === 'text/html') {
    try {
      const mod = await import(/* @vite-ignore */ 'https://esm.sh/dompurify@3');
      const DOMPurify = mod.default ?? mod;
      viewingHtml.value = DOMPurify.sanitize(d.full_text);
    } catch {
      viewingHtml.value = ''; // fall back to raw text rendering
    }
  }
}

// --- presentation helpers -----------------------------------------------------
function prettySize(bytes) {
  if (!bytes) return '—';
  if (bytes < 1024) return bytes + ' B';
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB';
  return (bytes / 1024 / 1024).toFixed(1) + ' MB';
}
function prettyDate(iso) {
  if (!iso) return '—';
  try { return new Date(iso).toLocaleDateString(); } catch { return iso; }
}

// Seed a sample document on first run so the annotator has something to work on.
const SAMPLE = {
  id: 'doc-sample',
  name: 'Sample — EU Marine Strategy Framework Directive (extract).txt',
  source: 'imported',
  mime: 'text/plain',
  size: 0,
  hash: '',
  full_text: [
    'Member States shall establish a marine strategy for their marine waters.',
    'Each marine strategy shall contain a detailed assessment of the state of the environment.',
    'Undertakings must not discharge prohibited substances into the marine environment.',
    'Member States shall cooperate to ensure a coordinated implementation of this Directive.',
  ].join('\n\n'),
  created_at: new Date(0).toISOString(),
};

onMounted(async () => {
  await refresh();
  if (!documents.value.length) {
    SAMPLE.size = new Blob([SAMPLE.full_text]).size;
    await persist(SAMPLE);
    await refresh();
  }
});
</script>

<style scoped>
.page-documents { max-width: 800px; margin: 0 auto; padding: 36px 24px; }
.page-header { margin-bottom: 20px; }
.page-title { font-size: 28px; font-weight: 400; margin: 6px 0 0; color: var(--ink); }

.doc-toolbar { display: flex; justify-content: space-between; align-items: center; gap: 12px; margin-top: 20px; }
.doc-search { flex: 1; padding: 8px 12px; border: 1px solid var(--line); border-radius: var(--radius); font-size: 12px; background: var(--white); color: var(--ink); }
.doc-danger { font-size: 11px; padding: 7px 12px; border: 1px solid var(--line); border-radius: var(--radius); background: var(--white); color: var(--vermillion); cursor: pointer; }

.doc-table { background: var(--white); border: 1px solid var(--line); border-radius: var(--radius); overflow: hidden; margin-top: 12px; }
.doc-table-head, .doc-row {
  display: grid; grid-template-columns: 1fr 110px 80px 100px 190px;
  padding: 12px 16px; gap: 16px; align-items: center;
}
.doc-table-head { background: var(--cream); font-weight: 600; font-size: 11px; text-transform: uppercase; letter-spacing: 0.08em; color: var(--muted); border-bottom: 1px solid var(--line); }
.doc-row { border-bottom: 1px solid var(--line-2); font-size: 13px; }
.doc-row:last-child { border-bottom: none; }
.doc-name { font-weight: 600; color: var(--ink); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.doc-dim { color: var(--muted); font-size: 12px; }
.doc-actions { display: flex; gap: 12px; }
.doc-link { color: var(--vermillion); text-decoration: none; font-weight: 600; font-size: 12px; background: none; border: none; padding: 0; cursor: pointer; }
.doc-link--danger { color: #B23A1E; }
.doc-note { font-size: 12px; color: var(--muted); margin-top: 16px; }

.doc-viewer { position: fixed; inset: 0; background: rgba(20,17,14,0.45); display: flex; align-items: center; justify-content: center; padding: 24px; z-index: 50; }
.doc-viewer-panel { background: var(--white); border-radius: var(--radius); max-width: 720px; width: 100%; max-height: 80vh; display: flex; flex-direction: column; overflow: hidden; }
.doc-viewer-head { display: flex; justify-content: space-between; align-items: center; padding: 14px 18px; border-bottom: 1px solid var(--line); }
.doc-viewer-body { padding: 18px; overflow-y: auto; font-size: 13px; line-height: 1.6; color: var(--ink); white-space: pre-wrap; word-break: break-word; }
</style>
