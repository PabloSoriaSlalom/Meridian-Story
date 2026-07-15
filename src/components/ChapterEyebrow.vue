<script setup lang="ts">
defineOptions({ inheritAttrs: false })

defineProps<{
  text: string
}>()
</script>

<template>
  <p class="chapter-kicker" v-bind="$attrs">{{ text }}</p>
</template>
