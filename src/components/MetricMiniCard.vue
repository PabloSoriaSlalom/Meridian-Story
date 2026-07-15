<script setup lang="ts">
defineProps<{
  label: string
  value: string
}>()
</script>

<template>
  <div class="metric-mini-card">
    <span>{{ label }}</span>
    <strong>{{ value }}</strong>
  </div>
</template>

<style scoped>
.metric-mini-card {
  border: 1px solid rgba(72, 86, 150, 0.42);
  background: rgba(18, 24, 44, 0.84);
  border-radius: 14px;
  padding: 1rem;
}

.metric-mini-card span {
  display: block;
  color: #d0d2db;
}

.metric-mini-card strong {
  display: block;
  margin-top: 0.45rem;
  font-size: clamp(1.9rem, 4.6vw, 3.2rem);
  line-height: 0.95;
}
</style>
