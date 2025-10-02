<script setup lang="ts">

import {ref} from 'vue';
const counter = ref(20);

const increment = () => {
  counter.value++;
}

</script>

<template>
  <h1>Hello world</h1>
  <button @click="increment">Increment</button>
  <p style="color: black; font-size: 50px;">{{ counter }}</p>
</template>
