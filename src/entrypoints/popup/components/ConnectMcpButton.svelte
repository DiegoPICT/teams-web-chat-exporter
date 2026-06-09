<script lang="ts">
  import { createEventDispatcher } from 'svelte';
  import { Plug, Unplug } from 'lucide-svelte';
  import { t } from '../../../i18n/i18n';
  import type { McpConnectionState } from '../../../types/messaging';

  export let state: McpConnectionState = 'DISCONNECTED';
  export let disabled = false;
  export let lang = 'en';
  export let lastError = '';
  export let inline = false;

  const dispatch = createEventDispatcher<{ connect: void; disconnect: void }>();

  $: isConnected = state !== 'DISCONNECTED' && state !== 'ERROR';
  $: isTransitioning = state === 'CONNECTING';
  $: buttonLabel = isTransitioning
    ? (t('mcp.connecting', {}, lang) || 'Connecting MCP…')
    : isConnected
      ? (t('mcp.disconnect', {}, lang) || 'Disconnect MCP')
      : (t('mcp.connect', {}, lang) || 'Connect MCP');
  $: stateLabel = (() => {
    const map: Record<McpConnectionState, string> = {
      DISCONNECTED: 'mcp.state.disconnected',
      CONNECTING: 'mcp.state.connecting',
      CONNECTED_IDLE: 'mcp.state.connectedIdle',
      BUSY: 'mcp.state.busy',
      CONTEXT_LOST: 'mcp.state.contextLost',
      ERROR: 'mcp.state.error',
    };
    const key = map[state] || 'mcp.state.disconnected';
    return t(key, {}, lang);
  })();

  function onClick() {
    if (disabled || isTransitioning) return;
    if (isConnected) dispatch('disconnect');
    else dispatch('connect');
  }
</script>

<button
  class="mcp-connect"
  class:inline
  type="button"
  disabled={disabled || isTransitioning}
  on:click={onClick}
>
  <span class="left">
    <span class="icon">{#if isConnected}<Unplug size={16} />{:else}<Plug size={16} />{/if}</span>
    <span class="title">{buttonLabel}</span>
  </span>
</button>
<div class="mcp-meta" class:inline aria-live="polite">
  <span class="state">{stateLabel}</span>
  {#if lastError}
    <span class="error">{t('mcp.lastError', { error: lastError }, lang) || `Last error: ${lastError}`}</span>
  {/if}
</div>

<style>
  .mcp-connect {
    width: 100%;
    border: 1px solid var(--color-border);
    border-radius: 10px;
    background: var(--color-surface);
    color: var(--color-text);
    padding: 10px 12px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    cursor: pointer;
    transition: border-color 0.15s ease, background 0.15s ease;
    margin-bottom: 6px;
  }
  .mcp-connect.inline {
    margin-bottom: 0;
    height: 100%;
  }
  .mcp-connect:hover:not(:disabled) {
    border-color: var(--color-accent);
    background: var(--color-accent-light);
  }
  .mcp-connect:disabled {
    opacity: 0.65;
    cursor: not-allowed;
  }
  .left {
    display: inline-flex;
    align-items: center;
    gap: 8px;
  }
  .icon {
    width: 16px;
    height: 16px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
  }
  .title {
    font-size: 13px;
    font-weight: 600;
  }
  .mcp-meta {
    margin-bottom: 12px;
    font-size: 11px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }
  .mcp-meta.inline {
    margin-bottom: 0;
    margin-top: 4px;
  }
  .state {
    color: var(--color-subtle);
  }
  .error {
    color: var(--color-danger);
  }
</style>
