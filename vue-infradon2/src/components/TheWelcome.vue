<script setup lang="ts">
import { ref } from 'vue'

//model
const counter = ref(0)

const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Célibataire voici mon numéro : 079 735 11 66</h1>
  <p>{{ counter }}</p>
  <button @click="increment">Ajouter 1</button>
</template>
