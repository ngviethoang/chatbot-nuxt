<script setup lang="ts">
const JOURNAL_APP_SETTINGS_KEY = 'JOURNAL_APP_SETTINGS';

const { promptTemplate, askWhenStart, canEditTemplate, defaultQuestion } =
  defineProps<{
    promptTemplate: string;
    askWhenStart?: boolean;
    canEditTemplate?: boolean;
    defaultQuestion?: string;
  }>();

const state = reactive({
  title: '',
  content: '',
  ai: {
    apiKey: '',
    question: defaultQuestion || '',
    template: promptTemplate,
  },
  showSettings: false,
  visualViewportHeight: 0,
});

const askAi = async () => {
  state.ai.question = 'I am thinking...';

  const messages = [
    {
      role: 'user',
      content: `${state.ai.template}\n${state.content}`,
    },
  ];

  const response: any = await fetch(
    'https://api.openai.com/v1/chat/completions',
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${state.ai.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo',
        messages,
        stream: true,
      }),
    }
  );

  const data = response.body;
  if (!data) {
    return;
  }

  const reader = data.getReader();
  const decoder = new TextDecoder();
  let done = false;
  state.ai.question = '';

  while (!done) {
    const { value, done: doneReading } = await reader.read();
    done = doneReading;
    const chunkValue = decoder.decode(value);
    // console.log('chunkValue', chunkValue);

    const lines = chunkValue
      ?.toString()
      ?.split('\n')
      .filter((line) => line.trim() !== '');
    for (const line of lines) {
      const message = line.replace(/^data: /, '');
      if (message === '[DONE]') {
        break; // Stream finished
      }
      try {
        const parsed = JSON.parse(message);
        const text = parsed.choices[0].delta.content || '';

        state.ai.question += text;
      } catch (error) {
        console.error('Could not JSON parse stream message', message, error);
      }
    }
  }
};

const onFocusContent = () => {
  if ('visualViewport' in window) {
    const VIEWPORT_VS_CLIENT_HEIGHT_RATIO = 0.75;
    window.visualViewport?.addEventListener('resize', (event: any) => {
      if (
        (event.target.height * event.target.scale) / window.screen.height <
        VIEWPORT_VS_CLIENT_HEIGHT_RATIO
      ) {
        state.visualViewportHeight =
          (event.target.height / event.target.scale) *
          VIEWPORT_VS_CLIENT_HEIGHT_RATIO;
      } else {
        state.visualViewportHeight = 0;
      }
    });
  }
};

const start = () => {
  const settings = JSON.parse(
    localStorage.getItem(JOURNAL_APP_SETTINGS_KEY) || '{}'
  );
  state.ai.apiKey = settings.apiKey || '';
  // state.ai.template = settings.aiTemplate || DEFAULT_AI_TEMPLATE;

  if (askWhenStart) {
    askAi();
  }
};

start();
</script>

<template>
  <div
    class="w-full h-screen text-sm text-gray-800 dark:text-white flex flex-col gap-3 py-3"
    :style="{
      height: `calc(100vh - ${state.visualViewportHeight}px)`,
    }"
  >
    <input
      class="block w-full text-lg px-0 bg-transparent dark:bg-transparent dark:placeholder-gray-400 focus:ring-0 outline-none"
      placeholder="Title"
      v-model="state.title"
    />
    <p class="italic font-thin">
      {{
        new Date().toLocaleDateString('en-US', {
          weekday: 'long',
          year: 'numeric',
          month: 'long',
          day: 'numeric',
        })
      }}
    </p>
    <input
      v-if="canEditTemplate"
      class="block w-full italic text-sm px-0 bg-transparent dark:bg-transparent dark:placeholder-gray-400 focus:ring-0 outline-none"
      placeholder="Template"
      v-model="state.ai.template"
    />

    <textarea
      id="editor"
      class="block h-full w-full px-0 bg-transparent dark:bg-transparent dark:placeholder-gray-400 focus:ring-0 outline-none resize-none"
      placeholder="Your thought... Press enter to ask AI."
      v-model="state.content"
      @keypress.enter="askAi"
      @focus="onFocusContent"
    ></textarea>
    <div class="flex gap-1 items-end">
      <button class="w-9 h-9" @click="state.showSettings = !state.showSettings">
        <img src="/favicon.ico" />
      </button>
      <JournalSettings
        v-if="state.showSettings"
        :title="state.title"
        :content="state.content"
        :template="state.ai.template"
        @change-template="state.ai.template = $event"
        @closed="state.showSettings = false"
      />
      <div
        v-if="state.ai.question"
        class="chat-bubble left w-full bg-blue-200 dark:bg-blue-500"
        @click="askAi"
      >
        {{ state.ai.question }}
      </div>
    </div>
  </div>
</template>
