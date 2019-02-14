<template>
  <div id="app">
    <SimpleP2P title="Ayame P2P Demo(Vue Version)" />
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import SimpleP2P from './components/SimpleP2P.vue';

@Component({
  components: {
    SimpleP2P,
  },
})
export default class App extends Vue {}
</script>

<style>
#app {
  text-align: center;
}
body {
  background-color: #333;
  color: #fff;
  font-family: "Roboto", "Open Sans", "Lucida Grande", sans-serif;
  height: 100%;
  margin: 0;
  padding: 0;
  width: 100%;
}
button {
  background-color: #4285f4;
  border: none;
  border-radius: 2px;
  box-shadow: 1px 1px 5px 0 rgba(0, 0, 0, 0.5);
  color: white;
  font-size: 0.8em;
  height: 2.75em;
  margin: 0 5px 20px 5px;
  padding: 0.5em 0.7em 0.5em 0.7em;
  width: 8em;
}
</style>
