<template>
  <UiModal
    :title="$t('batchMode')"
    :subtitle="!isProcessing && !showResults ? $t('textBatchModeDescription') : ''"
    icon="layers"
    size="lg"
    :close-on-backdrop="!isProcessing"
    @close="close"
  >
    <!-- Options (visible before processing and results) -->
    <div v-if="!isProcessing && !showResults" class="batch">
      <div class="batch__panel">
        <details class="howto" open>
          <summary><UiIcon name="info" /> {{ $t('textBatchHowToTitle') }}</summary>
          <ol>
            <li>{{ $t('textBatchStep1') }}</li>
            <li>{{ $t('textBatchStep2') }}</li>
            <li>{{ $t('textBatchStep3') }}</li>
          </ol>
        </details>

        <label class="field-stack">
          <span class="field-label">{{ $t('textBatchTextareaLabel') }}</span>
          <div class="batch__textarea-wrap">
            <textarea
              v-model="simpleTextInput"
              class="textarea textarea--mono batch__textarea"
              :placeholder="$t('textBatchTextareaPlaceholder')"
              rows="10"
              spellcheck="false"
            ></textarea>
            <span class="batch__count" :class="{ 'is-active': simpleTextLines.length > 0 }">
              {{ $t('textBatchTextareaHelp', { count: simpleTextLines.length }) }}
            </span>
          </div>
        </label>

        <!-- Warning for large batches -->
        <div v-if="simpleTextLines.length > 50" class="notice notice--warning">
          <UiIcon name="alert" />
          <span>{{ $t('textBatchLargeWarning', { count: simpleTextLines.length }) }}</span>
        </div>
      </div>
    </div>

    <!-- Processing Progress -->
    <div v-if="isProcessing" class="batch-progress" role="status" aria-live="polite">
      <div class="batch-progress__head">
        <span class="batch-progress__title">
          <UiIcon name="loader" class="spin" />
          {{ $t('textBatchProcessing') }}
        </span>
        <span class="batch-progress__percent">{{ progressPercent }}%</span>
      </div>
      <div class="progress">
        <div class="progress__bar" :style="{ width: progressPercent + '%' }"></div>
      </div>
      <p class="field-hint">{{ $t('batchProgress', { current: processedCount, total: totalCount }) }}</p>
      <p v-if="currentItemLabel" class="batch-progress__current">
        {{ $t('batchCurrentItem') }}: <code>{{ currentItemLabel }}</code>
      </p>
    </div>

    <!-- Results summary and countdown -->
    <div v-if="showResults" class="batch-results">
      <div class="batch-results__content">
        <div v-if="successCount > 0" class="notice notice--success">
          <UiIcon name="circle-check" />
          <span>{{ $t('textBatchSuccessCount', { count: successCount }) }}</span>
        </div>
        <div v-if="errorResults.length > 0" class="notice notice--danger">
          <UiIcon name="circle-x" />
          <div>
            <strong>{{ $t('textBatchErrorCount', { count: errorResults.length }) }}</strong>
            <ul class="batch-results__errors">
              <li v-for="(err, index) in errorResults.slice(0, 10)" :key="index">
                {{ $t('batchRowError', { row: err.row, error: err.error }) }}
              </li>
              <li v-if="errorResults.length > 10">
                {{ $t('batchMoreErrors', { count: errorResults.length - 10 }) }}
              </li>
            </ul>
          </div>
        </div>

        <div v-if="successCount > 0" class="batch-results__download">
          <div class="progress" :class="{ 'progress--indeterminate': countdownSeconds !== 0 }">
            <div class="progress__bar" :style="countdownSeconds === 0 ? { width: '100%' } : null"></div>
          </div>
          <p class="batch-results__countdown">
            <span v-if="countdownSeconds > 0">{{ $t('batchDownloadCountdown', { seconds: countdownSeconds }) }}</span>
            <span v-if="countdownSeconds === 0">{{ $t('batchDownloadStarting') }}</span>
          </p>
        </div>
      </div>
    </div>

    <template #footer>
      <template v-if="!isProcessing && !showResults">
        <button type="button" class="btn" @click="close">{{ $t('cancel') }}</button>
        <button
          type="button"
          class="btn btn--primary"
          :disabled="!canGenerate"
          @click="startBatchGeneration"
        >
          <UiIcon name="play" />
          <span>{{ $t('batchGenerate') }} ({{ simpleTextLines.length }})</span>
        </button>
      </template>
      <template v-if="isProcessing">
        <button type="button" class="btn btn--danger" @click="abortGeneration">
          <UiIcon name="square" />
          <span>{{ $t('batchAbort') }}</span>
        </button>
      </template>
      <template v-if="showResults">
        <button type="button" class="btn" @click="reset">{{ $t('batchStartNew') }}</button>
        <button type="button" class="btn" @click="close">{{ $t('close') }}</button>
        <button v-if="successCount > 0" type="button" class="btn btn--primary" @click="downloadZip">
          <UiIcon name="file-archive" />
          <span>{{ $t('batchDownloadZip') }}</span>
        </button>
      </template>
    </template>
  </UiModal>
</template>

<script>
import JSZip from 'jszip';
import { save } from '../utils';
import parseWorkerMeshes from '../model-worker/meshes';
import UiModal from './ui/UiModal.vue';
import UiIcon from './ui/UiIcon.vue';

export default {
  name: 'TextBatchModeModal',
  components: { UiModal, UiIcon },
  props: {
    options: Object,
    exporter: Object,
    stlType: String,
  },
  emits: ['close'],
  data() {
    return {
      simpleTextInput: '',
      isProcessing: false,
      aborted: false,
      processedCount: 0,
      totalCount: 0,
      currentItemLabel: '',
      successCount: 0,
      errorResults: [],
      showResults: false,
      generatedFiles: [],
      countdownSeconds: 5,
      countdownInterval: null,
      hasAutoDownloaded: false,
    };
  },
  computed: {
    simpleTextLines() {
      if (!this.simpleTextInput.trim()) return [];
      return this.simpleTextInput
        .split('\n')
        .map((line) => line.trim())
        .filter((line) => line.length > 0);
    },
    canGenerate() {
      return this.simpleTextLines.length > 0;
    },
    progressPercent() {
      if (!this.totalCount) return 0;
      return Math.round((this.processedCount / this.totalCount) * 100);
    },
  },
  watch: {
    countdownSeconds(newVal) {
      if (newVal === 0 && this.successCount > 0 && !this.hasAutoDownloaded) {
        this.hasAutoDownloaded = true;
        this.downloadZip();
      }
    },
  },
  beforeDestroy() {
    this.stopCountdown();
  },
  methods: {
    close() {
      this.stopCountdown();
      this.$emit('close');
    },

    startCountdown() {
      this.countdownSeconds = 5;
      this.hasAutoDownloaded = false;
      this.countdownInterval = setInterval(() => {
        if (this.countdownSeconds > 0) {
          this.countdownSeconds -= 1;
        }
      }, 1000);
    },

    stopCountdown() {
      if (this.countdownInterval) {
        clearInterval(this.countdownInterval);
        this.countdownInterval = null;
      }
    },

    truncateValue(value) {
      if (!value) return '';
      const str = String(value);
      return str.length > 30 ? `${str.substring(0, 30)}...` : str;
    },

    sanitizeFilename(text) {
      const cleaned = String(text)
        .trim()
        .replace(/[\\/:*?"<>|]/g, '')
        .replace(/\s+/g, ' ')
        .trim();
      return cleaned || 'texto';
    },

    async startBatchGeneration() {
      this.isProcessing = true;
      this.aborted = false;
      this.processedCount = 0;
      this.successCount = 0;
      this.errorResults = [];
      this.generatedFiles = [];
      this.showResults = false;

      const modelWorker = (await import('@/model-worker')).default;

      const lines = this.simpleTextLines;
      this.totalCount = lines.length;

      // Si varias lineas dan el mismo nombre de archivo, se numeran
      // (p. ej. "Miguel 001", "Miguel 002"); los nombres unicos se quedan como estan.
      const baseNames = lines.map((line) => this.sanitizeFilename(line));
      const nameCounts = {};
      baseNames.forEach((name) => {
        nameCounts[name] = (nameCounts[name] || 0) + 1;
      });
      const nameOccurrence = {};

      for (let i = 0; i < lines.length; i += 1) {
        if (this.aborted) break;

        const textValue = lines[i];
        const rowIndex = i + 1;

        try {
          const rowOptions = JSON.parse(JSON.stringify(this.options));
          rowOptions.base.textMessage = textValue;

          this.currentItemLabel = this.truncateValue(textValue);

          // eslint-disable-next-line no-await-in-loop
          const meshes = await this.generateModelAsync(modelWorker, rowOptions);

          const baseName = baseNames[i];
          let filename = baseName;
          if (nameCounts[baseName] > 1) {
            nameOccurrence[baseName] = (nameOccurrence[baseName] || 0) + 1;
            filename = `${baseName} ${String(nameOccurrence[baseName]).padStart(3, '0')}`;
          }
          // eslint-disable-next-line no-await-in-loop
          await this.exportToBuffer(meshes, filename);

          this.successCount += 1;
        } catch (error) {
          this.errorResults.push({
            row: rowIndex,
            error: error.message || String(error),
          });
        }

        this.processedCount += 1;
      }

      this.isProcessing = false;
      this.showResults = true;
      if (this.successCount > 0) {
        this.startCountdown();
      }
    },

    generateModelAsync(modelWorker, options) {
      let timeoutId;
      const timeout = new Promise((resolve, reject) => {
        timeoutId = setTimeout(() => reject(new Error('Model generation timeout')), 30000);
      });

      const generation = modelWorker.request({
        mode: 'Text',
        options,
      }).then((result) => {
        if (!result.meshes) {
          throw new Error('No meshes in worker response');
        }
        const meshes = parseWorkerMeshes(result.meshes, { preview: false });
        if (Object.keys(meshes).length === 0) {
          throw new Error('Empty meshes object');
        }
        return meshes;
      });

      return Promise.race([generation, timeout]).finally(() => clearTimeout(timeoutId));
    },

    async exportToBuffer(meshes, filename) {
      const exportAsBinary = this.stlType === 'binary';

      if (meshes.combined) {
        const stlData = this.exporter.parse(meshes.combined, { binary: exportAsBinary });
        if (exportAsBinary) {
          const content = stlData.buffer ? new Uint8Array(stlData.buffer) : new Uint8Array(stlData);
          this.generatedFiles.push({ filename: `${filename}.stl`, data: content });
        } else {
          this.generatedFiles.push({ filename: `${filename}.stl`, data: stlData });
        }
      }
    },

    abortGeneration() {
      this.aborted = true;
    },

    async downloadZip() {
      const zip = new JSZip();

      this.generatedFiles.forEach((file) => {
        if (file.data instanceof Uint8Array) {
          zip.file(file.filename, file.data, { binary: true });
        } else {
          zip.file(file.filename, file.data);
        }
      });

      const timestamp = new Date().getTime();
      const zipBlob = await zip.generateAsync({ type: 'blob' });
      save(zipBlob, `texto_batch_${timestamp}.zip`);
    },

    reset() {
      this.stopCountdown();
      this.simpleTextInput = '';
      this.isProcessing = false;
      this.aborted = false;
      this.processedCount = 0;
      this.totalCount = 0;
      this.currentItemLabel = '';
      this.successCount = 0;
      this.errorResults = [];
      this.showResults = false;
      this.generatedFiles = [];
      this.countdownSeconds = 5;
      this.hasAutoDownloaded = false;
    },
  },
};
</script>

<style>
/* Mismos estilos que BatchModeModal: se repiten aqui para que la ventana de texto se vea bien aunque se abra sola. */
.batch {
  display: grid;
  gap: 16px;
}

.batch__options {
  display: flex;
  flex-wrap: wrap;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
}

.batch__parts {
  display: grid;
  gap: 4px;
  max-width: 420px;
}

.batch__mode-help {
  margin-top: -6px;
}

.batch__panel {
  display: grid;
  gap: 14px;
}

.howto {
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
  background: var(--surface-inset);
  color: var(--text-2);
  font-size: 13.5px;
}

.howto summary {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 14px;
  color: var(--text);
  font-weight: 600;
  cursor: pointer;
  list-style: none;
}

.howto summary::-webkit-details-marker {
  display: none;
}

.howto summary::after {
  content: "";
  width: 7px;
  height: 7px;
  margin-left: auto;
  border-right: 2px solid var(--text-3);
  border-bottom: 2px solid var(--text-3);
  transform: rotate(45deg);
  transition: transform var(--duration) var(--ease-out);
}

.howto[open] summary::after {
  transform: rotate(-135deg);
}

.howto summary .svg-icon {
  width: 17px;
  height: 17px;
  color: var(--info-text);
}

.howto ol,
.howto ul,
.howto p {
  margin: 0;
  padding: 0 14px 10px 34px;
  line-height: 1.55;
}

.howto ol {
  list-style: decimal;
}

.howto ul {
  list-style: disc;
}

.howto li {
  margin: 3px 0;
}

.howto p {
  padding-left: 14px;
}

.batch__textarea-wrap {
  position: relative;
}

.batch__textarea.textarea {
  min-height: 220px;
  padding-bottom: 36px;
}

.batch__count {
  position: absolute;
  right: 10px;
  bottom: 10px;
  padding: 3px 9px;
  border-radius: 999px;
  background: var(--surface-inset);
  color: var(--text-3);
  font-size: 12px;
  font-weight: 600;
  pointer-events: none;
  transition: background-color var(--duration) ease, color var(--duration) ease;
}

.batch__count.is-active {
  background: var(--accent-soft);
  color: var(--accent-text);
}

.batch__steps {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.batch-step {
  display: flex;
  gap: 12px;
  padding: 14px;
  border: 1px solid var(--border);
  border-radius: var(--radius-md);
}

.batch-step__number {
  display: inline-flex;
  flex-shrink: 0;
  align-items: center;
  justify-content: center;
  width: 26px;
  height: 26px;
  border-radius: 50%;
  background: var(--accent-soft);
  color: var(--accent-text);
  font-size: 13px;
  font-weight: 700;
}

.batch-step__content {
  display: grid;
  flex: 1 1 auto;
  align-content: start;
  justify-items: start;
  gap: 8px;
  min-width: 0;
}

.batch-step__content .file-drop {
  width: 100%;
}

.batch-step__title {
  margin: 2px 0 0;
  color: var(--text);
  font-size: 14px;
  font-weight: 600;
}

.batch__preview {
  display: grid;
  gap: 8px;
}

.batch__preview-head {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.batch__validation {
  display: inline-flex;
  gap: 6px;
}

.badge--danger {
  background: var(--danger-soft);
  color: var(--danger-text);
}

.data-table-wrap {
  max-height: 260px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  overflow: auto;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12.5px;
}

.data-table th,
.data-table td {
  max-width: 200px;
  padding: 7px 10px;
  border-bottom: 1px solid var(--divider);
  overflow: hidden;
  text-align: left;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.data-table th {
  position: sticky;
  top: 0;
  background: var(--surface-inset);
  color: var(--text-2);
  font-family: var(--font-mono);
  font-weight: 600;
}

.data-table tbody tr:nth-child(even) {
  background: var(--surface-inset);
}

.data-table__empty {
  color: var(--text-3);
}

.batch-progress {
  display: grid;
  gap: 10px;
  padding: 24px 4px;
}

.batch-progress__head {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.batch-progress__title {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: var(--text);
  font-size: 16px;
  font-weight: 600;
}

.batch-progress__title .svg-icon {
  color: var(--accent);
}

.batch-progress__percent {
  color: var(--accent-text);
  font-size: 15px;
  font-weight: 700;
  font-variant-numeric: tabular-nums;
}

.batch-progress .progress {
  height: 10px;
}

.batch-progress__current {
  margin: 0;
  overflow: hidden;
  color: var(--text-3);
  font-size: 13px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.batch-results {
  display: grid;
  gap: 20px;
}

.batch-results.has-ad {
  grid-template-columns: auto minmax(0, 1fr);
  align-items: start;
}

.batch-results__content {
  display: grid;
  gap: 12px;
}

.batch-results__errors {
  margin: 6px 0 0;
  padding-left: 18px;
  list-style: disc;
}

.batch-results__download {
  display: grid;
  gap: 10px;
}

.batch-results__countdown {
  margin: 0;
  color: var(--text);
  font-size: 15px;
  font-weight: 600;
}

@media (max-width: 760px) {
  .batch__steps,
  .batch-results.has-ad {
    grid-template-columns: 1fr;
  }
}
</style>
