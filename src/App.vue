<template>
  <div id="app">
    <Main />
    <ToastHost />
  </div>
</template>

<script>
import Main from './components/Main.vue';
import ToastHost from './components/ToastHost.vue';

export default {
  name: 'app',
  components: {
    Main,
    ToastHost,
  },
};
</script>
