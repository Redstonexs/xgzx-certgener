<!-- src/App.vue -->
<template>
  <div id="app">
    <h1>证书生成器</h1>
    <CertificateGenerator />
  </div>
</template>

<script>
import CertificateGenerator from './components/CertificateGenerator.vue';

export default {
  name: 'App',
  components: {
    CertificateGenerator
  }
};
</script>

<style>
#app {
  max-width: 800px;
  margin: 40px auto;
  text-align: center;
  font-family: Arial, sans-serif;
}
</style>
