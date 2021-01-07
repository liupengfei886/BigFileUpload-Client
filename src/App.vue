<template>
  <div id="app">
    <FileUpload />
  </div>
</template>

<script>
import FileUpload from './components/FileUpload.vue'

export default {
  name: 'app',
  components: {
    FileUpload
  }
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
}
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  height: 100vh;
}
</style>
