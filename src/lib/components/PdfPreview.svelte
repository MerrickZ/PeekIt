<script lang="ts">
  import { onMount, onDestroy, tick } from 'svelte';
  import { convertFileSrc } from '@tauri-apps/api/core';
  import type { FilePreviewInfo, AppLanguage } from '../types';
  import { t } from '../i18n';
  import * as pdfjsLib from 'pdfjs-dist';
  import pdfjsWorker from 'pdfjs-dist/build/pdf.worker.min.mjs?url';

  if (typeof window !== 'undefined' && pdfjsLib.GlobalWorkerOptions) {
    pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker;
  }

  export let file: FilePreviewInfo;
  export let lang: AppLanguage = 'ru';

  interface PageEntry {
    num: number;
    width: number;
    height: number;
    rendered: boolean;
    rendering: boolean;
  }

  let scrollEl: HTMLElement;
  let totalPages = 1;
  let currentPage = 1;
  let scale = 1.3;
  let pdfDoc: any = null;
  let loading = true;
  let errorMsg = '';

  let pages: PageEntry[] = [];
  let canvasEls: { [num: number]: HTMLCanvasElement } = {};
  let pendingRaf = 0;

  // Pre-render pages within this distance (px) around the viewport
  const RENDER_MARGIN = 600;
  // Keep rendered canvases at most this many pages away from the current page
  const KEEP_ALIVE_PAGES = 12;

  async function loadPdf() {
    try {
      loading = true;
      errorMsg = '';
      const fileUrl = convertFileSrc(file.path);
      const response = await fetch(fileUrl);
      const buffer = await response.arrayBuffer();

      const loadingTask = pdfjsLib.getDocument({
        data: new Uint8Array(buffer),
        cMapUrl: 'https://cdn.jsdelivr.net/npm/pdfjs-dist@4.10.38/cmaps/',
        cMapPacked: true,
      });

      pdfDoc = await loadingTask.promise;
      totalPages = pdfDoc.numPages;

      // Estimate page size from the first page; each placeholder is corrected
      // to the real viewport size when the page is lazily rendered.
      const first = await pdfDoc.getPage(1);
      const vp = first.getViewport({ scale });
      pages = Array.from({ length: totalPages }, (_, i) => ({
        num: i + 1,
        width: vp.width,
        height: vp.height,
        rendered: false,
        rendering: false,
      }));
      loading = false;
      await tick();
      renderVisiblePages();
    } catch (err: any) {
      console.error('PDF load error:', err);
      errorMsg = err?.message || 'Failed to load PDF';
      loading = false;
    }
  }

  function isNearViewport(el: HTMLElement): boolean {
    const view = scrollEl.getBoundingClientRect();
    const rect = el.getBoundingClientRect();
    return rect.bottom > view.top - RENDER_MARGIN && rect.top < view.bottom + RENDER_MARGIN;
  }

  function renderVisiblePages() {
    if (!pdfDoc || !scrollEl) return;
    const slots = scrollEl.querySelectorAll<HTMLElement>('.page-slot');
    slots.forEach((slot) => {
      const num = Number(slot.dataset.page);
      if (num && isNearViewport(slot)) {
        renderPage(num);
      }
    });
  }

  async function renderPage(num: number) {
    if (!pdfDoc) return;
    const entry = pages[num - 1];
    if (!entry || entry.rendered || entry.rendering) return;
    entry.rendering = true;
    pages[num - 1] = { ...entry };

    try {
      const page = await pdfDoc.getPage(num);
      const viewport = page.getViewport({ scale });

      // Correct placeholder size to the real page dimensions
      pages[num - 1] = { ...pages[num - 1], width: viewport.width, height: viewport.height };
      await tick();

      const canvas = canvasEls[num];
      if (!canvas) return;
      const ctx = canvas.getContext('2d');
      if (!ctx) return;

      canvas.width = viewport.width;
      canvas.height = viewport.height;
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(0, 0, canvas.width, canvas.height);

      await page.render({
        canvasContext: ctx,
        viewport: viewport,
      }).promise;

      pages[num - 1] = { ...pages[num - 1], rendered: true, rendering: false };
    } catch (err) {
      console.error(`PDF render error (page ${num}):`, err);
      pages[num - 1] = { ...pages[num - 1], rendering: false };
    }
  }

  // Free canvases of pages far from the current one to bound memory usage
  function releaseFarPages() {
    if (!scrollEl) return;
    pages.forEach((p) => {
      if (!p.rendered) return;
      if (Math.abs(p.num - currentPage) <= KEEP_ALIVE_PAGES) return;
      const canvas = canvasEls[p.num];
      const slot = canvas?.parentElement;
      if (slot && isNearViewport(slot)) return; // still visible, keep it
      if (canvas) {
        canvas.width = 0;
        canvas.height = 0;
      }
      pages[p.num - 1] = { ...p, rendered: false };
    });
  }

  function updateCurrentPage() {
    if (!scrollEl || totalPages === 0) return;
    const view = scrollEl.getBoundingClientRect();
    const center = view.top + view.height / 2;
    let best = currentPage;
    let bestDist = Infinity;
    const slots = scrollEl.querySelectorAll<HTMLElement>('.page-slot');
    slots.forEach((slot) => {
      const rect = slot.getBoundingClientRect();
      const c = rect.top + rect.height / 2;
      const d = Math.abs(c - center);
      if (d < bestDist) {
        bestDist = d;
        best = Number(slot.dataset.page) || currentPage;
      }
    });
    currentPage = best;
  }

  function handleScroll() {
    if (pendingRaf) return;
    pendingRaf = requestAnimationFrame(() => {
      pendingRaf = 0;
      updateCurrentPage();
      renderVisiblePages();
      releaseFarPages();
    });
  }

  function scrollToPage(num: number) {
    const target = Math.min(totalPages, Math.max(1, num));
    const slot = scrollEl?.querySelector<HTMLElement>(`.page-slot[data-page="${target}"]`);
    if (slot) {
      slot.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  }

  function prevPage() {
    scrollToPage(currentPage - 1);
  }

  function nextPage() {
    scrollToPage(currentPage + 1);
  }

  function zoomIn() {
    applyZoom(Math.min(3.0, Math.round((scale + 0.2) * 10) / 10));
  }

  function zoomOut() {
    applyZoom(Math.max(0.6, Math.round((scale - 0.2) * 10) / 10));
  }

  async function applyZoom(next: number) {
    if (!pdfDoc || next === scale) return;
    const ratio = next / scale;
    scale = next;
    // Invalidate all rendered pages; placeholders scale linearly
    pages = pages.map((p) => ({
      ...p,
      width: p.width * ratio,
      height: p.height * ratio,
      rendered: false,
    }));
    await tick();
    renderVisiblePages();
  }

  onMount(() => {
    loadPdf();
  });

  onDestroy(() => {
    if (pendingRaf) cancelAnimationFrame(pendingRaf);
    try {
      pdfDoc?.destroy();
    } catch {
      // ignore
    }
  });
</script>

<div class="pdf-container">
  <div class="pdf-toolbar">
    <div class="page-nav">
      <button class="btn-nav" on:click={prevPage} disabled={currentPage <= 1} title={t('prev_file', lang)}>
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
          <polyline points="15 18 9 12 15 6" />
        </svg>
      </button>
      <span class="page-indicator">
        {currentPage} {t('page_of', lang)} {totalPages}
      </span>
      <button class="btn-nav" on:click={nextPage} disabled={currentPage >= totalPages} title={t('next_file', lang)}>
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
          <polyline points="9 18 15 12 9 6" />
        </svg>
      </button>
    </div>

    <div class="zoom-controls">
      <button class="btn-nav" on:click={zoomOut} title="Zoom Out" aria-label="Zoom Out">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="5" y1="12" x2="19" y2="12" />
        </svg>
      </button>
      <span class="zoom-label">{Math.round(scale * 100)}%</span>
      <button class="btn-nav" on:click={zoomIn} title="Zoom In" aria-label="Zoom In">
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
          <line x1="12" y1="5" x2="12" y2="19" />
          <line x1="5" y1="12" x2="19" y2="12" />
        </svg>
      </button>
    </div>
  </div>

  <div class="pdf-canvas-wrapper" bind:this={scrollEl} on:scroll={handleScroll}>
    {#if loading}
      <div class="loading-box">
        <div class="spinner"></div>
      </div>
    {:else if errorMsg}
      <div class="error-box">{errorMsg}</div>
    {:else}
      {#each pages as p (p.num)}
        <div
          class="page-slot"
          data-page={p.num}
          style="width:{Math.round(p.width)}px;height:{Math.round(p.height)}px"
        >
          <canvas bind:this={canvasEls[p.num]} class="pdf-canvas" class:hidden={!p.rendered}></canvas>
          {#if !p.rendered}
            <div class="page-placeholder">
              <span class="ph-num">{p.num}</span>
            </div>
          {/if}
        </div>
      {/each}
    {/if}
  </div>
</div>

<style>
  .pdf-container {
    display: flex;
    flex-direction: column;
    height: 100%;
    width: 100%;
    background: var(--bg-card);
  }

  .pdf-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 16px;
    background: var(--bg-subtle);
    border-bottom: 1px solid var(--border-color);
  }

  .page-nav, .zoom-controls {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .btn-nav {
    background: transparent;
    border: 1px solid var(--border-color);
    color: var(--text-main);
    border-radius: 6px;
    padding: 4px 8px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.15s ease;
  }

  .btn-nav:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  .btn-nav:hover:not(:disabled) {
    background: var(--bg-hover);
  }

  .page-indicator, .zoom-label {
    font-size: 12px;
    font-family: 'JetBrains Mono', monospace;
    color: var(--text-muted);
    user-select: none;
  }

  .pdf-canvas-wrapper {
    flex: 1;
    overflow: auto;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    padding: 24px;
    background: rgba(0, 0, 0, 0.2);
  }

  .page-slot {
    position: relative;
    flex-shrink: 0;
    background: #ffffff;
    border-radius: 4px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
    overflow: hidden;
  }

  .pdf-canvas {
    display: block;
    max-width: 100%;
  }

  .pdf-canvas.hidden {
    display: none;
  }

  .page-placeholder {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background:
      linear-gradient(45deg, rgba(0, 0, 0, 0.03) 25%, transparent 25%, transparent 50%, rgba(0, 0, 0, 0.03) 50%, rgba(0, 0, 0, 0.03) 75%, transparent 75%),
      #f8fafc;
    background-size: 24px 24px;
    border: 1px dashed rgba(100, 116, 139, 0.35);
    border-radius: 4px;
  }

  .ph-num {
    font-size: 14px;
    font-family: 'JetBrains Mono', monospace;
    color: #94a3b8;
    background: rgba(255, 255, 255, 0.85);
    border: 1px solid rgba(100, 116, 139, 0.3);
    border-radius: 6px;
    padding: 4px 12px;
    user-select: none;
  }

  .loading-box {
    display: flex;
    align-items: center;
    justify-content: center;
    height: 200px;
  }

  .spinner {
    width: 32px;
    height: 32px;
    border: 3px solid rgba(255, 255, 255, 0.1);
    border-top-color: var(--accent);
    border-radius: 50%;
    animation: spin 0.8s linear infinite;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .error-box {
    color: #ef4444;
    font-size: 14px;
    padding: 20px;
  }
</style>
