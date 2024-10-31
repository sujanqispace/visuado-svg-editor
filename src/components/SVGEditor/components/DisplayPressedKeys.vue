<template>
  <div v-if="pressedKeys.length > 0" class="pressed-keys">
    {{ pressedKeys.join(' + ') }}
  </div>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import { onKeyStroke, onKeyUp } from '@vueuse/core'

const pressedKeys = ref([])

const addToPressedKeys = (key) => {
  let tempKeys = pressedKeys.value
  // if last key is the same as the current key, replace it otherwise add it
  if (tempKeys[tempKeys.length - 1] === key) {
    tempKeys[tempKeys.length - 1] = key
  } else {
    tempKeys.push(key)
  }

  while (tempKeys.length > 3) {
    fruits.shift()
  }

  pressedKeys.value = tempKeys
}

onKeyStroke(
  true,
  (e) => {
    if (e.key === ' ') {
      addToPressedKeys('Space')
    } else {
      addToPressedKeys(e.key)
    }
  },
  { dedupe: true }
)

onKeyUp((e) => {
  pressedKeys.value = []
})
</script>
<style scoped>
.pressed-keys {
  position: fixed;
  bottom: 40px;
  left: 10px;
  pointer-events: none;
  z-index: 1001;
  padding: 0.5rem;
  background-color: #000;
  color: #fff;
  font-size: 0.75rem;
  border-radius: 0.25rem;
  opacity: 0.8;
}
</style>
