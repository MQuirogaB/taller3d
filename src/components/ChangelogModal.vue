<template>
  <UiModal title="Changelog" :subtitle="`v${version}`" icon="scroll-text" size="md" @close="close">
    <MarkdownRenderer :source="changelog" class="prose"></MarkdownRenderer>
    <template #footer>
      <button type="button" class="btn btn--primary" @click="close">OK</button>
    </template>
  </UiModal>
</template>

<script>
// eslint-disable-next-line import/no-webpack-loader-syntax
import changelog from '../../CHANGELOG.md?raw';
import packageJson from '../../package.json';
import { bus } from '../main';
import MarkdownRenderer from './MarkdownRenderer.vue';
import UiModal from './ui/UiModal.vue';

export default {
  name: 'ChangelogModal',
  components: {
    MarkdownRenderer,
    UiModal,
  },
  data() {
    return {
      changelog: changelog.split('\n').slice(3).join('\n'),
      version: packageJson.version,
    };
  },
  methods: {
    close() {
      bus.$emit('closeChangelogModal');
    },
  },
};
</script>

<style>
.changelog-intro {
  margin: 0 0 18px;
  padding: 12px 14px;
  border-radius: var(--radius-sm);
  background: var(--surface-inset);
  color: var(--text-2);
  font-size: 14px;
  line-height: 1.55;
}
</style>
