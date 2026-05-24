<template>
  <div class="text-block">
    <div :class="markdownClasses" v-html="renderedMarkdown" @click="handleLinkClick"></div>
  </div>
</template>

<script setup lang="ts">
import { computed, inject } from 'vue';
import type { TextBlock as TextBlockType } from '../../../models/ContentBlock';
import type { ToolContext } from '../../../types/tool';
import { marked } from 'marked';
import { RuntimeKey } from '../../../composables/runtimeContext';
// import DOMPurify from 'dompurify'; // TODO:

interface Props {
  block: TextBlockType;
  context?: ToolContext;
}

const props = defineProps<Props>();

const runtime = inject(RuntimeKey);

const copyIconSvg = `<svg xmlns="http://www.w3.org/2000/svg" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><rect x="9" y="9" width="13" height="13" rx="2" ry="2"></rect><path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"></path></svg>`;
const checkIconSvg = `<svg xmlns="http://www.w3.org/2000/svg" width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><polyline points="20 6 9 17 4 12"></polyline></svg>`;

function escapeHtml(text: string): string {
  return text
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

function handleLinkClick(event: MouseEvent) {
  const copyBtn = (event.target as HTMLElement).closest('.copy-btn');
  if (copyBtn) {
    event.stopPropagation();
    const wrapper = copyBtn.closest('.copy-wrapper');
    const content = wrapper?.querySelector('pre code, blockquote');
    if (content) {
      navigator.clipboard.writeText(content.textContent ?? '').then(() => {
        const btn = copyBtn as HTMLElement;
        const original = btn.innerHTML;
        btn.innerHTML = checkIconSvg;
        setTimeout(() => { btn.innerHTML = original; }, 1500);
      });
    }
    return;
  }

  const target = (event.target as HTMLElement).closest('a');
  if (!target) return;

  const href = target.getAttribute('data-href') ?? target.getAttribute('href');
  if (!href) return;

  event.preventDefault();

  // External URLs
  if (/^https?:\/\//i.test(href)) {
    runtime?.appContext.openURL(href);
    return;
  }

  // Relative file paths, optionally with #L42 or #L42-L51 fragment
  const match = href.match(/^([^#]+)(?:#L(\d+)(?:-L(\d+))?)?$/);
  if (match) {
    const filePath = match[1];
    const startLine = match[2] ? parseInt(match[2], 10) : undefined;
    const endLine = match[3] ? parseInt(match[3], 10) : undefined;
    const location = startLine !== undefined ? { startLine, endLine } : undefined;
    runtime?.appContext.fileOpener.open(filePath, location);
  }
}

// Markdown
const markdownClasses = computed(() => {
  const classes = ['markdown-content'];
  if (props.block.isSlashCommand) {
    classes.push('slash-command-text');
  }
  return classes;
});

// marked
marked.setOptions({
  gfm: true,
  breaks: true,
});

const renderer = new marked.Renderer();

renderer.link = ({ href, title, text }) => {
  const titleAttr = title ? ` title="${title}"` : '';
  // Use data-href so VSCode's webview doesn't intercept external link navigation
  return `<a data-href="${href}"${titleAttr}>${text}</a>`;
};

renderer.code = ({ text, lang }) => {
  const langClass = lang ? ` class="language-${lang}"` : '';
  const singleLine = !text.includes('\n');
  const wrapperClass = singleLine ? 'copy-wrapper single-line' : 'copy-wrapper';
  return `<div class="${wrapperClass}"><pre><code${langClass}>${escapeHtml(text)}</code></pre><button class="copy-btn" aria-label="Copy">${copyIconSvg}</button></div>`;
};

renderer.blockquote = ({ text }) => {
  const singleLine = !text.includes('<br') && (text.match(/<p>/g) ?? []).length <= 1;
  const wrapperClass = singleLine ? 'copy-wrapper single-line' : 'copy-wrapper';
  return `<div class="${wrapperClass}"><blockquote>${text}</blockquote><button class="copy-btn" aria-label="Copy">${copyIconSvg}</button></div>`;
};

// Markdown
const renderedMarkdown = computed(() => {
  const rawHtml = marked.parse(props.block.text, { renderer }) as string;
  // TODO: DOMPurify.sanitize(rawHtml)
  return rawHtml;
});
</script>

<style scoped>
.text-block {
  margin: 0;
  padding: 0;
}

.markdown-content {
  font-size: 13px;
  line-height: 1.6;
  color: var(--vscode-editor-foreground);
  word-wrap: break-word;
  user-select: text;
  padding: 8px 5px 2px 5px;
}

.slash-command-text {
  color: var(--vscode-textLink-foreground);
  font-weight: 600;
}

/* Markdown - Claudex */
.markdown-content :deep(p) {
  margin: 8px 0;
  line-height: 1.6;
}

.markdown-content :deep(p:first-child) {
  margin-top: 0;
}

.markdown-content :deep(code) {
  font-family: var(--vscode-editor-font-family, 'Hack Nerd Font Mono', 'SF Mono', Consolas, 'Courier New', monospace);
  word-break: break-all;
  cursor: default;
}

.markdown-content :deep(pre) {
  background-color: color-mix(in srgb, var(--vscode-editor-background) 50%, transparent);
  border: 1px solid var(--vscode-panel-border);
  border-radius: 4px;
  padding: 12px;
  margin: 8px 0;
  overflow-x: auto;
}

.markdown-content :deep(pre code) {
  background: none;
  border: none;
  padding: 0;
}

.markdown-content :deep(:not(pre) > code) {
  background-color: color-mix(in srgb, var(--vscode-editor-background) 50%, transparent);
  border: 1px solid var(--vscode-panel-border);
  border-radius: 3px;
  padding: 2px 4px;
  font-size: 1em;
}

.markdown-content :deep(a) {
  color: var(--vscode-textLink-foreground);
  text-decoration: none;
  cursor: pointer;
}

.markdown-content :deep(a:hover) {
  color: var(--vscode-textLink-activeForeground);
  text-decoration: underline;
}

.markdown-content :deep(ul),
.markdown-content :deep(ol) {
  margin: 0px 0px 0px 16px;
  padding: 0px;
}

.markdown-content :deep(li) {
  padding-top: 2px;
  padding-bottom: 2px;
  list-style-type: disc;
}

.markdown-content :deep(blockquote) {
  border-left: 4px solid var(--vscode-textBlockQuote-border);
  background-color: var(--vscode-textBlockQuote-background);
  margin: 8px 0;
  padding: 8px 16px;
}

.markdown-content :deep(h1),
.markdown-content :deep(h2),
.markdown-content :deep(h3),
.markdown-content :deep(h4),
.markdown-content :deep(h5),
.markdown-content :deep(h6) {
  color: var(--vscode-foreground);
  font-weight: 600;
  margin: 16px 0 8px 0;
  line-height: 1.3;
}

.markdown-content :deep(h1) {
  font-size: 18px;
}

.markdown-content :deep(h2) {
  font-size: 16px;
}

.markdown-content :deep(h3) {
  font-size: 14px;
}

.markdown-content :deep(table) {
  border-collapse: collapse;
  margin: 16px 0;
  width: 100%;
}

.markdown-content :deep(th),
.markdown-content :deep(td) {
  border: 1px solid var(--vscode-panel-border);
  padding: 8px 12px;
  text-align: left;
}

.markdown-content :deep(th) {
  background-color: color-mix(in srgb, var(--vscode-editor-background) 30%, transparent);
  font-weight: 600;
}

.markdown-content :deep(.copy-wrapper) {
  position: relative;
}

.markdown-content :deep(.copy-wrapper pre),
.markdown-content :deep(.copy-wrapper blockquote) {
  padding-right: 36px;
}

.markdown-content :deep(.copy-btn) {
  position: absolute;
  top: 8px;
  right: 8px;
  background: color-mix(in srgb, var(--vscode-editor-background) 85%, transparent);
  border: 1px solid var(--vscode-panel-border);
  border-radius: 4px;
  padding: 3px 5px;
  cursor: pointer;
  color: var(--vscode-editor-foreground);
  opacity: 0.35;
  transition: opacity 0.15s;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
}

.markdown-content :deep(.copy-btn:hover) {
  opacity: 1;
}

.markdown-content :deep(.copy-wrapper.single-line .copy-btn) {
  top: 50%;
  transform: translateY(-50%);
}
</style>
