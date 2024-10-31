<template>
  <div class="q-pa-md row justify-center">
    <div style="width: 100%; max-width: 100%">
      <!-- Contenedor con scroll y ref para controlarlo -->
      <div ref="chatContainer" style="max-height: 100%; overflow-y: auto;">
        <div v-for="(message, index) in messages" :key="index">
          <q-chat-message :text="[message.text]"
                          :sent="message.send"
                          :text-color="message.send ? 'white' : 'black'"
                          :bg-color="message.send ? 'primary' : 'amber'">
            <template v-slot:name>
              {{ message.send ? 'Yo' : 'Maria' }}
            </template>
            <template v-slot:avatar>
              <img class="q-message-avatar"
                   :class="message.send ? 'q-message-avatar--sent' : 'q-message-avatar--received'"
                   :src="message.send ? 'https://cdn.quasar.dev/img/avatar4.jpg' : 'https://cdn.quasar.dev/img/avatar2.jpg'" />
            </template>
          </q-chat-message>
        </div>

        <q-chat-message v-if="isTyping" bg-color="amber">
          <template v-slot:name>
            Maria
          </template>
          <template v-slot:avatar>
            <img class="q-message-avatar q-message-avatar--received"
                 src="https://cdn.quasar.dev/img/avatar2.jpg" />
          </template>
          <q-spinner-dots size="2rem" />
        </q-chat-message>
      </div>

      <!-- Campo de entrada del mensaje -->
      <q-input rounded outlined v-model="inputValue" @keyup.enter="sendMessage">
        <template v-slot:append>
          <q-btn dense flat icon="send" @click="sendMessage" />
        </template>
      </q-input>
    </div>
  </div>
</template>


<script setup lang="ts">
  import { ref, onMounted, nextTick, watch } from 'vue';

  // Define la interfaz para el tipo de mensaje que esperamos recibir y enviar
  interface Chat {
    text: string;
    send: boolean;
    time: string;
    hilo?: string; // Usamos string para representar el hilo opcionalmente
  }

  const inputValue = ref('');
  const messages = ref<Chat[]>([

  ]);

  const hilo = ref<string | null>(null);
  const isTyping = ref(false);
  const chatContainer = ref<HTMLElement | null>(null)


  async function sendMessage() {
    if (!inputValue.value.trim()) return;

    // Agregar el mensaje enviado a la lista de mensajes
    messages.value.push({
      text: inputValue.value,
      send: true,
      time: new Date().toLocaleTimeString(),
    });
    
    // Guardar el mensaje en una variable y limpiar el input
    const userMessage = inputValue.value;
    inputValue.value = '';

    // Mostrar indicador de "escribiendo"
    isTyping.value = true;
    nextTick(() => {
      scrollToBottom();
    });
    try {
      // Preparar el último mensaje en el formato esperado por la API
      const lastMessage: Chat = {
        text: userMessage,
        send: true,
        time: '',
        hilo: hilo.value ?? '',
      };

      // Realizar el request a la API con el último mensaje
      const response = await fetch('https://localhost:7098/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(lastMessage),
      });

      if (!response.ok) {
        throw new Error('Error al obtener la respuesta de la API');
      }

      // Parsear la respuesta como un objeto de tipo Chat
      const data: Chat = await response.json();

      // Agregar la respuesta de la API al historial de mensajes
      messages.value.push({
        text: data.text,
        send: data.send,
        time: new Date(data.time).toLocaleTimeString(),
      });

      // Actualizar el hilo con el valor que viene desde el backend
      hilo.value = data.hilo ?? hilo.value;
      nextTick(() => {
        scrollToBottom();
      });
    } catch (error) {
      console.error('Error al obtener la respuesta de la API:', error);
      messages.value.push({
        text: 'Error al obtener la respuesta de la API',
        send: false,
        time: new Date().toLocaleTimeString(),
      });
    } finally {
      // Ocultar indicador de "escribiendo"
    
      isTyping.value = false;
      nextTick(() => {
        scrollToBottom();
      });
    }
  }

  // Función para desplazar el scroll hacia abajo
  function scrollToBottom() {
    if (chatContainer.value) {
      chatContainer.value.scrollTop = chatContainer.value.scrollHeight;
    }
  }

  // Observar cambios en messages para actualizar el scroll
  watch(messages, () => {
    nextTick(() => {
      scrollToBottom();
    });
  });

  onMounted(() => {
    scrollToBottom();
  });
</script>

<style scoped>
  .q-message-avatar--sent {
    /* Opcional: Personaliza el estilo del avatar de mensajes enviados */
  }

  .q-message-avatar--received {
    /* Opcional: Personaliza el estilo del avatar de mensajes recibidos */
  }

  .chat-container {
    max-height: 500px;
    overflow-y: auto;
  }
</style>
