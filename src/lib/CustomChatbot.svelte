<script lang="ts">
  import {
    saveChatStore,
    chatsStorage,
    addMessage,
    updateChatSettings,
    checkStateChange,
    showSetChatSettings,
    submitExitingPromptsNow,
    continueMessage,
    getMessage,
    currentChatMessages,
    setCurrentChat,
    currentChatId,
  } from "./Storage.svelte";
  import { type Message, type Chat } from "./Types.svelte";
  import Prompts from "./Prompts.svelte";
  import Messages from "./Messages.svelte";
  import { restartProfile } from "./Profiles.svelte";
  import { afterUpdate, onMount, onDestroy } from "svelte";
  import Fa from "svelte-fa/src/fa.svelte";
  import {
    faClipboard,
    faArrowUpFromBracket,
    faPaperPlane,
    faGear,
    faPenToSquare,
    faMicrophone,
    faLightbulb,
    faCommentSlash,
    faCircleCheck,
    faUpload,
  } from "@fortawesome/free-solid-svg-icons/index";
  import { v4 as uuidv4 } from "uuid";
  import { getPrice } from "./Stats.svelte";
  import {
    autoGrowInputOnEvent,
    scrollToBottom,
    sizeTextElements,
  } from "./Util.svelte";
  import ChatSettingsModal from "./ChatSettingsModal.svelte";
  import Footer from "./Footer.svelte";
  import mammoth from "mammoth";
  import { openModal } from "svelte-modals";
  import PromptInput from "./PromptInput.svelte";
  import { ChatRequest } from "./ChatRequest.svelte";
  import { getModelDetail } from "./Models.svelte";
  import { marked } from "marked";
  let bupdated = false;
  export let params = { chatId: "" };
  const chatId: number = parseInt(params.chatId);

  console.log(chatId);

  const urls = [
    "https://hook.us1.make.com/hvimnl4ov8i413g6pnq514m39v66ydw2",
    "https://hook.us1.make.com/c4wu4lj74u73y7ro43upct2tt9fvn2h2",
    "https://hook.us1.make.com/tayvp8pb5uriet4dqdprv1o8vfd5pr4s",
  ];
  let chatRequest = new ChatRequest();
  let input: HTMLTextAreaElement;
  let recognition: any = null;
  let recording = false;
  let lastSubmitRecorded = false;

  $: chat = $chatsStorage.find((chat) => chat.id === chatId) as Chat;
  $: chatSettings = chat?.settings;
  let showSettingsModal;

  let scDelay;
  const onStateChange = (...args: any) => {
    if (!chat) return;
    clearTimeout(scDelay);
    setTimeout(() => {
      if (chat.startSession) {
        restartProfile(chatId);
        if (chat.startSession) {
          chat.startSession = false;
          saveChatStore();
          // Auto start the session
          submitForm(false, true);
        }
      }
      if ($showSetChatSettings) {
        $showSetChatSettings = false;
        showSettingsModal();
      }
      if ($submitExitingPromptsNow) {
        $submitExitingPromptsNow = false;
        submitForm(false, true);
      }
      if ($continueMessage) {
        const message = getMessage(chatId, $continueMessage);
        $continueMessage = "";
        if (
          message &&
          $currentChatMessages.indexOf(message) ===
            $currentChatMessages.length - 1
        ) {
          submitForm(lastSubmitRecorded, true, message);
        }
      }
    });
  };

  $: onStateChange(
    $checkStateChange,
    $showSetChatSettings,
    $submitExitingPromptsNow,
    $continueMessage,
  );

  const afterChatLoad = (...args: any) => {
    scrollToBottom();
  };

  $: afterChatLoad($currentChatId);

  setCurrentChat(0);
  // Make sure chat object is ready to go
  updateChatSettings(chatId);

  onDestroy(async () => {
    // clean up
    // abort any pending requests.
    chatRequest.controller.abort();
    ttsStop();
  });

  function convertMarkdownToHtml(text) {
    // Simple markdown to HTML conversion
    const replacements = [
      { pattern: /^### (.*$)/gim, replacement: "<h3>$1</h3>" },
      { pattern: /^#### (.*$)/gim, replacement: "<h4>$1</h4>" },
      { pattern: /^- (.*$)/gim, replacement: "<li>$1</li>" },
      { pattern: /\*\*(.*)\*\*/gim, replacement: "<strong>$1</strong>" },
      { pattern: /\*(.*)\*/gim, replacement: "<em>$1</em>" },
      { pattern: /---/gim, replacement: "<hr>" },
      { pattern: /\n$/gim, replacement: "<br>" },
    ];

    let html = text;
    replacements.forEach(({ pattern, replacement }) => {
      html = html.replace(pattern, replacement);
    });

    // Wrap list items with <ul> tags
    html = html.replace(/(<li>.*<\/li>)/gim, "<ul>$1</ul>");

    return html;
  }
  let formattedHtml = "";
  onMount(async () => {
    if (!chat) return;

    setCurrentChat(chatId);

    const dropZone = document.getElementById("drop-zone");
    dropZone.addEventListener("dragover", handleDragOver);
    dropZone.addEventListener("drop", handleDrop);
    chatRequest = new ChatRequest();
    chatRequest.setChat(chat);

    chat.lastAccess = Date.now();
    saveChatStore();
    $checkStateChange++;

    // Focus the input on mount
    focusInput();

    // Try to detect speech recognition support
    if ("SpeechRecognition" in window) {
      // @ts-ignore
      recognition = new window.SpeechRecognition();
    } else if ("webkitSpeechRecognition" in window) {
      // @ts-ignore
      recognition = new window.webkitSpeechRecognition(); // eslint-disable-line new-cap
    }

    if (recognition) {
      recognition.interimResults = false;
      recognition.onstart = () => {
        recording = true;
      };
      recognition.onresult = (event) => {
        // Stop speech recognition, submit the form and remove the pulse
        const last = event.results.length - 1;
        const text = event.results[last][0].transcript;
        input.value = text;
        recognition.stop();
        recording = false;
        submitForm(true);
      };
    } else {
      console.log("Speech recognition not supported");
    }
    if (chat.startSession) {
      restartProfile(chatId);
      if (chat.startSession) {
        chat.startSession = false;
        saveChatStore();
        // Auto start the session
        setTimeout(() => {
          submitForm(false, true);
        }, 0);
      }
    }
  });

  // Scroll to the bottom of the chat on update
  afterUpdate(() => {
    sizeTextElements();
    // Scroll to the bottom of the page after any updates to the messages array
    // focusInput()
  });

  // Scroll to the bottom of the chat on update
  const focusInput = () => {
    input.focus();
    scrollToBottom();
  };

  const addNewMessage = () => {
    if (chatRequest.updating) return;
    let inputMessage: Message;
    const lastMessage = $currentChatMessages[$currentChatMessages.length - 1];
    const uuid = uuidv4();
    if ($currentChatMessages.length === 0) {
      inputMessage = { role: "system", content: input.value, uuid };
    } else if (lastMessage && lastMessage.role === "user") {
      inputMessage = { role: "assistant", content: input.value, uuid };
    } else {
      inputMessage = { role: "user", content: input.value, uuid };
    }
    addMessage(chatId, inputMessage);

    // Clear the input value
    input.value = "";
    // input.blur()
    focusInput();
  };

  const ttsStart = (text: string, recorded: boolean) => {
    // Use TTS to read the response, if query was recorded
    if (recorded && "SpeechSynthesisUtterance" in window) {
      const utterance = new SpeechSynthesisUtterance(text);
      window.speechSynthesis.speak(utterance);
    }
  };

  const ttsStop = () => {
    if ("SpeechSynthesisUtterance" in window) {
      window.speechSynthesis.cancel();
    }
  };

  let waitingForCancel: any = 0;

  const cancelRequest = () => {
    if (!waitingForCancel) {
      // wait a second for another click to avoid accidental cancel
      waitingForCancel = setTimeout(() => {
        waitingForCancel = 0;
      }, 1000);
      return;
    }
    clearTimeout(waitingForCancel);
    waitingForCancel = 0;
    chatRequest.controller.abort();
  };

  const submitForm = async (
    recorded: boolean = false,
    skipInput: boolean = false,
    fillMessage: Message | undefined = undefined,
  ): Promise<void> => {
    fileNames = [];
    // Compose the system prompt message if there are no messages yet - disabled for now
    if (chatRequest.updating) return;

    lastSubmitRecorded = recorded;

    if (!skipInput) {
      chat.sessionStarted = true;
      saveChatStore();
      if (input.value !== "") {
        // Compose the input message
        const inputMessage: Message = {
          role: "user",
          content: input.value,
          uuid: uuidv4(),
        };
        addMessage(chatId, inputMessage);
      } else if (
        !fillMessage &&
        $currentChatMessages.length &&
        $currentChatMessages[$currentChatMessages.length - 1].role ===
          "assistant"
      ) {
        fillMessage = $currentChatMessages[$currentChatMessages.length - 1];
      }

      // Clear the input value
      input.value = "";
      input.blur();

      // Resize back to single line height
      input.style.height = "auto";
    }
    focusInput();

    let doScroll = true;
    let didScroll = false;

    const checkUserScroll = (e: Event) => {
      const el = e.target as HTMLElement;
      if (el && e.isTrusted && didScroll) {
        // from user
        doScroll =
          window.innerHeight + window.scrollY + 10 >=
          document.body.offsetHeight;
      }
    };

    window.addEventListener("scroll", checkUserScroll);
    chatRequest.updating = true;
    for (let index = 0; index < 3; index++) {
      try {
        const url = urls[index];
        const data = { prompt: gExtractedText };
        chatRequest.updating = true;
        chatRequest.updatingMessage = "";

        try {
          const response = await fetch(url, {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              Authorization: `Bearer sk-newsletter-pro-chatbot-eBgrhZ0TmYhPzOWhHVh3T3BlbkFJPSDsEtCA0EayoONNgckd`,
            },
            body: JSON.stringify(data),
          });
          console.log(response);

          const jsonResponse = await response.json();
          console.log(jsonResponse, response);

          if (true) {
            console.log(jsonResponse);

            const message = jsonResponse[0].message;
            $currentChatMessages.push({
              role: "assistant",
              content: message.content,
              uuid: "5e83807d-74bc-40bd-8a2d-6059cb7e029b",
              model: "gpt-4",
              finish_reason: "stop",
              usage: {
                completion_tokens: 9,
                total_tokens: 18,
                prompt_tokens: 9,
              },
              created: 1719709086042,
            });
            updateChatSettings(-3);
            // chatsStorage.set($currentChatMessages);
            console.log($currentChatMessages.length);
            // formattedHtml = convertMarkdownToHtml(message.content);
            if (message) {
              ttsStart(message.content, recorded);
            }
            bupdated = !bupdated;

            // output.textContent = JSON.stringify(jsonResponse, null, 2);
          } else {
            const errorResponse = await jsonResponse.json();
            console.log(errorResponse);

            // output.textContent = `Error: ${response.status} - ${JSON.stringify(errorResponse, null, 2)}`;
          }
        } catch (error) {
          console.log(error);

          // output.textContent = `Error: ${error.message}`;
        }

        scrollToBottom();
        chatRequest.updating = false;
        chatRequest.updatingMessage = "";

        // const url = urls[chatId.toString()];

        // test for open ai

        // const message = response.getMessages()[0];
        // if (message) {
        //   ttsStart(message.content, recorded);
        // }
      } catch (e) {
        console.error(e);
      }
    }
    window.removeEventListener("scroll", checkUserScroll);

    focusInput();
    // window.location.reload();
  };

  const suggestName = async (): Promise<void> => {
    const suggestMessage: Message = {
      role: "user",
      content:
        "Using appropriate language, please tell me a short 6 word summary of this conversation's topic for use as a book title. Only respond with the summary.",
      uuid: uuidv4(),
    };

    const suggestMessages = $currentChatMessages.slice(0, 10); // limit to first 10 messages
    suggestMessages.push(suggestMessage);

    chatRequest.updating = true;
    chatRequest.updatingMessage = "Getting suggestion for chat name...";
    const response = await chatRequest.sendRequest(suggestMessages, {
      chat,
      autoAddMessages: false,
      streaming: false,
      summaryRequest: true,
      maxTokens: 30,
    });

    try {
      await response.promiseToFinish();
    } catch (e) {
      console.error("Error generating name suggestion", e, e.stack);
    }
    chatRequest.updating = false;
    chatRequest.updatingMessage = "";
    if (response.hasError()) {
      addMessage(chatId, {
        role: "error",
        content: `Unable to get suggested name: ${response.getError()}`,
        uuid: uuidv4(),
      });
    } else {
      response.getMessages().forEach((m) => {
        const name = m.content
          .split(/\s+/)
          .slice(0, 8)
          .join(" ")
          .replace(/^[^a-z0-9!?]+|[^a-z0-9!?]+$/gi, "")
          .trim();
        if (name) chat.name = name;
      });
      saveChatStore();
    }
  };

  function promptRename() {
    openModal(PromptInput, {
      title: "Enter Name for Chat",
      label: "Name",
      value: chat.name,
      class: "is-info",
      onSubmit: (value) => {
        chat.name = (value || "").trim() || chat.name;
        saveChatStore();
        $checkStateChange++;
      },
    });
  }

  const recordToggle = () => {
    ttsStop();
    if (chatRequest.updating) return;
    // Check if already recording - if so, stop - else start
    if (recording) {
      recognition?.stop();
      recording = false;
    } else {
      recognition?.start();
    }
  };

  let fileNames = [];
  let gExtractedText = "";

  const uploadFiles = async (event) => {
    const files = event.target.files || event.dataTransfer.files;
    if (files && files.length > 0) {
      let extractedText = "";
      fileNames = [
        ...fileNames,
        ...Array.from(files).map((file) => ({
          name: file.name,
          file: file,
        })),
      ];

      for (const { file } of fileNames) {
        const fileType = file.type;
        const arrayBuffer = await file.arrayBuffer();

        if (fileType === "application/pdf") {
          const loadingTask = pdfjsLib.getDocument({ data: arrayBuffer });
          const pdf = await loadingTask.promise;

          for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
            const page = await pdf.getPage(pageNum);
            const textContent = await page.getTextContent();
            const pageText = textContent.items
              .map((item) => item.str)
              .join(" ");
            extractedText += pageText + " ";
          }
        } else if (
          fileType === "application/msword" ||
          fileType ===
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
        ) {
          const doc = await mammoth.extractRawText({ arrayBuffer });
          extractedText += doc.value + " ";
        } else if (fileType === "text/plain") {
          extractedText += new TextDecoder().decode(arrayBuffer) + " ";
        } else if (
          fileType ===
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" ||
          fileType === "text/csv"
        ) {
          const text = new TextDecoder().decode(arrayBuffer);
          extractedText += text + " ";
        }
      }

      gExtractedText = extractedText.trim();
      console.log(gExtractedText);
    }
  };

  const removeFile = (index) => {
    fileNames.splice(index, 1);
    fileNames = [...fileNames];
  };

  const handleDragOver = (event) => {
    event.preventDefault();
    event.dataTransfer.dropEffect = "copy";
  };

  const handleDrop = (event) => {
    event.preventDefault();
    uploadFiles(event);
  };

  onMount(() => {
    const dropZone = document.getElementById("drop-zone");
    dropZone.addEventListener("dragover", handleDragOver);
    dropZone.addEventListener("drop", handleDrop);
  });
  let count = 0;
</script>

{#if chat}
  <ChatSettingsModal {chatId} bind:show={showSettingsModal} />
  <div
    class="chat-page"
    style="--running-totals: {Object.entries(chat.usage || {}).length}"
  >
    <div class="chat-content">
      <nav class="level chat-header">
        <div class="level-left">
          <div class="level-item">
            <p class="subtitle is-5">
              <span>{chat.name || `Chat ${chat.id}`}</span>
              <a
                href={"#"}
                class="greyscale ml-2 is-hidden has-text-weight-bold editbutton"
                title="Rename chat"
                on:click|preventDefault={promptRename}
                ><Fa icon={faPenToSquare} /></a
              >
              <a
                href={"#"}
                class="greyscale ml-2 is-hidden has-text-weight-bold editbutton"
                title="Suggest a chat name"
                on:click|preventDefault={suggestName}
                ><Fa icon={faLightbulb} /></a
              >
            </p>
          </div>
        </div>

        <div class="level-right">
          <div class="level-item">
            <!-- <button class="button is-warning" on:click={() => { clearMessages(chatId); window.location.reload() }}><span class="greyscale mr-2"><Fa icon={faTrash} /></span> Clear messages</button> -->
          </div>
        </div>
      </nav>

      <Messages messages={$currentChatMessages} {chatId} {chat} />
      {#if chatRequest.updating === true || $currentChatId === 0}
        <article class="message is-success assistant-message">
          <div class="message-body content">
            <span class="is-loading"></span>
            <span>{@html formattedHtml}</span>
          </div>
        </article>
      {/if}
      <!-- {#if chatRequest.updating === true || $currentChatId === 0}
        <article class="message is-success assistant-message">
          <div class="message-body content">
            <span class="is-loading"></span>
            <span>{chatRequest.updatingMessage}</span>
          </div>
        </article>
      {/if} -->
      <!-- 
      {#if $currentChatId !== 0 && ($currentChatMessages.length === 0 || ($currentChatMessages.length === 1 && $currentChatMessages[0].role === "system"))}
        <Prompts bind:input />
      {/if} -->
    </div>
    <Footer class="prompt-input-container" strongMask={true}>
      <form
        class="field has-addons has-addons-right is-align-items-flex-end"
        on:submit|preventDefault={() => submitForm()}
      >
        <p class="control is-expanded"></p>
        <div id="drop-zone" class="drop-zone">
          <div class="file-input-wrapper">
            <button
              type="button"
              on:click={() => input.click()}
              class="attach-button"
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                fill="none"
                viewBox="0 0 24 24"
                ><path
                  fill="currentColor"
                  fill-rule="evenodd"
                  d="M9 7a5 5 0 0 1 10 0v8a7 7 0 1 1-14 0V9a1 1 0 0 1 2 0v6a5 5 0 0 0 10 0V7a3 3 0 1 0-6 0v8a1 1 0 1 0 2 0V9a1 1 0 1 1 2 0v6a3 3 0 1 1-6 0z"
                  clip-rule="evenodd"
                ></path></svg
              >
            </button>
          </div>
          {#if fileNames.length === 0}
            <p>Drag and drop files here, or click to select files</p>
          {/if}
          <input
            class="input is-info is-focused chat-input auto-size"
            type="file"
            accept=".doc,.docx,.pdf,.txt,.csv,.xlsx,.gdoc"
            placeholder="Upload your file here..."
            multiple
            on:change={uploadFiles}
            bind:this={input}
            style="display:none;"
          />

          <div class="file-list">
            {#each fileNames as { name }, index}
              <div class="file-item">
                <span>{name}</span>
                <button on:click={() => removeFile(index)}>X</button>
              </div>
            {/each}
          </div>
        </div>
        <!-- <textarea
            class="input is-info is-focused chat-input auto-size"
            placeholder="Type your message here..."
            rows="1"
            on:keydown={(e) => {
              // Only send if Enter is pressed, not Shift+Enter
              if (e.key === "Enter" && !e.shiftKey) {
                e.stopPropagation();
                submitForm();
                e.preventDefault();
              }
            }}
            on:input={(e) => autoGrowInputOnEvent(e)}
            bind:this={input}
          /> -->
        <!-- </p> -->
        <!-- <p class="control mic">
          <button class="button" on:click|preventDefault={uploadFile}
            ><span class="icon"><Fa icon={faUpload} /></span></button
          >
        </p>
        <p class="control mic" class:is-hidden={!recognition}>
          <button
            class="button"
            class:is-disabled={chatRequest.updating}
            class:is-pulse={recording}
            on:click|preventDefault={recordToggle}
            ><span class="icon"><Fa icon={faMicrophone} /></span></button
          >
        </p>
        <p class="control settings">
          <button
            title="Chat/Profile Settings"
            class="button"
            on:click|preventDefault={showSettingsModal}
            ><span class="icon"><Fa icon={faGear} /></span></button
          >
        </p> -->
        <!-- <p class="control queue">
          <button
            title="Queue message, don't send yet"
            class:is-disabled={chatRequest.updating}
            class="button is-ghost"
            on:click|preventDefault={addNewMessage}
            ><span class="icon"><Fa icon={faArrowUpFromBracket} /></span
            ></button
          >
        </p> -->
        {#if chatRequest.updating}
          <p class="control send">
            <button
              title="Cancel Response"
              class="button is-danger"
              type="button"
              on:click={cancelRequest}
              ><span class="icon">
                {#if waitingForCancel}
                  <Fa icon={faCircleCheck} />
                {:else}
                  <Fa icon={faCommentSlash} />
                {/if}
              </span></button
            >
          </p>
        {:else}
          <p class="control send">
            <button title="Send" class="button is-info" type="submit"
              ><span class="icon"><Fa icon={faPaperPlane} /></span></button
            >
          </p>
        {/if}
      </form>
      <!-- a target to scroll to -->
      <div class="content has-text-centered running-total-container">
        {#each Object.entries(chat.usage || {}) as [model, usage]}
          <p class="is-size-7 running-totals">
            <em>{getModelDetail(model || "").label || model}</em> total
            <span class="has-text-weight-bold">{usage.total_tokens}</span>
            tokens ~=
            <span class="has-text-weight-bold"
              >${getPrice(usage, model).toFixed(6)}</span
            >
          </p>
        {/each}
      </div>
    </Footer>
    <div></div>
  </div>
{/if}
