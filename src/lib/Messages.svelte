<script lang="ts">
  // Iterate messages
  import type { Message, Chat } from "./Types.svelte";
  import { globalStorage } from "./Storage.svelte";
  import EditMessage from "./EditMessage.svelte";

  export let messages: Message[];
  export let chatId: number;
  export let chat: Chat;

  $: chatSettings = chat.settings;

  console.log(messages);
</script>

{#each messages as message, i}
  <!-- {#if !((message.summarized) && $globalStorage.hideSummarized) && !(i === 0 && message.role === 'system' && !chatSettings.useSystemPrompt)} -->
  {#key message.uuid}<EditMessage bind:message {chatId} {chat} />{/key}
  <!-- {/if} -->
{/each}
