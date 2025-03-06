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
        conn: connection
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
      conn: connection
    };

    connection?.on('open', () => {
      users.value.push(newUser);
      setupConnection(connection, newUser);
      connection.send({
        type: 'NICKNAME',
        nickname: nickname.value
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
          text: data.message
        });
      } else if (data.type === 'VOICE_MESSAGE') {
        messages.value.push({
          sender: data.sender,
          type: 'voice',
          audio: data.audio
        });
      }
    });
  };

  const sendMessage = () => {
    if (!message.value) return;

    messages.value.push({
      sender: nickname.value || 'Вы',
      type: 'text',
      text: message.value,
    });

    users.value.forEach((user) => {
      if (user.conn && user.conn.open) {
        try {
          user.conn.send({
            type: 'MESSAGE',
            message: message.value,
            sender: nickname.value || 'Пользователь',
          });
        } catch (err) {
          console.error('Ошибка при отправке сообщения:', err);
        }
      }
    });

    message.value = '';
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
        <p>ID комнаты: {{ myId }}</p>
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
          placeholder="Введите ваш никнейм"
        />
        <input
          v-if="!isConnected"
          v-model="roomId"
          placeholder="ID комнаты для подключения"
        />
        <button
          v-if="!isConnected"
          @click="joinRoom"
          >Присоединиться к комнате</button
        >
        <button
          v-else
          class="host-button"
          >Вы подключены к комнате</button
        >
      </div>
    </div>

    <div class="messages">
      <div
        v-for="(msg, index) in messages"
        :key="index"
        class="message"
        :class="{ 'voice-message': msg.type === 'voice' }"
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
      <input
        v-model="message"
        @keyup.enter="sendMessage"
        placeholder="Введите сообщение"
      />
      <button @click="sendMessage">Отправить</button>
      <button
        class="voice-button"
        @mousedown="startRecording"
        @mouseup="stopRecording"
        :class="{ recording: isRecording }"
      >
        {{ isRecording ? 'Запись...' : 'Голосовое сообщение' }}
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
    max-width: 800px;
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
  }

  input {
    padding: 8px;
    flex: 1;
    color: var(--text-color);
    background-color: var(--input-bg-color);
    border: 1px solid var(--border-color);
    border-radius: 4px;
  }

  button {
    padding: 8px 16px;
    background-color: var(--primary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background-color: var(--primary-hover-color);
  }

  .voice-button {
    background-color: var(--accent-color);
  }

  .voice-button.recording {
    background-color: var(--accent-active-color);
    animation: pulse 1s infinite;
  }

  @keyframes pulse {
    0% {
      opacity: 1;
    }
    50% {
      opacity: 0.5;
    }
    100% {
      opacity: 1;
    }
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
      flex-direction: column;
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
</style>
