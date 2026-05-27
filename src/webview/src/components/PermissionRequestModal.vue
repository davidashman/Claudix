<template>
  <div
    ref="containerRef"
    :class="['permission-request-container', { 'plan-modal': isPlanMode }]"
    tabIndex="0"
    data-permission-panel="1"
  >
    <div v-if="props.request.agentName" class="agent-badge">
      <span class="codicon codicon-robot"></span>
      <span>{{ props.request.agentName }}</span>
    </div>
    <div class="tool-preview">
      <ToolBlock :block="fakeBlock" :context="props.context" />
    </div>
    <div class="permission-header">Do you approve?</div>
    <input
      ref="inputRef"
      class="reject-message-input"
      placeholder="Tell Claude what to do instead (optional)"
      v-model="rejectMessage"
      @keydown="handleInputKeyDown"
    />
    <div class="button-row">
      <button class="btn btn-decline" @click="handleReject">
        Decline
        <span class="shortcut-hint">Esc</span>
      </button>
      <div class="btn-group-right">
        <button
          v-if="showSecondButton"
          class="btn btn-always-approve"
          @click="handleApproveAndDontAsk"
        >
          Always Approve
          <span class="shortcut-hint">⇧⌘↵</span>
        </button>
        <button class="btn btn-approve" @click="handleApprove">
          Approve
          <span class="shortcut-hint">⌘↵</span>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, provide, onMounted } from 'vue';
import type { PermissionRequest } from '../core/PermissionRequest';
import type { ToolContext } from '../types/tool';
import { useKeybinding } from '../utils/useKeybinding';
import ToolBlock from './Messages/blocks/ToolBlock.vue';

interface Props {
  request: PermissionRequest;
  context: ToolContext;
  onResolve: (request: PermissionRequest, allow: boolean) => void;
}

const props = defineProps<Props>();

const isPlanMode = computed(() => props.request.toolName === 'ExitPlanMode');
provide('toolForceCollapsed', false);
provide('toolForceExpanded', true);
provide('toolHideStatusIndicator', true);

const containerRef = ref<HTMLDivElement | null>(null);
const inputRef = ref<HTMLInputElement | null>(null);
const rejectMessage = ref('');

onMounted(() => {
  // Only grab focus if this webview's document is already focused. Otherwise,
  // a permission prompt arriving in one window would steal focus from an
  // input box the user is actively typing in elsewhere.
  if (document.hasFocus()) {
    containerRef.value?.focus();
  }
});

const showSecondButton = computed(
  () => props.request.suggestions && props.request.suggestions.length > 0
);

const fakeBlock = computed(() => ({
  type: 'tool_use' as const,
  id: 'permission-preview',
  name: props.request.toolName,
  input: props.request.inputs,
}));

const handleApprove = () => {
  props.onResolve(props.request, true);
};

const handleApproveAndDontAsk = () => {
  props.request.accept(props.request.inputs, props.request.suggestions || []);
};

const handleReject = () => {
  const trimmedMessage = rejectMessage.value.trim();
  const rejectionMessage = trimmedMessage
    ? `The user doesn't want to proceed with this tool use. The tool use was rejected (eg. if it was a file edit, the new_string was NOT written to the file). The user provided the following reason for the rejection: ${trimmedMessage}`
    : "The user doesn't want to proceed with this tool use. The tool use was rejected (eg. if it was a file edit, the new_string was NOT written to the file). STOP what you are doing and wait for the user to tell you how to proceed.";

  props.request.reject(rejectionMessage, !trimmedMessage);
};

// Enter in the input (without modifiers) submits the rejection message
const handleInputKeyDown = (e: KeyboardEvent) => {
  if (e.key === 'Enter' && !e.metaKey && !e.ctrlKey && !e.shiftKey) {
    e.preventDefault();
    handleReject();
  }
};

// Global keybindings — registered only while this modal is mounted,
// so they are naturally scoped to when a permission modal is showing.
// The webview only receives keyboard events when the window is active,
// so no additional window-focus guard is needed.
useKeybinding([
  {
    keys: 'escape',
    priority: 200,
    allowInEditable: true,
    handler: handleReject,
  },
  {
    keys: 'cmd+enter',
    priority: 200,
    allowInEditable: true,
    handler: handleApprove,
  },
  {
    keys: 'cmd+shift+enter',
    priority: 200,
    allowInEditable: true,
    handler: () => {
      if (showSecondButton.value) handleApproveAndDontAsk();
      else handleApprove();
    },
  },
]);
</script>

<style scoped>
.permission-request-container {
  display: flex;
  flex-direction: column;
  gap: 8px;
  outline: none;
  margin-bottom: 0;
  padding: 12px;
  background: var(--vscode-editorWidget-background, color-mix(in srgb, var(--vscode-foreground) 5%, var(--vscode-editor-background)));
  border-radius: 8px 8px 0 0;
  border: 1px solid #3794ff;
  border-bottom: none;
}

.agent-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 11px;
  font-weight: 500;
  color: var(--vscode-badge-foreground);
  background: var(--vscode-badge-background);
  border-radius: 10px;
  padding: 2px 8px 2px 6px;
  align-self: flex-start;
}

.agent-badge .codicon {
  font-size: 12px;
}

.tool-preview {
  margin: 0 -4px;
  max-height: 160px;
  overflow-y: auto;
}

.permission-header {
  font-size: 13px;
  font-weight: 600;
  color: var(--vscode-foreground);
  margin-bottom: 2px;
}

.reject-message-input {
  padding: 8px 12px;
  font-size: 13px;
  background: var(--vscode-input-background);
  color: var(--vscode-input-foreground);
  border: 1px solid var(--vscode-input-border);
  border-radius: 4px;
  outline: none;
  width: 100%;
  box-sizing: border-box;
}

.reject-message-input:focus {
  border-color: var(--vscode-focusBorder);
}

.reject-message-input::placeholder {
  color: var(--vscode-input-placeholderForeground);
}

.button-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}

.btn-group-right {
  display: flex;
  flex-direction: row;
  align-items: center;
  gap: 8px;
}

.btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  font-size: 13px;
  border-radius: 4px;
  cursor: pointer;
  border: 1px solid var(--vscode-button-border, transparent);
  transition: background-color 0.15s;
  white-space: nowrap;
}

.btn-decline {
  background: var(--vscode-button-secondaryBackground);
  color: var(--vscode-button-secondaryForeground);
}

.btn-decline:hover {
  background: var(--vscode-button-secondaryHoverBackground);
}

.btn-always-approve {
  background: var(--vscode-button-secondaryBackground);
  color: var(--vscode-button-secondaryForeground);
}

.btn-always-approve:hover {
  background: var(--vscode-button-secondaryHoverBackground);
}

.btn-approve {
  background: var(--vscode-button-background);
  color: var(--vscode-button-foreground);
}

.btn-approve:hover {
  background: var(--vscode-button-hoverBackground);
}

.shortcut-hint {
  font-size: 10px;
  opacity: 0.6;
  font-family: var(--vscode-editor-font-family, monospace);
  letter-spacing: 0;
}

.plan-modal {
  max-height: 66vh;
  display: flex;
  flex-direction: column;
}

.plan-modal .tool-preview {
  flex: 1;
  max-height: none;
  overflow-y: auto;
  min-height: 0;
}
</style>
