<template>
  <!-- TaskOutput is surfaced under the originating Agent block; suppress standalone rendering -->
</template>

<script setup lang="ts">
defineProps<{
  toolUse?: any;
  toolResult?: any;
  toolUseResult?: any;
}>();
</script>
