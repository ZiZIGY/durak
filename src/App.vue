<script setup lang="ts">
  import { ref, onMounted } from 'vue';
  import Peer from 'peerjs';

  interface User {
    id: string;
    nickname: string;
    conn?: any;
  }

  interface Message {
    sender: string;
    type: 'text' | 'voice';
    text?: string;
    audio?: string;
    isOwnMessage: boolean;
    color: string;
  }

  const myId = ref('');
  const nickname = ref('');
  const message = ref('');
  const messages = ref<Message[]>([]);
  const peer = ref<Peer>();
  const users = ref<User[]>([]);

  const roomId = ref('');
  const isRecording = ref(false);
  const mediaRecorder = ref<MediaRecorder | null>(null);
  const audioChunks = ref<Blob[]>([]);
  const isDarkTheme = ref(
    window.matchMedia('(prefers-color-scheme: dark)').matches
  );
  const isConnected = ref(false);

  const messageInput = ref<HTMLTextAreaElement | null>(null);

  const userColor = ref('#42b883');

  onMounted(() => {
    peer.value = new Peer();

    peer.value.on('open', (id) => {
      myId.value = id;
      console.log('My peer ID:', id);
    });

    peer.value.on('connection', (connection) => {
      console.log('New connection:', connection.peer);
      const newUser: User = {
        id: connection.peer,
        nickname: '',
        conn: connection,
      };
      users.value.push(newUser);
      setupConnection(connection, newUser);
    });

    window
      .matchMedia('(prefers-color-scheme: dark)')
      .addEventListener('change', (event) => {
        isDarkTheme.value = event.matches;
      });
  });

  const joinRoom = () => {
    if (!nickname.value || !roomId.value) {
      alert('Введите ваш никнейм и ID комнаты!');
      return;
    }

    const connection = peer.value?.connect(roomId.value.trim());

    const newUser: User = {
      id: roomId.value,
      nickname: 'Хост',
      conn: connection,
    };

    connection?.on('open', () => {
      users.value.push(newUser);
      setupConnection(connection, newUser);
      connection.send({
        type: 'NICKNAME',
        nickname: nickname.value,
      });
      isConnected.value = true;
    });
  };

  const setupConnection = (connection: any, user: User) => {
    connection.on('data', (data: any) => {
      if (data.type === 'NICKNAME') {
        user.nickname = data.nickname;
      } else if (data.type === 'MESSAGE') {
        messages.value.push({
          sender: data.sender || user.nickname,
          type: 'text',
          text: data.message,
          isOwnMessage: false,
          color: data.color,
        });
      } else if (data.type === 'VOICE_MESSAGE') {
        messages.value.push({
          sender: data.sender,
          type: 'voice',
          audio: data.audio,
          isOwnMessage: false,
          color: data.color,
        });
      }
    });
  };

  const autoGrow = () => {
    if (messageInput.value) {
      messageInput.value.style.height = 'auto';
      messageInput.value.style.height = messageInput.value.scrollHeight + 'px';
    }
  };

  const sendMessage = (e: Event) => {
    if (e.type === 'keyup' && e instanceof KeyboardEvent) {
      if (e.shiftKey) return;
    }

    if (!message.value.trim()) return;

    messages.value.push({
      sender: nickname.value || 'Вы',
      type: 'text',
      text: message.value,
      isOwnMessage: true,
      color: userColor.value,
    });

    users.value.forEach((user) => {
      if (user.conn && user.conn.open) {
        try {
          user.conn.send({
            type: 'MESSAGE',
            message: message.value,
            sender: nickname.value || 'Пользователь',
            color: userColor.value,
          });
        } catch (err) {
          console.error('Ошибка при отправке сообщения:', err);
        }
      }
    });

    message.value = '';

    if (messageInput.value) {
      messageInput.value.style.height = 'auto';
    }
  };

  const startRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      mediaRecorder.value = new MediaRecorder(stream);
      audioChunks.value = [];
      isRecording.value = true;

      mediaRecorder.value.ondataavailable = (event) => {
        audioChunks.value.push(event.data);
      };

      mediaRecorder.value.onstop = async () => {
        const audioBlob = new Blob(audioChunks.value, { type: 'audio/webm' });
        const reader = new FileReader();
        reader.readAsDataURL(audioBlob);
        reader.onloadend = () => {
          const base64Audio = reader.result as string;
          users.value.forEach((user) => {
            if (user.conn) {
              user.conn.send({
                type: 'VOICE_MESSAGE',
                audio: base64Audio,
                sender: nickname.value,
              });
            }
          });

          messages.value.push({
            sender: nickname.value,
            type: 'voice',
            audio: base64Audio,
            isOwnMessage: true,
            color: userColor.value,
          });
        };
      };

      mediaRecorder.value.start();
    } catch (err) {
      console.error('Ошибка при записи аудио:', err);
    }
  };

  const stopRecording = () => {
    if (mediaRecorder.value && isRecording.value) {
      mediaRecorder.value.stop();
      isRecording.value = false;
      mediaRecorder.value.stream.getTracks().forEach((track) => track.stop());
    }
  };

  const copyId = async () => {
    try {
      await navigator.clipboard.writeText(myId.value);
    } catch (err) {
      console.error('Ошибка при копировании:', err);
      const textArea = document.createElement('textarea');
      textArea.value = myId.value;
      document.body.appendChild(textArea);
      textArea.select();
      try {
        document.execCommand('copy');
      } catch (err) {
        console.error('Ошибка при копировании:', err);
      }
      document.body.removeChild(textArea);
    }
  };
</script>

<template>
  <div
    class="chat-container"
    :class="{ dark: isDarkTheme }"
  >
    <div class="connection-info">
      <div class="id-container">
        <p>ID комнаты: {{ myId }} 🔑</p>
        <button
          class="copy-button"
          @click="copyId"
          title="Копировать ID комнаты"
        >
          📋
        </button>
      </div>
      <div class="connect-form">
        <input
          v-model="nickname"
          placeholder="Введите ваш никнейм 👤"
        />
        <input
          type="color"
          v-model="userColor"
          title="Выберите ваш цвет"
          class="color-picker"
        />
        <input
          v-if="!isConnected"
          v-model="roomId"
          placeholder="ID комнаты для подключения 🔑"
        />
        <button
          v-if="!isConnected"
          @click="joinRoom"
        >
          Присоединиться к комнате 🚪
        </button>
        <button
          v-else
          class="host-button"
        >
          Вы подключены к комнате ✅
        </button>
      </div>
    </div>

    <div class="messages">
      <div
        v-for="(msg, index) in messages"
        :key="index"
        class="message"
        :class="{
          'voice-message': msg.type === 'voice',
          'own-message': msg.isOwnMessage,
          'other-message': !msg.isOwnMessage,
        }"
        :style="{ backgroundColor: msg.color + '80' }"
      >
        <strong>{{ msg.sender }}:</strong>
        <template v-if="msg.type === 'text'">
          {{ msg.text }}
        </template>
        <template v-else-if="msg.type === 'voice'">
          <audio
            controls
            :src="msg.audio"
          ></audio>
        </template>
      </div>
    </div>

    <div class="message-input">
      <textarea
        ref="messageInput"
        v-model="message"
        @input="autoGrow"
        @keyup.enter.exact="sendMessage"
        placeholder="Введите сообщение ✍️"
        rows="1"
      ></textarea>
      <button
        class="send-button"
        @click="sendMessage"
        title="Отправить"
      >
        📨
      </button>
      <button
        class="voice-button"
        @mousedown="startRecording"
        @mouseup="stopRecording"
        :class="{ recording: isRecording }"
        title="Голосовое сообщение"
      >
        {{ isRecording ? '🔴' : '🎤' }}
      </button>
    </div>

    <div class="connection-status">
      <p
        >Участники чата:
        {{ users.map((u) => u.nickname || 'Гость').join(', ') }}</p
      >
    </div>
  </div>
</template>

<style scoped>
  .chat-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    color: var(--text-color);
    background-color: var(--bg-color);
  }

  .connection-info {
    background-color: var(--surface-color);
    padding: 15px;
    border-radius: 8px;
    margin-bottom: 20px;
  }

  .id-container {
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 15px;
  }

  .id-container p {
    font-weight: bold;
    margin: 0;
  }

  .copy-button {
    padding: 4px 8px;
    font-size: 1.2em;
    cursor: pointer;
    background: none;
    border: none;
    transition: transform 0.2s;
  }

  .copy-button:hover {
    transform: scale(1.1);
  }

  .copy-button:active {
    transform: scale(0.95);
  }

  .connect-form {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
  }

  .connect-form input {
    min-width: 200px;
  }

  .messages {
    height: 400px;
    border: 1px solid var(--border-color);
    background-color: var(--surface-color);
    padding: 10px;
    margin-bottom: 20px;
    overflow-y: auto;
  }

  .message {
    margin-bottom: 8px;
  }

  .voice-message {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .message-input {
    display: flex;
    gap: 10px;
    align-items: flex-start;
  }

  textarea {
    padding: 8px;
    flex: 1;
    min-height: 40px;
    max-height: 150px;
    resize: none;
    border-radius: 20px;
    color: var(--text-color);
    background-color: var(--input-bg-color);
    border: 1px solid var(--border-color);
    font-family: inherit;
    font-size: inherit;
    line-height: 1.4;
  }

  .send-button,
  .voice-button {
    width: 40px;
    height: 40px;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    font-size: 1.2em;
    transition: all 0.2s ease;
    background-color: var(--button-bg-color);
  }

  .voice-button {
    background-color: var(--voice-button-bg);
  }

  .voice-button.recording {
    background-color: var(--recording-bg);
    animation: pulse 1s infinite;
  }

  .host-button {
    background-color: #2196f3;
    cursor: default;
  }

  .host-button:hover {
    background-color: #2196f3;
  }

  .connection-status {
    margin-top: 15px;
    padding: 10px;
    background-color: var(--surface-color);
    border-radius: 4px;
  }

  @media (max-width: 600px) {
    .connect-form {
      flex-direction: column;
    }

    .connect-form input,
    .connect-form button {
      width: 100%;
    }

    .message-input {
      gap: 8px;
    }

    .send-button,
    .voice-button {
      width: 36px;
      height: 36px;
    }
  }

  audio {
    height: 32px;
    filter: var(--audio-filter);
  }

  .chat-container.dark audio {
    --audio-filter: invert(100%);
  }

  /* Светлая тема */
  .chat-container {
    --text-color: #000000;
    --bg-color: #ffffff;
    --surface-color: #f5f5f5;
    --border-color: #cccccc;
    --input-bg-color: #ffffff;
    --primary-color: #42b883;
    --primary-hover-color: #3aa876;
    --accent-color: #ff4081;
    --accent-active-color: #ff1744;
    --button-bg-color: #e0e0e0;
    --voice-button-bg: #e0e0e0;
    --recording-bg: #ff4444;
  }

  /* Темная тема */
  .chat-container.dark {
    --text-color: #ffffff;
    --bg-color: #121212;
    --surface-color: #1e1e1e;
    --border-color: #333333;
    --input-bg-color: #2d2d2d;
    --primary-color: #42b883;
    --primary-hover-color: #3aa876;
    --accent-color: #ff4081;
    --accent-active-color: #ff1744;
    --button-bg-color: #424242;
    --voice-button-bg: #424242;
    --recording-bg: #ff4444;
  }

  @media (prefers-color-scheme: dark) {
    .chat-container {
      --text-color: #ffffff;
      --bg-color: #121212;
      --surface-color: #1e1e1e;
      --border-color: #333333;
      --input-bg-color: #2d2d2d;
    }
  }

  @keyframes pulse {
    0% {
      transform: scale(1);
    }
    50% {
      transform: scale(1.1);
    }
    100% {
      transform: scale(1);
    }
  }

  .send-button:hover,
  .voice-button:hover {
    transform: scale(1.1);
  }

  .send-button:active,
  .voice-button:active {
    transform: scale(0.95);
  }

  .voice-message audio {
    border-radius: 20px;
    background-color: var(--button-bg-color);
  }

  .color-picker {
    width: 40px;
    height: 40px;
    aspect-ratio: 1/1;
    min-width: 40px !important;
    padding: 0;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  .color-picker::-webkit-color-swatch-wrapper {
    padding: 0;
  }

  .color-picker::-webkit-color-swatch {
    border: none;
    border-radius: 4px;
  }

  .messages {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .message {
    max-width: 80%;
    padding: 10px;
    border-radius: 12px;
    word-break: break-word;
  }

  .own-message {
    align-self: flex-start;
    border-bottom-left-radius: 4px;
  }

  .other-message {
    align-self: flex-end;
    border-bottom-right-radius: 4px;
  }

  .voice-message {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .voice-message audio {
    width: 200px;
  }

  @media (max-width: 600px) {
    .message {
      max-width: 90%;
    }

    .voice-message audio {
      width: 150px;
    }
  }
</style>
