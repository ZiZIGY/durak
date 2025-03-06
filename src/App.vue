<script setup lang="ts">
  import { ref, onMounted, watch, nextTick } from 'vue';
  import Peer from 'peerjs';

  interface User {
    id: string;
    nickname: string;
    conn?: any;
    color?: string;
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

  // Добавляем новые состояния для звонка
  const isInCall = ref(false);
  const incomingCall = ref(false);
  const localStream = ref<MediaStream | null>(null);
  const remoteStreams = ref<Map<string, MediaStream>>(new Map());
  const audioElements = ref<Map<string, HTMLAudioElement>>(new Map());

  // Добавляем ref для контейнера сообщений
  const messagesContainer = ref<HTMLDivElement | null>(null);

  // Добавляем watch для автоскролла при новых сообщениях
  watch(
    () => messages.value.length,
    () => {
      nextTick(() => {
        if (messagesContainer.value) {
          messagesContainer.value.scrollTop =
            messagesContainer.value.scrollHeight;
        }
      });
    }
  );

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
      color: userColor.value,
    };

    connection?.on('open', () => {
      users.value.push(newUser);
      setupConnection(connection, newUser);
      connection?.send({
        type: 'NICKNAME',
        nickname: nickname.value,
        color: userColor.value,
      });
      isConnected.value = true;
    });
  };

  const setupConnection = (connection: any, user: User) => {
    connection.on('data', (data: any) => {
      console.log('Получены данные:', data);

      if (data.type === 'NICKNAME') {
        user.nickname = data.nickname;
        user.color = data.color;

        // Если мы хост, отправляем всем обновленный список пользователей
        if (peer.value?.id === myId.value) {
          const usersList = users.value.map(u => ({
            id: u.id,
            nickname: u.nickname,
            color: u.color
          }));
          
          users.value.forEach(otherUser => {
            if (otherUser.conn && otherUser.conn.open) {
              otherUser.conn.send({
                type: 'USERS_LIST',
                users: usersList
              });
            }
          });
        }
      } else if (data.type === 'USERS_LIST') {
        users.value = data.users.map((u: any) => ({
          ...u,
          conn: users.value.find(existingUser => existingUser.id === u.id)?.conn
        }));
      } else if (data.type === 'MESSAGE') {
        // Добавляем сообщение в свой список
        messages.value.push({
          sender: data.sender || user.nickname,
          type: 'text',
          text: data.message,
          isOwnMessage: false,
          color: data.color
        });

        // Если мы хост, ретранслируем сообщение остальным
        if (peer.value?.id === myId.value) {
          users.value.forEach(otherUser => {
            if (otherUser.id !== user.id && otherUser.conn && otherUser.conn.open) {
              otherUser.conn.send(data);
            }
          });
        }
      } else if (data.type === 'VOICE_MESSAGE') {
        messages.value.push({
          sender: data.sender,
          type: 'voice',
          audio: data.audio,
          isOwnMessage: false,
          color: data.color,
        });

        if (peer.value?.id === myId.value) {
          users.value.forEach((otherUser) => {
            if (
              otherUser.id !== user.id &&
              otherUser.conn &&
              otherUser.conn.open
            ) {
              otherUser.conn.send({
                type: 'VOICE_MESSAGE',
                audio: data.audio,
                sender: data.sender,
                color: data.color,
              });
            }
          });
        }
      } else if (data.type === 'CALL_INVITE') {
        if (!isInCall.value) {
          incomingCall.value = true;
        }
      } else if (data.type === 'CALL_ACCEPTED') {
        console.log(`${data.accepter} присоединился к звонку`);
      } else if (data.type === 'CALL_DECLINED') {
        console.log(`${data.decliner} отклонил звонок`);
      } else if (data.type === 'CALL_ENDED') {
        if (isInCall.value) {
          endCall();
        }
      }
    });

    // Обновляем обработку звонков
    peer.value?.on('call', async (call) => {
      try {
        if (!localStream.value) {
          localStream.value = await navigator.mediaDevices.getUserMedia({ audio: true });
        }
        
        call.answer(localStream.value);
        
        call.on('stream', (remoteStream: MediaStream) => {
          console.log('Получен удаленный стрим:', call.peer);
          addRemoteStream(call.peer, remoteStream);
        });

        call.on('error', (err) => {
          console.error('Ошибка в звонке:', err);
        });
      } catch (err) {
        console.error('Ошибка при ответе на звонок:', err);
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
    
    // Добавляем свое сообщение в список
    const newMessage = {
      sender: nickname.value || 'Вы',
      type: 'text' as const,
      text: message.value,
      isOwnMessage: true,
      color: userColor.value
    };
    
    messages.value.push(newMessage);

    // Отправляем сообщение всем пользователям
    users.value.forEach(user => {
      if (user.conn && user.conn.open) {
        try {
          user.conn.send({
            type: 'MESSAGE',
            message: message.value,
            sender: nickname.value || 'Пользователь',
            color: userColor.value
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

          messages.value.push({
            sender: nickname.value,
            type: 'voice',
            audio: base64Audio,
            isOwnMessage: true,
            color: userColor.value,
          });

          users.value.forEach((user) => {
            if (user.conn) {
              user.conn.send({
                type: 'VOICE_MESSAGE',
                audio: base64Audio,
                sender: nickname.value,
                color: userColor.value,
              });
            }
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

  // Добавляем функции для звонка
  const startCall = async () => {
    try {
      localStream.value = await navigator.mediaDevices.getUserMedia({
        audio: true,
      });
      isInCall.value = true;

      // Отправляем всем приглашение в звонок
      users.value.forEach((user) => {
        if (user.conn && user.conn.open) {
          user.conn.send({
            type: 'CALL_INVITE',
            caller: nickname.value,
          });
        }
      });

      // Инициализируем аудио соединения
      setupAudioStream();
    } catch (err) {
      console.error('Ошибка при начале звонка:', err);
      alert('Не удалось получить доступ к микрофону');
    }
  };

  const acceptCall = async () => {
    try {
      localStream.value = await navigator.mediaDevices.getUserMedia({
        audio: true,
      });
      incomingCall.value = false;
      isInCall.value = true;

      // Отправляем подтверждение
      users.value.forEach((user) => {
        if (user.conn && user.conn.open) {
          user.conn.send({
            type: 'CALL_ACCEPTED',
            accepter: nickname.value,
          });
        }
      });

      setupAudioStream();
    } catch (err) {
      console.error('Ошибка при присоединении к звонку:', err);
      alert('Не удалось получить доступ к микрофону');
    }
  };

  const declineCall = () => {
    incomingCall.value = false;
    users.value.forEach((user) => {
      if (user.conn && user.conn.open) {
        user.conn.send({
          type: 'CALL_DECLINED',
          decliner: nickname.value,
        });
      }
    });
  };

  const endCall = () => {
    isInCall.value = false;
    if (localStream.value) {
      localStream.value.getTracks().forEach((track) => track.stop());
      localStream.value = null;
    }

    // Очищаем удаленные потоки
    remoteStreams.value.clear();
    audioElements.value.forEach((audio) => audio.remove());
    audioElements.value.clear();

    // Уведомляем остальных
    users.value.forEach((user) => {
      if (user.conn && user.conn.open) {
        user.conn.send({
          type: 'CALL_ENDED',
          ender: nickname.value,
        });
      }
    });
  };

  const setupAudioStream = async () => {
    if (!localStream.value) return;

    users.value.forEach(user => {
      if (!user.conn) return;

      try {
        console.log('Вызываем пользователя:', user.id);
        const call = peer.value?.call(user.id, localStream.value!);
        
        if (call) {
          call.on('stream', (remoteStream: MediaStream) => {
            console.log('Получен ответный стрим от:', user.id);
            addRemoteStream(user.id, remoteStream);
          });

          call.on('error', (err) => {
            console.error('Ошибка при звонке пользователю:', user.id, err);
          });
        }
      } catch (err) {
        console.error('Ошибка при установке аудио стрима:', err);
      }
    });
  };

  const addRemoteStream = (userId: string, stream: MediaStream) => {
    console.log('Добавляем удаленный стрим для:', userId);
    remoteStreams.value.set(userId, stream);
    
    const audio = new Audio();
    audio.srcObject = stream;
    audio.autoplay = true; // Добавляем автовоспроизведение
    
    audio.onloadedmetadata = () => {
      console.log('Аудио элемент готов к воспроизведению');
      audio.play().catch(err => console.error('Ошибка воспроизведения:', err));
    };

    audioElements.value.set(userId, audio);
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

    <div
      ref="messagesContainer"
      class="messages"
    >
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
        <span
          v-for="(user, index) in users"
          :key="user.id"
        >
          <span :style="{ color: user.color }">{{
            user.nickname || 'Гость'
          }}</span>
          <span v-if="index < users.length - 1">, </span>
        </span>
      </p>
    </div>

    <!-- Добавляем кнопки управления звонком -->
    <div class="call-controls">
      <button
        v-if="!isInCall && !incomingCall"
        @click="startCall"
        class="call-button"
        title="Начать групповой звонок"
      >
        <svg
          viewBox="0 0 24 24"
          class="call-icon"
        >
          <path
            d="M20.01 15.38c-1.23 0-2.42-.2-3.53-.56-.35-.12-.74-.03-1.01.24l-1.57 1.97c-2.83-1.35-5.48-3.9-6.89-6.83l1.95-1.66c.27-.28.35-.67.24-1.02-.37-1.11-.56-2.3-.56-3.53 0-.54-.45-.99-.99-.99H4.19C3.65 3 3 3.24 3 3.99 3 13.28 10.73 21 20.01 21c.71 0 .99-.63.99-1.18v-3.45c0-.54-.45-.99-.99-.99z"
          />
        </svg>
      </button>
      <button
        v-if="isInCall"
        @click="endCall"
        class="call-button end-call"
        title="Завершить звонок"
      >
        <svg
          viewBox="0 0 24 24"
          class="call-icon"
        >
          <path
            d="M12 9c-1.6 0-3.15.25-4.6.72v3.1c0 .39-.23.74-.56.9-.98.49-1.87 1.12-2.66 1.85-.18.18-.43.28-.7.28-.28 0-.53-.11-.71-.29L.29 13.08c-.18-.17-.29-.42-.29-.7 0-.28.11-.53.29-.71C3.34 8.78 7.46 7 12 7s8.66 1.78 11.71 4.67c.18.18.29.43.29.71 0 .28-.11.53-.29.71l-2.48 2.48c-.18.18-.43.29-.71.29-.27 0-.52-.11-.7-.28-.79-.73-1.68-1.36-2.66-1.85-.33-.16-.56-.5-.56-.9v-3.1C15.15 9.25 13.6 9 12 9z"
          />
        </svg>
      </button>
    </div>

    <!-- Добавляем модальное окно входящего звонка -->
    <div
      v-if="incomingCall"
      class="incoming-call-modal"
    >
      <div class="modal-content">
        <p>Входящий групповой звонок</p>
        <div class="call-actions">
          <button
            @click="acceptCall"
            class="accept-call"
            >✅ Принять</button
          >
          <button
            @click="declineCall"
            class="decline-call"
            >❌ Отклонить</button
          >
        </div>
      </div>
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
    overflow-y: auto;
    padding: 15px;
    display: flex;
    flex-direction: column;
    gap: 8px;
    scroll-behavior: smooth; /* Плавный скролл */
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

  .connection-status span {
    font-weight: 500;
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

  .call-controls {
    position: fixed;
    bottom: 30px;
    right: 30px;
    z-index: 1000;
  }

  .call-button {
    width: 56px;
    height: 56px;
    border-radius: 50%;
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: var(--call-button-bg);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    transition: all 0.3s ease;
  }

  .call-button:hover {
    transform: scale(1.1);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
  }

  .call-button:active {
    transform: scale(0.95);
  }

  .call-icon {
    width: 24px;
    height: 24px;
    fill: white;
  }

  .end-call {
    background-color: var(--end-call-bg);
  }

  .incoming-call-modal {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
  }

  .modal-content {
    background-color: var(--surface-color);
    padding: 20px;
    border-radius: 8px;
    text-align: center;
  }

  .call-actions {
    display: flex;
    gap: 10px;
    justify-content: center;
    margin-top: 15px;
  }

  .accept-call {
    background-color: var(--accept-call-bg);
  }

  .decline-call {
    background-color: var(--decline-call-bg);
  }

  /* Добавляем переменные для цветов */
  .chat-container {
    --call-button-bg: #42b883;
    --end-call-bg: #dc3545;
    --accept-call-bg: #42b883;
    --decline-call-bg: #ff4444;
  }

  .chat-container.dark {
    --call-button-bg: #2c7a57;
    --end-call-bg: #bb2d3b;
    --accept-call-bg: #2c7a57;
    --decline-call-bg: #cc0000;
  }

  /* Добавляем стили для скроллбара */
  .messages::-webkit-scrollbar {
    width: 8px;
  }

  .messages::-webkit-scrollbar-track {
    background: var(--scroll-track-bg, rgba(0, 0, 0, 0.1));
    border-radius: 4px;
  }

  .messages::-webkit-scrollbar-thumb {
    background: var(--scroll-thumb-bg, rgba(0, 0, 0, 0.2));
    border-radius: 4px;
  }

  .messages::-webkit-scrollbar-thumb:hover {
    background: var(--scroll-thumb-hover-bg, rgba(0, 0, 0, 0.3));
  }

  .chat-container.dark {
    --scroll-track-bg: rgba(255, 255, 255, 0.1);
    --scroll-thumb-bg: rgba(255, 255, 255, 0.2);
    --scroll-thumb-hover-bg: rgba(255, 255, 255, 0.3);
  }
</style>
