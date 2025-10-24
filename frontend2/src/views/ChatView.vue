<template>
  <div class="chat-container">
    <!-- Header -->
    <Motion :initial="{ opacity: 0, y: -20 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.6 }">
      <div class="chat-header">
        <div class="chat-title">
          <i class="bi bi-pencil-square"></i>
          <h3>NMSU-TutoringBot-MeatScience</h3>
        </div>
        <button class="menu-toggle-btn" @click="toggleSidebar">
          <i class="bi bi-list"></i>
        </button>
      </div>
    </Motion>

    <!-- Bot Info -->
    <Motion :initial="{ opacity: 0, x: -30 }" :animate="{ opacity: 1, x: 0 }" :transition="{ duration: 0.5, delay: 0.2 }">
      <div class="bot-info justify-between">
        <div>
          <h2>IntelligentTutor-MeatScience</h2>
          <p class="bot-status">User: {{ username }}</p>
          <p class="bot-status">Game: {{ gameNumber }} · Question: {{ userMessageCount }}/20</p>
        </div>
      </div>
    </Motion>

    <!-- Messages -->
    <div ref="chatMessages" class="chat-messages" v-if="messages.length > 0 || isLoading">
      <!-- Conversation -->
      <Motion
        v-for="(message, index) in messages"
        :key="index"
        :initial="{ opacity: 0, y: 20, scale: 0.95 }"
        :animate="{ opacity: 1, y: 0, scale: 1 }"
        :transition="{ duration: 0.5, delay: 0.0 * index, type: 'spring', stiffness: 200, damping: 20 }"
      >
        <div class="message-container" :class="message.sender">
          <div class="message-bubble" :class="message.sender">
            <!-- Avatar -->
            <div v-if="message.sender === 'user'" class="user-avatar"><i class="bi bi-person"></i></div>
            <div v-else class="bot-avatar"><div class="bot-logo"><span class="bot-logo-text">NMSU</span></div></div>

            <!-- Message -->
            <div class="message-content">
              <div class="message-header">
                <span class="sender-name">{{ message.sender === "user" ? "You" : "NMSU" }}</span>
              </div>
              <div class="message-text" v-html="message.text"></div>
            </div>
          </div>
        </div>
      </Motion>

      <!-- Typing Indicator (bubble completo del bot) -->
      <Motion
        v-if="isLoading"
        :initial="{ opacity: 0, y: 20, scale: 0.9 }"
        :animate="{ opacity: 1, y: 0, scale: 1 }"
        :transition="{ duration: 0.5 }"
      >
        <div class="message-container bot">
          <div class="message-bubble bot loading">
            <div class="bot-avatar">
              <div class="bot-logo"><span class="bot-logo-text">NMSU</span></div>
            </div>
            <div class="message-content">
              <div class="message-header">
                <span class="sender-name">NMSU</span>
                <span class="bot-status">Typing...</span>
              </div>
              <div class="message-text">
                <Motion :animate="{ opacity: [0.3, 1, 0.3] }" :transition="{ duration: 1.5, repeat: Infinity }">
                  <div class="typing-indicator">
                    <span class="typing-dot"></span><span class="typing-dot"></span><span class="typing-dot"></span>
                  </div>
                </Motion>
              </div>
            </div>
          </div>
        </div>
      </Motion>
    </div>

    <!-- Welcome -->
    <Motion v-else :initial="{ opacity: 0, y: 30 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.8 }">
      <div class="chat-welcome">
        <h3 class="welcome-message">How can I help you today?</h3>
      </div>
    </Motion>

    <!-- Input -->
    <Motion :initial="{ opacity: 0, y: 20 }" :animate="{ opacity: 1, y: 0 }" :transition="{ duration: 0.6, delay: 0.4 }">
      <div class="chat-input-container">
        <div class="input-wrapper">
          <input
            type="text"
            placeholder="Send a message"
            class="message-input"
            v-model="messageText"
            @keyup.enter="sendMessage"
          />
          <button class="send-button" @click="sendMessage">
            <i class="bi bi-send mt-1"></i>
          </button>
        </div>
        <p class="disclaimer">LLMs can make mistakes. Verify important information.</p>
      </div>
    </Motion>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from "vue";
import { Motion } from "motion-v";
import { useRouter } from "vue-router";
import { useAuthStore } from "../store/auth";
import { useGameStore } from "../store/gameStore";
import axios from "axios";
import { storeToRefs } from "pinia";

const router = useRouter();
const authStore = useAuthStore();
const gameStore = useGameStore();

const { userMessageCount, score, gameNumber, highestScore } = storeToRefs(gameStore);
const username = ref("");
const messages = ref([]);
const messageText = ref("");
const isLoading = ref(false);
const chatMessages = ref(null);

/* --- Toggle sidebar --- */

const toggleSidebar = () => {
  const sidebar = document.querySelector(".sidebar");
  const mainContent = document.querySelector(".main-content");

  if (sidebar && mainContent) {
    sidebar.classList.toggle("isOpen");
    mainContent.classList.toggle("hide");
  }
};

/* --- Scroll automático --- */
const scrollToBottom = () => {
  nextTick(() => {
    if (chatMessages.value) {
      chatMessages.value.scrollTo({ top: chatMessages.value.scrollHeight, behavior: "smooth" });
    }
  });
};
watch([messages, isLoading], () => setTimeout(scrollToBottom, 300), { deep: true });

/* --- Cargar o crear nuevo juego para el usuario actual --- */
const loadUserData = async () => {
  if (!username.value) return;

  try {
    const { data } = await axios.get(`http://localhost:8000/user/get_stats/${username.value}`);

    // Buscar el último juego de ESTE usuario
    let newGameNumber = 1;
    if (data.games && data.games.length > 0) {
      const lastGame = data.games[data.games.length - 1];
      newGameNumber = lastGame.game_number + 1; // sumamos 1 al último
    }

    // Reiniciar progreso y asignar nuevo juego
    gameStore.gameNumber = newGameNumber;
    gameStore.userMessageCount = 0;
    gameStore.score = 0;
    gameStore.highestScore = data.best_score || 0;

    // Guardar el nuevo juego en la BD (ligado al usuario actual)
    await axios.post("http://localhost:8000/user/update_game", {
      username: username.value,
      game_number: gameStore.gameNumber,
      question_number: 0,
      correct_count: 0,
      highest_score: gameStore.highestScore,
    });

    // Mostrar mensaje inicial
    messages.value = [
      {
        id: Date.now(),
        text: `🎮 Welcome back, ${username.value}! Starting new Game #${gameStore.gameNumber}.`,
        sender: "bot",
      },
    ];

    scrollToBottom();
  } catch (error) {
    console.error("Error loading user data:", error);
  }
};




/* --- Enviar mensaje --- */
const sendMessage = async () => {
  if (!messageText.value.trim()) return;

  const text = messageText.value.trim();
  messageText.value = "";

  // Mostrar de inmediato el mensaje del usuario
  messages.value.push({ id: Date.now(), text, sender: "user" });
  scrollToBottom();
  isLoading.value = true;

  // 1) Calcula el siguiente número de pregunta SIN mutar todavía
  const nextCount = gameStore.userMessageCount + 1;

  // 2) Si supera 20, inicia juego nuevo y NO guardes este mensaje
  if (nextCount > 20) {
    gameStore.gameNumber++;
    gameStore.userMessageCount = 0; // ← clave: reseteamos para que la próxima sea #1

    // limpiar pantalla y anunciar nuevo juego
    messages.value = [
      { id: Date.now(), text: `🎮 New Game #${gameStore.gameNumber} started!`, sender: "bot" },
    ];

    // crear registro del nuevo juego en BD con question_number=0
    try {
      await axios.post("http://localhost:8000/user/update_game", {
        username: username.value,
        game_number: gameStore.gameNumber,
        question_number: 0,
        correct_count: 0,
        highest_score: gameStore.highestScore,
      });
    } catch (e) {
      console.error(e);
      messages.value.push({
        id: Date.now() + 2,
        text: "⚠️ Error creating new game in DB.",
        sender: "bot",
      });
    } finally {
      isLoading.value = false;
      scrollToBottom();
    }
    return; // ← muy importante
  }

  // 3) No hay rollover: ahora sí confirmamos el incremento y guardamos
  gameStore.userMessageCount = nextCount;

  try {
    const { data } = await axios.post("http://localhost:8000/user/save_message", {
      username: username.value,
      text,
      game_number: gameStore.gameNumber,
      question_number: gameStore.userMessageCount, // será 1 en la primera del juego nuevo
    });

    // animación de tipeo
    await new Promise((r) => setTimeout(r, 1000));

    messages.value.push({
      id: Date.now() + 1,
      text: data.bot_response,
      sender: "bot",
    });
    scrollToBottom();

    // ✅ Persistir progreso del juego actualizado (nuevo paso clave)
    // esto sincroniza la BD con el número actual de pregunta real
    await axios.post("http://localhost:8000/user/update_game", {
      username: username.value,
      game_number: gameStore.gameNumber,
      question_number: gameStore.userMessageCount,
      correct_count: gameStore.score,
      highest_score: gameStore.highestScore,
    });

  } catch (error) {
    console.error(error);
    messages.value.push({
      id: Date.now() + 2,
      text: "Error contacting API or DB.",
      sender: "bot",
    });
  } finally {
    isLoading.value = false;
    scrollToBottom();
  }
};



/* --- Inicialización --- */
onMounted(async () => {
  authStore.initialize();
  username.value = authStore.user?.username || localStorage.getItem("username") || "guest";
  await loadUserData();
});

onMounted(() => {
  window.addEventListener("load-conversation", (event) => {
    const { gameNumber, messages: rawMessages } = event.detail;

    gameStore.gameNumber = gameNumber;
    messages.value = rawMessages.flatMap((m) => [
      { sender: "user", text: m.user_message },
      { sender: "bot", text: m.bot_response },
    ]);
  });
});

/* --- Logout --- */
const logoutUser = () => {
  authStore.logout();
  router.push("/login");
};
</script>


<style scoped src="../styles/chat.css"></style>
<style scoped src="../styles/chatMessages.css"></style>
<style scoped src="../styles/animation.css"></style>

<style scoped>
.typing-indicator {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-top: 8px;
}
.typing-dot {
  width: 8px;
  height: 8px;
  background-color: #8b1538;
  border-radius: 50%;
  margin: 0 2px;
  animation: blink 1.2s infinite ease-in-out both;
}
@keyframes blink {
  0%, 80%, 100% { opacity: 0; }
  40% { opacity: 1; }
}
</style>
