<script setup lang="ts">
  import { ref, onMounted } from 'vue';
  import Peer from 'peerjs';

  const myId = ref('');
  const peerId = ref('');
  const message = ref('');
  const messages = ref<string[]>([]);
  const peer = ref<Peer>();
  const conn = ref<any>(null);

  onMounted(() => {
    peer.value = new Peer();

    peer.value.on('open', (id) => {
      myId.value = id;
    });

    peer.value.on('connection', (connection) => {
      conn.value = connection;
      setupConnection();
    });
  });

  const connect = () => {
    if (peer.value && peerId.value) {
      try {
        conn.value = peer.value.connect(peerId.value, {
          reliable: true,
        });
        setupConnection();
      } catch (err) {
        console.error('Ошибка при подключении:', err);
      }
    }
  };

  const setupConnection = () => {
    if (!conn.value) return;

    conn.value.on('open', () => {
      console.log('Соединение установлено!');
    });

    conn.value.on('data', (data: string) => {
      messages.value.push(`Собеседник: ${data}`);
    });

    conn.value.on('error', (err: any) => {
      console.error('Ошибка соединения:', err);
    });
  };

  const sendMessage = () => {
    if (conn.value && message.value) {
      try {
        conn.value.send(message.value);
        messages.value.push(`Вы: ${message.value}`);
        message.value = '';
      } catch (err) {
        console.error('Ошибка при отправке:', err);
      }
    }
  };
</script>

<template>
  <div class="chat-container">
    <div class="connection-info">
      <p>Ваш ID: {{ myId }}</p>
      <div class="connect-form">
        <input
          v-model="peerId"
          placeholder="Введите ID собеседника"
        />
        <button @click="connect">Подключиться</button>
      </div>
    </div>

    <div class="messages">
      <div
        v-for="(msg, index) in messages"
        :key="index"
      >
        {{ msg }}
      </div>
    </div>

    <div class="message-input">
      <input
        v-model="message"
        @keyup.enter="sendMessage"
        placeholder="Введите сообщение"
      />
      <button @click="sendMessage">Отправить</button>
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
</style>
