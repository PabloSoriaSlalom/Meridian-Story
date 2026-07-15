<script setup lang="ts">
const props = withDefaults(
  defineProps<{
    imageSrc: string
    overlayFrom?: string
    overlayTo?: string
    toneFrom?: string
    toneTo?: string
    position?: string
  }>(),
  {
    overlayFrom: 'rgba(5, 9, 18, 0.18)',
    overlayTo: 'rgba(5, 9, 18, 0.38)',
    toneFrom: 'rgba(10, 16, 31, 0.1)',
    toneTo: 'rgba(6, 12, 24, 0.26)',
    position: 'center center',
  },
)
</script>

<template>
  <div class="chapter-background" aria-hidden="true">
    <div
      class="background-image"
      :style="{ backgroundImage: `url(${props.imageSrc})`, backgroundPosition: props.position }"
    ></div>
    <div
      class="background-tone"
      :style="{ background: `linear-gradient(132deg, ${props.toneFrom}, ${props.toneTo})` }"
    ></div>
    <div
      class="background-overlay"
      :style="{ background: `linear-gradient(180deg, ${props.overlayFrom}, ${props.overlayTo})` }"
    ></div>
  </div>
</template>

<style scoped>
.chapter-background {
  position: absolute;
  inset: 0;
  opacity: 0.72;
}

.background-image,
.background-tone,
.background-overlay {
  position: absolute;
  inset: 0;
}

.background-image {
  background-size: cover;
  background-repeat: no-repeat;
}
</style>
