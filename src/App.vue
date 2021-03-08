<template>
  <PowergateUpload/>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import PowergateUpload from './components/PowergateUpload.vue';

    
export default defineComponent({
  name: 'App',
  components: {
    PowergateUpload
  },
});
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #123;
}

#app::after{
  content:""; 
  display: block;
  height: 10rem; 
}

/* {
  outline: 1px solid purple;
} /* */
</style>
