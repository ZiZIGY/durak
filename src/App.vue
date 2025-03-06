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
  const isConnected = ref(false);

  // Добавляем новые переменные для работы с аудио
  const isRecording = ref(false);
  const mediaRecorder = ref<MediaRecorder | null>(null);
  const audioChunks = ref<Blob[]>([]);

  onMounted(() => {
    peer.value = new Peer();

    peer.value.on('open', (id) => {
      myId.value = id;
    });

    peer.value.on('connection', (connection) => {
      const newUser: User = {
        id: connection.peer,
        nickname: '',
        conn: connection,
      };
      users.value.push(newUser);
      setupConnection(connection, newUser);
    });
  });

  const joinChat = () => {
    if (!nickname.value || isConnected.value) return;
    isConnected.value = true;

    // Если вы первый пользователь, просто ждите подключений
    // Если нет - подключитесь к существующему пиру
    const knownPeerId = prompt(
      'Введите ID существующего пользователя или оставьте пустым, если вы первый'
    );
    if (knownPeerId) {
      connectToUser(knownPeerId);
    }
  };

  const connectToUser = (userId: string) => {
    if (peer.value && userId !== myId.value) {
      try {
        const connection = peer.value.connect(userId, {
          reliable: true,
        });

        const newUser: User = {
          id: userId,
          nickname: '',
          conn: connection,
        };

        connection.on('open', () => {
          users.value.push(newUser);
          setupConnection(connection, newUser);

          // Отправляем наш никнейм
          connection.send({
            type: 'NICKNAME',
            nickname: nickname.value,
          });
        });
      } catch (err) {
        console.error('Ошибка при подключении:', err);
      }
    }
  };

  const setupConnection = (connection: any, user: User) => {
    connection.on('data', (data: any) => {
      if (data.type === 'NICKNAME') {
        user.nickname = data.nickname;
      } else if (data.type === 'MESSAGE') {
        messages.value.push({
          sender: user.nickname,
          type: 'text',
          text: data.message,
        });
      } else if (data.type === 'VOICE_MESSAGE') {
        messages.value.push({
          sender: data.sender,
          type: 'voice',
          audio: data.audio,
        });
      }
    });
  };

  const sendMessage = () => {
    if (!message.value || !isConnected.value) return;

    users.value.forEach((user) => {
      if (user.conn) {
        user.conn.send({
          type: 'MESSAGE',
          message: message.value,
        });
      }
    });

    messages.value.push({
      sender: nickname.value,
      type: 'text',
      text: message.value,
    });

    message.value = '';
  };

  // Функция для начала записи
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
        // Конвертируем Blob в base64
        const reader = new FileReader();
        reader.readAsDataURL(audioBlob);
        reader.onloadend = () => {
          const base64Audio = reader.result as string;
          // Отправляем аудио всем пользователям
          users.value.forEach((user) => {
            if (user.conn) {
              user.conn.send({
                type: 'VOICE_MESSAGE',
                audio: base64Audio,
                sender: nickname.value,
              });
            }
          });

          // Добавляем свое аудио в список сообщений
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

  // Функция для остановки записи
  const stopRecording = () => {
    if (mediaRecorder.value && isRecording.value) {
      mediaRecorder.value.stop();
      isRecording.value = false;
      // Останавливаем все треки
      mediaRecorder.value.stream.getTracks().forEach((track) => track.stop());
    }
  };
</script>

<template>
  <div class="chat-container">
    <div class="connection-info">
      <p>Ваш ID: {{ myId }}</p>
      <div class="connect-form">
        <input
          v-model="nickname"
          placeholder="Введите никнейм"
        />
        <button @click="joinChat">Войти в чат</button>
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
  </div>
</template>

<style scoped>
  .chat-container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
  }

  .connection-info {
    margin-bottom: 20px;
  }

  .connect-form {
    display: flex;
    gap: 10px;
    margin-bottom: 20px;
  }

  .messages {
    height: 400px;
    border: 1px solid #ccc;
    padding: 10px;
    margin-bottom: 20px;
    overflow-y: auto;
  }

  .message-input {
    display: flex;
    gap: 10px;
  }

  input {
    padding: 8px;
    flex: 1;
  }

  button {
    padding: 8px 16px;
    background-color: #42b883;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background-color: #3aa876;
  }

  .voice-message {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .voice-button {
    background-color: #ff4081;
  }

  .voice-button.recording {
    background-color: #ff1744;
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

  audio {
    height: 32px;
  }

  .message {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-bottom: 12px;
  }
</style>
