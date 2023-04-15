<script setup lang="ts">
interface LanguageProps {
  value: string;
  label: string;
}

const LANGUAGES: LanguageProps[] = [
  { value: 'en-US', label: 'English US' },
  { value: 'en-GB', label: 'English UK' },
  // { value: 'es-ES', label: 'Spanish' },
  // { value: 'fr-FR', label: 'French' },
  // { value: 'de-DE', label: 'German' },
  // { value: 'it-IT', label: 'Italian' },
  // { value: 'pt-BR', label: 'Portuguese' },
  // { value: 'ru-RU', label: 'Russian' },
  { value: 'ja-JP', label: 'Japanese' },
  { value: 'ko-KR', label: 'Korean' },
  { value: 'zh-CN', label: 'Chinese' },
];

const STORAGE_STATE_KEY = 'CHAT_APP_STATE';

interface MessageProps {
  role: string;
  content: string;
}

interface RecognitionProps {
  recognition: any;
  listening: boolean;
  continuous: boolean;
  lang: string;
  askAfterRecognition: boolean;
}

interface SpeechProps {
  content: string;
  voice: SpeechSynthesisVoice | null;
  voiceName: string;
  voices: SpeechSynthesisVoice[];
  speaking: boolean;
  checkSpeaking: any;
  lang: string;
}

interface ChatProps {
  apiKey: string;
  pending: boolean;
  content: string;
  messages: MessageProps[];
  system: string;
  temperature: number;
}

interface StateProps {
  recognition: RecognitionProps;
  speech: SpeechProps;
  chat: ChatProps;
  showSettings: boolean;
}

const starting = ref(true);

const DEFAULT_STATE: StateProps = {
  recognition: {
    recognition: null,
    listening: false,
    continuous: true,
    lang: 'en-US',
    askAfterRecognition: true,
  },
  speech: {
    content: '',
    voice: null,
    voiceName: '',
    voices: [],
    speaking: false,
    checkSpeaking: null,
    lang: LANGUAGES[0].value,
  },
  chat: {
    pending: false,
    content: '',
    messages: [],
    apiKey: '',
    system: '',
    temperature: 1,
  },
  showSettings: false,
};

let state = reactive<StateProps>(DEFAULT_STATE);

const speechLang = ref(LANGUAGES[0].value);

const getVoicesByLang = (lang: string) => {
  let voices = speechSynthesis.getVoices();
  console.log(voices);
  return voices.filter((voice) => voice.lang === lang);
};

watch(speechLang, (lang) => {
  state.speech = { ...state.speech, lang, voices: getVoicesByLang(lang) };
});

watch(state, (newValue) => {
  console.log('state changed', newValue);
  localStorage.setItem(STORAGE_STATE_KEY, JSON.stringify(newValue));
});

const start = () => {
  const stateValue = localStorage.getItem(STORAGE_STATE_KEY);
  const currentState = stateValue ? JSON.parse(stateValue) : DEFAULT_STATE;
  console.log('start', currentState);

  state.chat = currentState.chat;
  state.recognition = currentState.recognition;

  speechLang.value = currentState.speech.lang;
  state.speech = {
    ...currentState.speech,
    voices: getVoicesByLang(currentState.speech.lang),
  };

  starting.value = false;
};

const handleSelectVoice = (e: any) => {
  state.speech.voice =
    state.speech.voices.find((voice) => voice.name === e.target.value) || null;
};

const autoresize = (e: any) => {
  setTimeout(function () {
    e.target.style.cssText = 'height:' + e.target.scrollHeight + 'px';
  }, 0);
};

const speak = (content: string) => {
  const utterance = new SpeechSynthesisUtterance(content);
  const lang = state.speech.lang;
  utterance.lang = lang;
  utterance.rate = 1;
  utterance.pitch = 1;

  if (!state.speech.voice) {
    let voices = state.speech.voices;

    if (voices.length === 0) {
      state.speech.voices = getVoicesByLang(lang);
    }

    state.speech.voice = voices[0];
  }

  utterance.voice = state.speech.voice;
  // console.log(utterance);

  speechSynthesis.speak(utterance);

  state.speech.speaking = true;
  if (!state.speech.checkSpeaking) {
    state.speech.checkSpeaking = setInterval(() => {
      if (!speechSynthesis.speaking) {
        clearInterval(state.speech.checkSpeaking);
        state.speech.speaking = false;
        state.speech.checkSpeaking = null;
      }
    }, 100);
  }
};

const speakReply = (content: string) => {
  if (!content) {
    return;
  }

  state.speech.content += content;
  if (!['.', '?', '!', '\n'].some((c) => content.endsWith(c))) {
    return;
  }
  const speakContent = state.speech.content.trim();
  state.speech.content = '';

  speak(speakContent);
};

const streamReply = (content: string) => {
  if (!content) {
    return;
  }
  const currentMessage = state.chat.messages[state.chat.messages.length - 1];
  if (currentMessage.role === 'assistant') {
    currentMessage.content += content;
  } else {
    state.chat.messages.push({ role: 'assistant', content });
  }
  speakReply(content);
};

const ask = async () => {
  if (!state.chat.apiKey) {
    alert('Please enter your OpenAI API key.');
    return;
  }
  state.chat.pending = true;
  if (state.chat.messages.length === 0 && state.chat.system) {
    state.chat.messages.push({
      role: 'system',
      content: state.chat.system,
    });
  }
  state.chat.messages.push({ role: 'user', content: state.chat.content });
  state.chat.content = '';

  const response: any = await fetch(
    'https://api.openai.com/v1/chat/completions',
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${state.chat.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-3.5-turbo',
        messages: [...state.chat.messages],
        stream: true,
      }),
    }
  );

  state.chat.pending = false;

  const data = response.body;
  if (!data) {
    return;
  }

  const reader = data.getReader();
  const decoder = new TextDecoder();
  let done = false;

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
        if (state.speech.content) {
          speak(state.speech.content);
          state.speech.content = '';
        }
        break; // Stream finished
      }
      try {
        const parsed = JSON.parse(message);
        const text = parsed.choices[0].delta.content || '';
        streamReply(text);
      } catch (error) {
        console.error('Could not JSON parse stream message', message, error);
      }
    }
  }
};

const voice = () => {
  if (!state.recognition.recognition) {
    const speechRecognition = (window as any).webkitSpeechRecognition;
    const recognition = new speechRecognition();

    recognition.onresult = (event: any) => {
      // console.log('onresult', event);
      const current = event.resultIndex;
      const speechResult = event.results[current][0].transcript;
      state.chat.content += speechResult;
    };

    recognition.onspeechend = () => {
      recognition.stop();
      state.recognition.listening = false;
      console.log('Speech recognition has stopped.');
      if (state.recognition.askAfterRecognition) {
        setTimeout(() => {
          ask();
        }, 500);
      }
    };

    state.recognition.recognition = recognition;
  }

  const recognition = state.recognition.recognition;

  recognition.lang = state.recognition.lang;
  recognition.interimResults = false;
  recognition.maxAlternatives = 1;
  recognition.continuous = state.recognition.continuous;

  if (state.recognition.listening) {
    recognition.stop();
    state.recognition.listening = false;
  } else {
    recognition.start();
    state.recognition.listening = true;
  }
};
</script>

<template>
  <div v-if="starting" class="w-full h-screen flex items-center justify-center">
    <button
      type="button"
      class="text-white bg-gradient-to-br from-purple-600 to-blue-500 hover:bg-gradient-to-bl focus:ring-4 focus:outline-none focus:ring-blue-300 dark:focus:ring-blue-800 font-medium rounded-lg text-base px-5 py-3 text-center"
      @click="start"
    >
      Start
    </button>
  </div>
  <div v-if="!starting" class="mt-5 mb-28 flex flex-col gap-2">
    <div
      v-for="(message, i) in state.chat.messages"
      :key="i"
      class="flex"
      :class="{
        'justify-end': message.role === 'user',
        'justify-start': ['assistant', 'system'].includes(message.role),
      }"
    >
      <div
        class="rounded-lg p-2.5 text-sm text-gray-900 dark:text-white"
        :class="{
          'bg-blue-400 dark:bg-blue-700 text-gray-900 dark:text-white':
            message.role === 'user',
          'bg-gray-50 dark:bg-gray-700 text-gray-900 dark:text-white':
            message.role === 'assistant',
          'bg-orange-50 dark:bg-orange-700 text-orange-900 dark:text-orange':
            message.role === 'system',
        }"
        v-html="message.content.replaceAll('\n', '</br>')"
      ></div>
    </div>
  </div>
  <div v-if="!starting" class="fixed bottom-0 left-0 w-full">
    <div v-if="state.speech.speaking" class="py-2">
      <VoiceAnimation />
    </div>
    <div
      v-if="state.showSettings"
      class="flex flex-col px-3 py-5 gap-4 bg-gray-50 dark:bg-gray-700"
    >
      <div class="flex flex-col gap-1">
        <h6 class="text-lg font-bold dark:text-white">Voice Recognition</h6>
        <div>
          <label class="relative inline-flex items-center cursor-pointer">
            <input
              type="checkbox"
              :checked="state.recognition.continuous"
              @change="
                ($event) =>
                  (state.recognition.continuous = !state.recognition.continuous)
              "
              class="sr-only peer"
            />
            <div
              class="w-11 h-6 bg-gray-200 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 dark:peer-focus:ring-blue-800 rounded-full peer dark:bg-gray-700 peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all dark:border-gray-600 peer-checked:bg-blue-600"
            ></div>
            <span
              class="ml-3 text-sm font-medium text-gray-900 dark:text-gray-300"
              >Continuous recognition</span
            >
          </label>
        </div>
        <div>
          <label class="relative inline-flex items-center cursor-pointer">
            <input
              type="checkbox"
              :checked="state.recognition.askAfterRecognition"
              @change="
                ($event) =>
                  (state.recognition.askAfterRecognition =
                    !state.recognition.askAfterRecognition)
              "
              class="sr-only peer"
            />
            <div
              class="w-11 h-6 bg-gray-200 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 dark:peer-focus:ring-blue-800 rounded-full peer dark:bg-gray-700 peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all dark:border-gray-600 peer-checked:bg-blue-600"
            ></div>
            <span
              class="ml-3 text-sm font-medium text-gray-900 dark:text-gray-300"
              >Auto ask after recognition</span
            >
          </label>
        </div>
        <div class="flex items-center gap-4">
          <label class="block text-sm font-medium text-gray-900 dark:text-white"
            >Language</label
          >
          <select
            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
            v-model="state.recognition.lang"
            @change="(event) => (state.recognition.lang = event.target.value)"
          >
            <option
              v-for="lang in LANGUAGES"
              :key="lang.value"
              :value="lang.value"
            >
              {{ lang.label }}
            </option>
          </select>
        </div>
      </div>
      <div class="flex flex-col gap-1">
        <h6 class="text-lg font-bold dark:text-white">Speech</h6>
        <div class="flex items-center gap-4">
          <label class="block text-sm font-medium text-gray-900 dark:text-white"
            >Language</label
          >
          <select
            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
            v-model="speechLang"
            @change="(event) => (speechLang = event.target.value)"
          >
            <option
              v-for="lang in LANGUAGES"
              :key="lang.value"
              :value="lang.value"
            >
              {{ lang.label }}
            </option>
          </select>
        </div>
        <div class="flex items-center gap-4">
          <label class="block text-sm font-medium text-gray-900 dark:text-white"
            >Voice</label
          >
          <select
            class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
            v-model="state.speech.voiceName"
            @change="handleSelectVoice"
          >
            <option
              v-for="voice in state.speech.voices"
              :key="voice.name"
              :value="voice.name"
            >
              {{ voice.name }}
            </option>
          </select>
        </div>
      </div>
      <div class="flex flex-col gap-2">
        <h6 class="text-lg font-bold dark:text-white">Chat</h6>

        <input
          type="password"
          class="bg-gray-50 border border-gray-300 text-gray-900 text-sm rounded-lg focus:ring-blue-500 focus:border-blue-500 block w-full p-2.5 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
          placeholder="API Key"
          :value="state.chat.apiKey"
          @change="
            ($event) => {
              state.chat.apiKey = $event.target.value.trim();
            }
          "
        />

        <textarea
          rows="3"
          class="block p-2.5 w-full text-sm text-gray-900 bg-gray-50 rounded-lg border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-700 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500"
          placeholder="System message..."
          v-model="state.chat.system"
        ></textarea>

        <div class="flex items-center gap-3">
          <label
            class="block w-40 text-sm font-medium text-gray-900 dark:text-white"
          >
            Temperature: {{ state.chat.temperature }}
          </label>
          <input
            type="range"
            min="0"
            max="2"
            step="0.1"
            class="w-full h-1 bg-gray-100 rounded-lg appearance-none cursor-pointer range-sm dark:bg-gray-600"
            v-model="state.chat.temperature"
          />
        </div>
      </div>
      <div class="flex gap-4"></div>
    </div>
    <div class="flex items-center px-3 py-2 bg-gray-50 dark:bg-gray-700">
      <button
        type="button"
        class="inline-flex justify-center p-2 text-gray-500 rounded-lg cursor-pointer hover:text-gray-900 hover:bg-gray-100 dark:text-gray-400 dark:hover:text-white dark:hover:bg-gray-600"
        title="Settings"
        @click="state.showSettings = !state.showSettings"
      >
        <svg
          class="w-6 h-6"
          fill="none"
          stroke="currentColor"
          stroke-width="1.5"
          viewBox="0 0 24 24"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M9.594 3.94c.09-.542.56-.94 1.11-.94h2.593c.55 0 1.02.398 1.11.94l.213 1.281c.063.374.313.686.645.87.074.04.147.083.22.127.324.196.72.257 1.075.124l1.217-.456a1.125 1.125 0 011.37.49l1.296 2.247a1.125 1.125 0 01-.26 1.431l-1.003.827c-.293.24-.438.613-.431.992a6.759 6.759 0 010 .255c-.007.378.138.75.43.99l1.005.828c.424.35.534.954.26 1.43l-1.298 2.247a1.125 1.125 0 01-1.369.491l-1.217-.456c-.355-.133-.75-.072-1.076.124a6.57 6.57 0 01-.22.128c-.331.183-.581.495-.644.869l-.213 1.28c-.09.543-.56.941-1.11.941h-2.594c-.55 0-1.02-.398-1.11-.94l-.213-1.281c-.062-.374-.312-.686-.644-.87a6.52 6.52 0 01-.22-.127c-.325-.196-.72-.257-1.076-.124l-1.217.456a1.125 1.125 0 01-1.369-.49l-1.297-2.247a1.125 1.125 0 01.26-1.431l1.004-.827c.292-.24.437-.613.43-.992a6.932 6.932 0 010-.255c.007-.378-.138-.75-.43-.99l-1.004-.828a1.125 1.125 0 01-.26-1.43l1.297-2.247a1.125 1.125 0 011.37-.491l1.216.456c.356.133.751.072 1.076-.124.072-.044.146-.087.22-.128.332-.183.582-.495.644-.869l.214-1.281z"
          ></path>
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"
          ></path>
        </svg>
      </button>
      <button
        type="button"
        class="inline-flex justify-center p-2 text-gray-500 rounded-lg cursor-pointer hover:text-gray-900 hover:bg-gray-100 dark:text-gray-400 dark:hover:text-white dark:hover:bg-gray-600"
        title="New conversation"
        @click="state.chat.messages = []"
      >
        <svg
          class="w-6 h-6"
          fill="none"
          stroke="currentColor"
          stroke-width="1.5"
          viewBox="0 0 24 24"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M2.25 12.76c0 1.6 1.123 2.994 2.707 3.227 1.087.16 2.185.283 3.293.369V21l4.076-4.076a1.526 1.526 0 011.037-.443 48.282 48.282 0 005.68-.494c1.584-.233 2.707-1.626 2.707-3.228V6.741c0-1.602-1.123-2.995-2.707-3.228A48.394 48.394 0 0012 3c-2.392 0-4.744.175-7.043.513C3.373 3.746 2.25 5.14 2.25 6.741v6.018z"
          ></path>
        </svg>
        <span class="sr-only">New conversation</span>
      </button>

      <textarea
        rows="1"
        class="block mx-4 p-2.5 w-full text-sm text-gray-900 bg-white rounded-lg border border-gray-300 focus:ring-blue-500 focus:border-blue-500 dark:bg-gray-800 dark:border-gray-600 dark:placeholder-gray-400 dark:text-white dark:focus:ring-blue-500 dark:focus:border-blue-500 resize-none overflow-hidden box-border"
        placeholder="Your message..."
        v-model="state.chat.content"
        @keydown="autoresize"
      ></textarea>
      <button
        type="button"
        class="p-2 rounded-lg cursor-pointer hover:bg-gray-100 dark:hover:bg-gray-600"
        :class="{
          'text-red-500 hover:text-red-900  dark:text-red-400 dark:hover:text-red':
            state.recognition.listening,
          'text-gray-500 hover:text-gray-900  dark:text-gray-400 dark:hover:text-white ':
            !state.recognition.listening,
        }"
        title="Voice"
        @click="voice"
      >
        <svg
          class="w-6 h-6"
          fill="none"
          stroke="currentColor"
          stroke-width="1.5"
          viewBox="0 0 24 24"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            d="M12 18.75a6 6 0 006-6v-1.5m-6 7.5a6 6 0 01-6-6v-1.5m6 7.5v3.75m-3.75 0h7.5M12 15.75a3 3 0 01-3-3V4.5a3 3 0 116 0v8.25a3 3 0 01-3 3z"
          ></path>
        </svg>
        <span class="sr-only">Voice</span>
      </button>
      <button
        class="inline-flex justify-center p-2 text-blue-600 rounded-full cursor-pointer hover:bg-blue-100 dark:text-blue-500 dark:hover:bg-gray-600"
        @click="ask"
        :disabled="state.chat.pending || !state.chat.content"
      >
        <svg
          aria-hidden="true"
          class="w-6 h-6 rotate-90"
          fill="currentColor"
          viewBox="0 0 20 20"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            d="M10.894 2.553a1 1 0 00-1.788 0l-7 14a1 1 0 001.169 1.409l5-1.429A1 1 0 009 15.571V11a1 1 0 112 0v4.571a1 1 0 00.725.962l5 1.428a1 1 0 001.17-1.408l-7-14z"
          ></path>
        </svg>
        <span class="sr-only">Send message</span>
      </button>
    </div>
  </div>
</template>