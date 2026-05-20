<template>
  <Tooltip :content="tooltipText" side="top" :side-offset="6">
    <div class="token-indicator">
      <span v-if="contextLabel" class="context-label">{{ contextLabel }}</span>
      <div class="progress-circle">
        <svg :width="size" :height="size" class="progress-svg">
          <circle
            :cx="center"
            :cy="center"
            :r="radius"
            :stroke="strokeColor"
            :stroke-width="STROKE_WIDTH"
            fill="none"
            opacity="0.25"
          />
          <circle
            :cx="center"
            :cy="center"
            :r="radius"
            :stroke="strokeColor"
            :stroke-width="STROKE_WIDTH"
            fill="none"
            stroke-linecap="round"
            opacity="0.9"
            :stroke-dasharray="circumference"
            :stroke-dashoffset="strokeOffset"
            :transform="`rotate(-90 ${center} ${center})`"
            class="progress-arc"
          />
        </svg>
      </div>
    </div>
  </Tooltip>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import Tooltip from './Common/Tooltip.vue'

interface Props {
  percentage: number
  contextTokens?: number
  contextWindow?: number
  contextTooltip?: string
  size?: number
}

const props = withDefaults(defineProps<Props>(), {
  percentage: 0,
  contextTooltip: '',
  size: 18
})

const STROKE_WIDTH = 2
const center = computed(() => (props.size / 2) - STROKE_WIDTH)
const radius = computed(() => center.value - STROKE_WIDTH)

const circumference = computed(() => {
  return 2 * Math.PI * radius.value
})

const strokeOffset = computed(() => {
  const progress = Math.max(0, Math.min(100, props.percentage))
  return circumference.value - (progress / 100) * circumference.value
})

const fmt = (n: number, f = 1) => {
  if (n >= 1_000_000) return `${(n / 1_000_000).toFixed(f)}M`
  if (n >= 1_000) return `${(n / 1_000).toFixed(f)}K`
  return `${n}`
}

const contextLabel = computed(() => {
  if (props.contextTokens == null || props.contextWindow == null) return undefined
  return `${fmt(props.contextTokens)} / ${fmt(props.contextWindow, 0)}`
})

const formattedPercentage = computed(() => {
  const value = props.percentage
  return `${value % 1 === 0 ? Math.round(value) : value.toFixed(1)}%`
})

const tooltipText = computed(() => {
  if (props.contextTooltip) {
    return `${formattedPercentage.value} · ${props.contextTooltip}`
  }
  return formattedPercentage.value
})

const strokeColor = computed(() => {
  if (props.percentage >= 80) {
    return 'color-mix(in srgb,var(--vscode-chart-red) 92%,transparent)'
  }
  else {
    return 'color-mix(in srgb,var(--vscode-foreground) 92%,transparent)'
  }
})
</script>

<style scoped>
.token-indicator {
  display: flex;
  align-items: center;
  gap: 4px;
}

.progress-circle {
  display: flex;
  align-items: center;
  flex-shrink: 0;
  margin-bottom: 2px;
}

.progress-arc {
  transition: stroke-dashoffset 0.3s ease;
}

.context-label {
  font-size: 12px;
  font-weight: 400;
  opacity: 0.45;
  white-space: nowrap;
  color: var(--vscode-foreground);
  height: 13px;
  line-height: 13px;
  font-variant-numeric: tabular-nums;
}
</style>