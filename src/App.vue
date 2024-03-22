<script setup>
import { computed, ref } from 'vue';
import xmlFormatter from 'xml-formatter';

const inputString = ref('');
const outputString = ref('');
const copy = ref('Copy to Clipboard');

const modifyXML = () => {
  if (!inputString.value) {
    return;
  }

  const modified = inputString.value
    .replace(/\\n/g, '')
    .replace(/\\/g, '')
    .replace(/\r/g, '')
    .replace(/\\"/g, '"');
  outputString.value = beautifyXML(modified);
  copy.value = 'Copy to Clipboard';
};

const beautifyXML = (xmlString) => {
  try {
    return xmlFormatter(xmlString, {
      collapseContent: true,
    });
  } catch (error) {
    console.error('Error formatting XML:', error);
    return xmlString;
  }
};

const copyToClipboard = () => {
  navigator.clipboard.writeText(outputString.value);
  copy.value = 'Success! Copied to Clipboard';
};

const reset = () => {
  inputString.value = '';
  outputString.value = '';
  copy.value = 'Copy to Clipboard';
};

const ableToCopy = computed(() => outputString.value.length > 0);
const ableToReset = computed(() => outputString.value.length > 0 || inputString.value.length > 0);
</script>

<template>
  <main class="container">
    <h1>Formatter</h1>
    <textarea
      v-model="inputString"
      placeholder="Enter your string"
      rows="15"
    ></textarea>
    <button @click="modifyXML" class="btn">Convert</button>
    <textarea v-model="outputString" rows="15" readonly></textarea>
    <button @click="copyToClipboard" class="btn-copy" :disabled="!ableToCopy">{{ copy }}</button>
    <button @click="reset" class="btn-reset" :disabled="!ableToReset">Reset</button>
  </main>
  <footer>
    <p>Made with ❤️ by <span><a href="https://id.linkedin.com/in/pandikapinata">Pandika Pinata</a></span></p>
  </footer>
</template>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.btn {
  padding-bottom: 0.375rem;
  padding-top: 0.375rem;
  background-color: var(--primary);
}
.btn:hover {
  background-color: var(--primary-hover);
}

.btn-copy {
  background-color: var(--secondary);
}
.btn-copy:hover:enabled {
  background-color: var(--secondary-hover);
}
.btn-copy:disabled,
.btn-copy[disabled]{
  background-color: #292929;
  color: #525252;
  cursor: not-allowed;
}

.btn-reset {
  color: #242424;
  background-color: #fff;
}
.btn-reset:hover:enabled {
  background-color: #f3f3f3;
}
.btn-reset:disabled,
.btn-reset[disabled]{
  background-color: #dcdcdc;
  color: #525252;
  cursor: not-allowed;
}
</style>
