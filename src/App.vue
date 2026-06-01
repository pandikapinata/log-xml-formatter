<script setup>
import { ref, watch, computed } from 'vue';
import xmlFormatter from 'xml-formatter';
import changelogTextRaw from '../CHANGELOG.md?raw';

// ─── State ────────────────────────────────────────────
const inputString = ref('');
const outputString = ref('');
const errorInfo = ref(null);
const isValid = ref(false);
const copyText = ref('Copy');
const isChangelogOpen = ref(false);
let copyTimeout = null;

// ─── Computed ─────────────────────────────────────────
const inputLineCount = computed(() => {
  if (!inputString.value) return 0;
  return inputString.value.split('\n').length;
});

const inputCharCount = computed(() => inputString.value.length);

const outputLineCount = computed(() => {
  if (!outputString.value) return 0;
  return outputString.value.split('\n').length;
});

const outputCharCount = computed(() => outputString.value.length);

const ableToCopy = computed(() => outputString.value.length > 0);
const ableToReset = computed(
  () => outputString.value.length > 0 || inputString.value.length > 0
);

// ─── Syntax-highlighted output lines ──────────────────
const highlightedLines = computed(() => {
  if (!outputString.value) return [];
  return outputString.value.split('\n').map((line) => highlightLine(line));
});

// ─── Debounced live formatting ────────────────────────
let debounceTimer = null;
watch(inputString, () => {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => formatXML(), 300);
});

// ─── Formatting pipeline ─────────────────────────────
const formatXML = () => {
  if (!inputString.value.trim()) {
    outputString.value = '';
    errorInfo.value = null;
    isValid.value = false;
    return;
  }

  // Step 1: Unescape log encoding
  const cleaned = unescapeLog(inputString.value);

  // Step 2: Extract XML from noisy log text
  const xml = extractXML(cleaned);

  // Step 3: Try to format
  let result;
  try {
    result = xmlFormatter(xml, { 
      collapseContent: true,
      indentation: '    ' 
    });
  } catch {
    result = xml;
  }

  outputString.value = result;

  // Step 4: Validate the displayed output
  try {
    const parser = new DOMParser();
    const doc = parser.parseFromString(result, 'text/xml');
    const parseError = doc.querySelector('parsererror');

    if (parseError) {
      errorInfo.value = parseErrorInfo(parseError);
      isValid.value = false;
    } else {
      errorInfo.value = null;
      isValid.value = true;
    }
  } catch {
    errorInfo.value = { message: 'Unable to validate XML' };
    isValid.value = false;
  }

  copyText.value = 'Copy';
};

// ─── Unescape common log escape sequences ─────────────
const unescapeLog = (input) => {
  let cleaned = input;

  // Decode HTML entities if input appears to be HTML-encoded
  // (only if no raw < tags exist — avoids breaking legitimate XML entities)
  const hasRawXml = cleaned.includes('<');
  if (!hasRawXml && cleaned.includes('&lt;')) {
    cleaned = cleaned
      .replace(/&lt;/g, '<')
      .replace(/&gt;/g, '>')
      .replace(/&amp;/g, '&')
      .replace(/&quot;/g, '"')
      .replace(/&apos;/g, "'");
  }

  // JSON-style unescape with placeholder to handle \\ correctly
  cleaned = cleaned
    .replace(/\\\\/g, '\x00')    // protect literal \\ with placeholder
    .replace(/\\\//g, '/')       // \/ → /
    .replace(/\\r\\n/g, '')      // \r\n → remove
    .replace(/\\r/g, '')         // \r → remove
    .replace(/\\n/g, '')         // \n → remove
    .replace(/\\t/g, '')         // \t → remove (FIX: was missing in v1)
    .replace(/\\"/g, '"')        // \" → "
    .replace(/\x00/g, '\\');     // restore placeholder → single \

  return cleaned;
};

// ─── Extract XML from surrounding log noise ────────────
const extractXML = (text) => {
  let cleaned = text.trim();

  // Remove surrounding quotes (JSON string values)
  if (
    (cleaned.startsWith('"') && cleaned.endsWith('"')) ||
    (cleaned.startsWith("'") && cleaned.endsWith("'"))
  ) {
    cleaned = cleaned.slice(1, -1);
  }

  // Find the first < and last > to extract XML portion
  const xmlStart = cleaned.indexOf('<');
  const xmlEnd = cleaned.lastIndexOf('>');
  if (xmlStart !== -1 && xmlEnd !== -1 && xmlEnd > xmlStart) {
    return cleaned.substring(xmlStart, xmlEnd + 1);
  }

  return cleaned;
};

// ─── Parse DOMParser error info ────────────────────────
const parseErrorInfo = (errorElement) => {
  const text = errorElement.textContent || '';

  const lineMatch = text.match(/line\s*(?:number\s*)?(\d+)/i);
  const colMatch = text.match(/column\s+(\d+)/i);

  let message = text
    .replace(/This page contains the following errors?:\s*/i, '')
    .replace(/Below is a rendering of the page up to the first error\.\s*/i, '')
    .replace(/Location:\s*http[^\s]+\s*/i, '')
    .replace(/\s+/g, ' ')
    .trim();

  if (message.length > 200) {
    message = message.substring(0, 200) + '…';
  }

  return {
    message,
    line: lineMatch ? parseInt(lineMatch[1]) : null,
    column: colMatch ? parseInt(colMatch[1]) : null,
  };
};

// ─── Syntax highlighting ──────────────────────────────

/** Escape HTML entities to prevent XSS in v-html output.
 *  TODO(security): All user content MUST pass through this before v-html rendering. */
const escapeHtml = (str) => {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
};

/** Find the real end of a tag, respecting quoted attribute values */
const findTagEnd = (str, startIdx) => {
  let inQuote = false;
  let quoteChar = '';
  for (let i = startIdx; i < str.length; i++) {
    if (inQuote) {
      if (str[i] === quoteChar) inQuote = false;
    } else {
      if (str[i] === '"' || str[i] === "'") {
        inQuote = true;
        quoteChar = str[i];
      } else if (str[i] === '>') {
        return i;
      }
    }
  }
  return -1;
};

/** Highlight a single line of XML */
const highlightLine = (line) => {
  if (!line) return '';

  let result = '';
  let i = 0;

  // Handle leading spaces for indent guides
  const leadingSpaceMatch = line.match(/^(\s+)/);
  if (leadingSpaceMatch) {
    const spaces = leadingSpaceMatch[1];
    let guides = '';
    
    // Group spaces by 4 (our explicit indentation size)
    const groupSize = (spaces.length > 0 && spaces.length % 4 !== 0 && spaces.length % 2 === 0) ? 2 : 4;
    for (let s = 0; s < spaces.length; s += groupSize) {
      const chunk = spaces.substring(s, s + groupSize);
      guides += `<span class="indent-guide">${chunk}</span>`;
    }
    result += guides;
    i = spaces.length;
  }

  while (i < line.length) {
    if (line.startsWith('<!--', i)) {
      // XML comment
      const end = line.indexOf('-->', i);
      const endIdx = end !== -1 ? end + 3 : line.length;
      result += `<span class="hl-comment">${escapeHtml(line.substring(i, endIdx))}</span>`;
      i = endIdx;
    } else if (line.startsWith('<?', i)) {
      // Processing instruction
      const end = line.indexOf('?>', i);
      const endIdx = end !== -1 ? end + 2 : line.length;
      result += `<span class="hl-pi">${escapeHtml(line.substring(i, endIdx))}</span>`;
      i = endIdx;
    } else if (line.startsWith('<![CDATA[', i)) {
      // CDATA section
      const end = line.indexOf(']]>', i);
      const endIdx = end !== -1 ? end + 3 : line.length;
      result += `<span class="hl-cdata">${escapeHtml(line.substring(i, endIdx))}</span>`;
      i = endIdx;
    } else if (line[i] === '<') {
      // Regular tag
      const end = findTagEnd(line, i + 1);
      const endIdx = end !== -1 ? end + 1 : line.length;
      const tagStr = line.substring(i, endIdx);
      result += highlightTag(tagStr);
      i = endIdx;
    } else {
      // Text content or whitespace
      const nextTag = line.indexOf('<', i);
      const endIdx = nextTag !== -1 ? nextTag : line.length;
      const text = line.substring(i, endIdx);
      if (text.trim()) {
        result += `<span class="hl-text">${escapeHtml(text)}</span>`;
      } else {
        result += escapeHtml(text);
      }
      i = endIdx;
    }
  }

  return result;
};

/** Highlight a complete tag string like <tag attr="val"> or </tag> */
const highlightTag = (tag) => {
  // Closing tag: </name>
  const closingMatch = tag.match(/^<\/([\w:.-]+)\s*>$/);
  if (closingMatch) {
    return (
      `<span class="hl-bracket">&lt;/</span>` +
      `<span class="hl-tag">${escapeHtml(closingMatch[1])}</span>` +
      `<span class="hl-bracket">&gt;</span>`
    );
  }

  // Opening or self-closing tag: <name ...> or <name ... />
  const openMatch = tag.match(/^<([\w:.-]+)([\s\S]*?)(\/?)>$/);
  if (openMatch) {
    const [, name, attrs, selfClose] = openMatch;
    let result = `<span class="hl-bracket">&lt;</span>`;
    result += `<span class="hl-tag">${escapeHtml(name)}</span>`;

    if (attrs.trim()) {
      result += highlightAttrs(attrs);
    }

    if (selfClose) {
      result += `<span class="hl-bracket">/&gt;</span>`;
    } else {
      result += `<span class="hl-bracket">&gt;</span>`;
    }

    return result;
  }

  // Fallback
  return `<span class="hl-bracket">${escapeHtml(tag)}</span>`;
};

/** Highlight attributes within a tag */
const highlightAttrs = (attrs) => {
  let result = '';
  const regex = /\s+([\w:.-]+)\s*=\s*"([^"]*)"/g;
  let lastIndex = 0;
  let match;

  while ((match = regex.exec(attrs)) !== null) {
    if (match.index > lastIndex) {
      result += escapeHtml(attrs.substring(lastIndex, match.index));
    }
    result += ` <span class="hl-attr">${escapeHtml(match[1])}</span>`;
    result += `<span class="hl-bracket">=</span>`;
    result += `<span class="hl-value">&quot;${escapeHtml(match[2])}&quot;</span>`;
    lastIndex = match.index + match[0].length;
  }

  if (lastIndex < attrs.length) {
    result += escapeHtml(attrs.substring(lastIndex));
  }

  return result;
};

// ─── Actions ──────────────────────────────────────────
const copyToClipboard = async () => {
  try {
    await navigator.clipboard.writeText(outputString.value);
    copyText.value = '✓ Copied!';
  } catch {
    // Fallback for non-HTTPS contexts
    const textarea = document.createElement('textarea');
    textarea.value = outputString.value;
    textarea.style.position = 'fixed';
    textarea.style.opacity = '0';
    document.body.appendChild(textarea);
    textarea.select();
    document.execCommand('copy');
    document.body.removeChild(textarea);
    copyText.value = '✓ Copied!';
  }

  clearTimeout(copyTimeout);
  copyTimeout = setTimeout(() => {
    copyText.value = 'Copy';
  }, 2000);
};

const reset = () => {
  inputString.value = '';
  outputString.value = '';
  errorInfo.value = null;
  isValid.value = false;
  copyText.value = 'Copy';
};

const handleOutputKeydown = (e) => {
  if (e.ctrlKey && e.key === 'a') {
    e.preventDefault();
    const outputEl = document.getElementById('xml-output');
    if (outputEl) {
      const range = document.createRange();
      range.selectNodeContents(outputEl);
      const sel = window.getSelection();
      sel.removeAllRanges();
      sel.addRange(range);
    }
  }
};
</script>

<template>
  <header class="header">
    <h1 class="header-title">
      Log XML Formatter
    </h1>
    <div class="header-actions">
      <button class="btn-text" @click="isChangelogOpen = true">Changelog</button>
      <span class="header-version">v2.0</span>
    </div>
  </header>

  <!-- ─── Changelog Modal ─── -->
  <div v-if="isChangelogOpen" class="modal-backdrop" @click="isChangelogOpen = false">
    <div class="modal-content" @click.stop>
      <div class="modal-header">
        <h2>Changelog</h2>
        <button class="btn-close" @click="isChangelogOpen = false">×</button>
      </div>
      <pre class="changelog-text">{{ changelogTextRaw }}</pre>
    </div>
  </div>

  <main class="editor-container">
    <!-- ─── Input Panel ─── -->
    <section class="panel">
      <div class="panel-header">
        <span class="panel-label">input</span>
        <span class="panel-stats" v-if="inputString">
          {{ inputLineCount }} {{ inputLineCount === 1 ? 'line' : 'lines' }} · {{ inputCharCount }} {{ inputCharCount === 1 ? 'char' : 'chars' }}
        </span>
      </div>
      <textarea
        id="xml-input"
        v-model="inputString"
        class="editor-input"
        placeholder="Paste your XML log here…"
        spellcheck="false"
        autocomplete="off"
        autocorrect="off"
        autocapitalize="off"
      ></textarea>
    </section>

    <!-- ─── Output Panel ─── -->
    <section class="panel">
      <div class="panel-header">
        <div class="panel-header-left">
          <span class="panel-label">output</span>
          <span v-if="isValid" class="badge badge-valid">✓ valid</span>
          <span v-if="errorInfo" class="badge badge-error">✕ error</span>
        </div>
        <span class="panel-stats" v-if="outputString">
          {{ outputLineCount }} {{ outputLineCount === 1 ? 'line' : 'lines' }} · {{ outputCharCount }} {{ outputCharCount === 1 ? 'char' : 'chars' }}
        </span>
      </div>
      <div id="xml-output" class="editor-output" tabindex="0" @keydown="handleOutputKeydown">
        <div v-if="highlightedLines.length" class="code-lines">
          <div
            v-for="(line, idx) in highlightedLines"
            :key="idx"
            class="code-line"
            :class="{ 'error-line': errorInfo?.line === idx + 1 }"
          >
            <span class="line-num">{{ idx + 1 }}</span>
            <!-- TODO(security): highlightLine() escapes all user content via escapeHtml() before v-html rendering -->
            <code class="line-code" v-html="line || '&nbsp;'"></code>
          </div>
        </div>
        <div v-else class="output-placeholder">
          <span>formatted xml will appear here</span>
        </div>
      </div>
    </section>
  </main>

  <!-- ─── Error Banner ─── -->
  <div v-if="errorInfo" class="error-banner">
    <span class="error-icon">!</span>
    <span class="error-message">{{ errorInfo.message }}</span>
    <span v-if="errorInfo.line" class="error-location">
      ln {{ errorInfo.line }}<template v-if="errorInfo.column">, col {{ errorInfo.column }}</template>
    </span>
  </div>

  <!-- ─── Toolbar ─── -->
  <div class="toolbar">
    <button
      id="btn-copy"
      @click="copyToClipboard"
      class="btn btn-primary"
      :class="{ 'btn-copied': copyText !== 'Copy' }"
      :disabled="!ableToCopy"
    >
      {{ copyText }}
    </button>
    <button
      id="btn-reset"
      @click="reset"
      class="btn btn-secondary"
      :disabled="!ableToReset"
    >
      Reset
    </button>
  </div>

  <footer class="footer">
    <p>
      made with ❤️ by
      <a href="https://id.linkedin.com/in/pandikapinata" target="_blank" rel="noopener noreferrer">
        pandika pinata
      </a>
      and AI 🤖
    </p>
  </footer>
</template>


