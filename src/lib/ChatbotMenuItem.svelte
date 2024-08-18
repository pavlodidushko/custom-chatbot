<script lang="ts">
  import { replace } from "svelte-spa-router";
  import type { Chat } from "./Types.svelte";
  import { deleteChat, pinMainMenu, saveChatStore } from "./Storage.svelte";
  import Fa from "svelte-fa/src/fa.svelte";
  import {
    faTrash,
    faCircleCheck,
    faPencil,
  } from "@fortawesome/free-solid-svg-icons/index";
  import { faCheckSquare } from "@fortawesome/free-regular-svg-icons/index";
  import { onMount } from "svelte";
  import { hasActiveModels } from "./Models.svelte";

  export let botmenuitem: string;
</script>

<li>
  <a
    href={`#/chat/customgpt/${botmenuitem.id}`}
    class="chat-menu-item"
    on:click={() => {
      $pinMainMenu = false;
    }}
  >
    <span class="chat-item-name"
      >{botmenuitem.name || `Chat ${botmenuitem}`}</span
    >
  </a>
</li>
