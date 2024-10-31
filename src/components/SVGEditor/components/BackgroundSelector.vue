<template>
  <!-- File Input -->

  <v-file-input v-model="selectedFile" accept="image/*" density="compact">
    <template v-slot:append>
      <!-- Clear Button -->
      <v-btn icon="mdi-close" size="x-small" @click="clearFile"></v-btn>
    </template>
  </v-file-input>

  <!-- Image Preview -->
  <v-img
    v-if="previewUrl"
    :src="previewUrl"
    alt="Image Preview"
    max-height="80"
    class="my-2"
  ></v-img>
  <!-- {{ previewUrl }} -->
</template>

<script setup>
import { ref, computed, watch, defineEmits } from 'vue'
const selectedFile = ref(null)
const fileName = ref(null)

const emits = defineEmits(['setBackground'])

watch(selectedFile, (oldFile, newFile) => {
  if (newFile) {
    fileName.value = newFile.name
    console.log('newfile', newFile)
  }
})

const clearFile = () => {
  selectedFile.value = null
  fileName.value = null
  emits('setBackground', null)
}
const previewUrl = computed(() => {
  if (selectedFile.value) {
    return URL.createObjectURL(selectedFile.value)
  } else {
    return ''
  }
})
watch(previewUrl, (newUrl, oldUrl) => {
  console.log('new previewUrl', newUrl, oldUrl, 'hel1')
  if (newUrl) {
    console.log('new previewUrl', newUrl, oldUrl, 'hel')
    emits('setBackground', newUrl)
  }
})
</script>
