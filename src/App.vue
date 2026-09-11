<script setup lang="ts">
import {
  computed,
  nextTick,
  onMounted,
  onUnmounted,
  ref,
} from 'vue'

interface Message {
  id: number
  text: string
  sender: 'me' | 'other'
  time: string

  // Картинка сообщения
  image?: string
  imageName?: string

  // Пользователь, которому принадлежит сообщение
  userId?: number
}

interface Chat {
  id: number
  name: string
  username: string
  avatar: string
  online: boolean
  lastMessage: string
  lastTime: string
  unread: number
}

interface SelectedFile {
  name: string
  size: number
  type: string
  dataUrl: string
}

interface User {
  id: number
  name: string
  username: string
  avatar: string
  online: boolean
}

const search = ref('')
const messageText = ref('')
const activeChatId = ref(1)

/*
|--------------------------------------------------------------------------
| ПОЛЬЗОВАТЕЛИ
|--------------------------------------------------------------------------
*/

const users = ref<User[]>([
  {
    id: 1,
    name: 'Yaraasx',
    username: '@yaraasx',
    avatar: 'Y',
    online: true,
  },
  {
    id: 2,
    name: 'Yaraasx 2',
    username: '@yaraasx2',
    avatar: 'Y2',
    online: true,
  },
])

const activeUserId = ref(1)

const userSwitcherOpen = ref(false)

/*
|--------------------------------------------------------------------------
| EMOJI / FILE
|--------------------------------------------------------------------------
*/

const emojiOpen = ref(false)

const fileInput =
    ref<HTMLInputElement | null>(null)

const selectedFile =
    ref<SelectedFile | null>(null)

/*
|--------------------------------------------------------------------------
| CHATS
|--------------------------------------------------------------------------
*/

const chats = ref<Chat[]>([
  {
    id: 1,
    name: 'Oleg',
    username: '@oleg',
    avatar: 'O',
    online: true,
    lastMessage: 'Привет! Как дела?',
    lastTime: '12:41',
    unread: 2,
  },
  {
    id: 2,
    name: 'Kirill',
    username: '@kirill',
    avatar: 'K',
    online: false,
    lastMessage: 'До встречи!',
    lastTime: 'Вчера',
    unread: 0,
  },
])

/*
|--------------------------------------------------------------------------
| MESSAGES
|--------------------------------------------------------------------------
|
| userId = 1 — сообщения первого пользователя.
| Старые сообщения без userId автоматически считаются сообщениями
| первого пользователя.
|
*/

const messages = ref<Record<number, Message[]>>({
  1: [
    {
      id: 1,
      text: 'Привет!',
      sender: 'other',
      time: '12:39',
      userId: 1,
    },
    {
      id: 2,
      text: 'Привет! Как дела?',
      sender: 'me',
      time: '12:40',
      userId: 1,
    },
    {
      id: 3,
      text: 'Отлично 😎 А у тебя?',
      sender: 'other',
      time: '12:41',
      userId: 1,
    },
  ],

  2: [
    {
      id: 4,
      text: 'Привет, Kirill!',
      sender: 'me',
      time: 'Вчера',
      userId: 1,
    },
    {
      id: 5,
      text: 'Привет! До встречи!',
      sender: 'other',
      time: 'Вчера',
      userId: 1,
    },
  ],
})

/*
|--------------------------------------------------------------------------
| EMOJI
|--------------------------------------------------------------------------
*/

const allEmojis = [
  '😀', '😃', '😄', '😁', '😆', '😅', '😂', '🤣',
  '😊', '😇', '🙂', '🙃', '😉', '😌', '😍', '🥰',
  '😘', '😗', '😙', '😚', '😋', '😛', '😝', '😜',
  '🤪', '🤨', '🧐', '🤓', '😎', '🥳', '🤩', '🥺',
  '😏', '😒', '😞', '😔', '😟', '😕', '🙁', '☹️',
  '😣', '😖', '😫', '😩', '🥹', '😢', '😭', '😤',
  '😠', '😡', '🤬', '🤯', '😳', '🥵', '🥶', '😱',
  '😨', '😰', '😥', '😓', '🫣', '🤗', '🫡', '🤔',
  '🫢', '🤭', '🤫', '🤥', '😶', '😐', '😑', '😬',
  '🙄', '😯', '😦', '😧', '😮', '😲', '🥱', '😴',
  '🤤', '😪', '😵', '🤐', '🥴', '🤢', '🤮', '🤧',
  '😷', '🤒', '🤕', '🤑', '🤠', '😈', '👿', '👹',
  '👺', '🤡', '💩', '👻', '💀', '☠️', '👽', '👾',
  '🤖', '🎃', '😺', '😸', '😹', '😻', '😼', '😽',
  '🙀', '😿', '😾',

  '👍', '👎', '👌', '✌️', '🤞', '🤟', '🤘', '🤙',
  '👈', '👉', '👆', '👇', '☝️', '✋', '🤚', '🖐️',
  '🖖', '👏', '🙌', '👐', '🤝', '🙏', '💪', '🫶',

  '❤️', '🧡', '💛', '💚', '💙', '💜', '🖤', '🤍',
  '🤎', '💔', '❣️', '💕', '💞', '💓', '💗', '💖',
  '💘', '💝', '💟', '🔥', '✨', '⭐', '🌟', '💫',
  '🎉', '🎊', '💯', '✅', '❌', '⚡', '💥',

  '🐶', '🐱', '🐭', '🐹', '🐰', '🦊', '🐻', '🐼',
  '🐨', '🐯', '🦁', '🐮', '🐷', '🐸', '🐵', '🙈',
  '🙉', '🙊', '🐔', '🐧', '🐦', '🦄', '🐝', '🦋',

  '🍎', '🍊', '🍋', '🍉', '🍇', '🍓', '🍒', '🍑',
  '🍍', '🥝', '🍌', '🍕', '🍔', '🍟', '🌭', '🌮',
  '🍿', '🍩', '🍪', '🎂', '🍰', '☕', '🥤', '🍺',

  '⚽', '🏀', '🏈', '⚾', '🎾', '🏆', '🎮', '🎯',
  '🎸', '🎧', '🎬', '🚗', '✈️', '🚀', '🏠', '💻',
]

const visibleEmojis = ref<string[]>([])

/*
|--------------------------------------------------------------------------
| ACTIVE USER
|--------------------------------------------------------------------------
*/

const activeUser = computed(() => {
  return (
      users.value.find(
          user =>
              user.id === activeUserId.value,
      ) ?? users.value[0]
  )
})

/*
|--------------------------------------------------------------------------
| ПЕРЕКЛЮЧЕНИЕ ПОЛЬЗОВАТЕЛЯ
|--------------------------------------------------------------------------
*/

function toggleUserSwitcher() {
  userSwitcherOpen.value =
      !userSwitcherOpen.value
}

function switchUser(userId: number) {
  activeUserId.value = userId

  userSwitcherOpen.value = false

  messageText.value = ''

  selectedFile.value = null

  emojiOpen.value = false

  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

/*
|--------------------------------------------------------------------------
| ПЕРЕМЕШИВАНИЕ EMOJI
|--------------------------------------------------------------------------
*/

function shuffle<T>(array: T[]): T[] {
  const result = [...array]

  for (
      let i = result.length - 1;
      i > 0;
      i--
  ) {
    const j =
        Math.floor(
            Math.random() * (i + 1),
        )

    ;[result[i], result[j]] = [
      result[j],
      result[i],
    ]
  }

  return result
}

function generateRandomEmojis() {
  visibleEmojis.value =
      shuffle(allEmojis).slice(0, 42)
}

/*
|--------------------------------------------------------------------------
| COMPUTED
|--------------------------------------------------------------------------
*/

const activeChat = computed(() => {
  return chats.value.find(
      chat =>
          chat.id === activeChatId.value,
  )
})

const activeMessages = computed(() => {
  const chatMessages =
      messages.value[
          activeChatId.value
          ] ?? []

  return chatMessages.filter(
      message =>
          (message.userId ?? 1) ===
          activeUserId.value,
  )
})

const filteredChats = computed(() => {
  const value =
      search.value
          .toLowerCase()
          .trim()

  if (!value) {
    return chats.value
  }

  return chats.value.filter(
      chat =>
          chat.name
              .toLowerCase()
              .includes(value) ||
          chat.username
              .toLowerCase()
              .includes(value),
  )
})

/*
|--------------------------------------------------------------------------
| CHAT
|--------------------------------------------------------------------------
*/

function selectChat(chatId: number) {
  activeChatId.value = chatId

  const chat = chats.value.find(
      item => item.id === chatId,
  )

  if (chat) {
    chat.unread = 0
  }
}

/*
|--------------------------------------------------------------------------
| EMOJI
|--------------------------------------------------------------------------
*/

function toggleEmojiPicker() {
  emojiOpen.value =
      !emojiOpen.value
}

function addEmoji(emoji: string) {
  messageText.value += emoji

  nextTick(() => {
    const textarea =
        document.querySelector(
            '.composer textarea',
        ) as HTMLTextAreaElement | null

    textarea?.focus()
  })
}

/*
|--------------------------------------------------------------------------
| FILE
|--------------------------------------------------------------------------
|
| ВАЖНО:
| Раньше здесь использовался Tauri open().
| Он открывал окно выбора файла, но выбранный файл
| не передавался в handleFileSelected().
|
| Теперь используем обычный hidden input.
|
*/

function openFilePicker() {
  fileInput.value?.click()
}

function handleFileSelected(
    event: Event,
) {
  const input =
      event.target as HTMLInputElement

  const file = input.files?.[0]

  if (!file) {
    return
  }

  if (!file.type.startsWith('image/')) {
    input.value = ''

    return
  }

  const reader = new FileReader()

  reader.onload = () => {
    selectedFile.value = {
      name: file.name,
      size: file.size,
      type: file.type,
      dataUrl: String(
          reader.result,
      ),
    }
  }

  reader.readAsDataURL(file)
}

function removeSelectedFile() {
  selectedFile.value = null

  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

function formatFileSize(
    bytes: number,
): string {
  if (bytes < 1024) {
    return `${bytes} Б`
  }

  if (bytes < 1024 * 1024) {
    return `${(
        bytes / 1024
    ).toFixed(1)} КБ`
  }

  return `${(
      bytes /
      1024 /
      1024
  ).toFixed(1)} МБ`
}

/*
|--------------------------------------------------------------------------
| SEND MESSAGE
|--------------------------------------------------------------------------
*/

function sendMessage() {
  const text =
      messageText.value.trim()

  if (
      !text &&
      !selectedFile.value
  ) {
    return
  }

  if (
      !messages.value[
          activeChatId.value
          ]
  ) {
    messages.value[
        activeChatId.value
        ] = []
  }

  const file =
      selectedFile.value

  messages.value[
      activeChatId.value
      ].push({
    id: Date.now(),

    text,

    sender: 'me',

    time: new Date().toLocaleTimeString(
        [],
        {
          hour: '2-digit',
          minute: '2-digit',
        },
    ),

    userId:
    activeUserId.value,

    image:
    file?.dataUrl,

    imageName:
    file?.name,
  })

  const chat = chats.value.find(
      item =>
          item.id ===
          activeChatId.value,
  )

  if (chat) {
    if (file) {
      chat.lastMessage = text
          ? `🖼️ ${text}`
          : `🖼️ ${file.name}`
    } else {
      chat.lastMessage = text
    }

    chat.lastTime = 'Сейчас'
  }

  messageText.value = ''

  removeSelectedFile()

  emojiOpen.value = false
}

/*
|--------------------------------------------------------------------------
| ENTER
|--------------------------------------------------------------------------
*/

function handleEnter(
    event: KeyboardEvent,
) {
  if (event.shiftKey) {
    return
  }

  event.preventDefault()

  sendMessage()
}

/*
|--------------------------------------------------------------------------
| OUTSIDE CLICK
|--------------------------------------------------------------------------
*/

function handleOutsideClick(
    event: MouseEvent,
) {
  const target =
      event.target as HTMLElement

  if (
      !target.closest(
          '.emoji-wrapper',
      )
  ) {
    emojiOpen.value = false
  }

  if (
      !target.closest(
          '.profile',
      )
  ) {
    userSwitcherOpen.value = false
  }
}

function handleEscape(
    event: KeyboardEvent,
) {
  if (event.key === 'Escape') {
    emojiOpen.value = false

    userSwitcherOpen.value = false
  }
}

/*
|--------------------------------------------------------------------------
| LIFECYCLE
|--------------------------------------------------------------------------
*/

onMounted(() => {
  generateRandomEmojis()

  document.addEventListener(
      'click',
      handleOutsideClick,
  )

  document.addEventListener(
      'keydown',
      handleEscape,
  )
})

onUnmounted(() => {
  document.removeEventListener(
      'click',
      handleOutsideClick,
  )

  document.removeEventListener(
      'keydown',
      handleEscape,
  )
})
</script>

```vue
<template>
  <div class="app">

    <!-- =========================
         SIDEBAR
    ========================== -->

    <aside class="sidebar">

      <!-- HEADER -->
      <div class="sidebar-header">
        <div class="brand">
          <div class="brand-logo">
            Y
          </div>

          <span>YARAASX</span>
        </div>

        <button
            class="icon-button"
            title="Новое сообщение"
        >
          +
        </button>
      </div>

      <!-- SEARCH -->
      <div class="search-container">
        <div class="search-box">

          <span class="search-icon">
            ⌕
          </span>

          <input
              v-model="search"
              type="text"
              placeholder="Поиск"
          />

          <span
              v-if="search"
              class="clear-search"
              @click="search = ''"
          >
            ×
          </span>

        </div>
      </div>

      <!-- CHAT LIST -->
      <div class="chat-list">

        <button
            v-for="chat in filteredChats"
            :key="chat.id"
            class="chat-item"
            :class="{
              active:
                chat.id === activeChatId,
            }"
            @click="selectChat(chat.id)"
        >

          <!-- AVATAR -->
          <div class="avatar">

            {{ chat.avatar }}

            <span
                v-if="chat.online"
                class="online-dot"
            />

          </div>

          <!-- CHAT INFO -->
          <div class="chat-info">

            <div class="chat-top">

              <span class="chat-name">
                {{ chat.name }}
              </span>

              <span class="chat-time">
                {{ chat.lastTime }}
              </span>

            </div>

            <div class="chat-bottom">

              <span class="last-message">
                {{ chat.lastMessage }}
              </span>

              <span
                  v-if="chat.unread"
                  class="unread"
              >
                {{ chat.unread }}
              </span>

            </div>

          </div>

        </button>

        <!-- EMPTY SEARCH -->
        <div
            v-if="filteredChats.length === 0"
            class="empty-search"
        >
          Ничего не найдено
        </div>

      </div>


      <!-- =========================
           PROFILE / USER SWITCHER
      ========================== -->

      <div class="profile">

        <!-- CURRENT USER -->
        <button
            class="profile-main"
            @click="toggleUserSwitcher"
        >

          <div class="profile-avatar">
            {{ activeUser.avatar }}
          </div>

          <div class="profile-info">

            <strong>
              {{ activeUser.name }}
            </strong>

            <span>
              {{
                activeUser.online
                    ? 'online'
                    : 'offline'
              }}
            </span>

          </div>

        </button>


        <!-- THREE DOTS -->
        <button
            class="profile-button"
            title="Переключить пользователя"
            @click="toggleUserSwitcher"
        >
          ⋮
        </button>


        <!-- USER SWITCHER -->
        <div
            v-if="userSwitcherOpen"
            class="user-switcher"
            @click.stop
        >

          <div class="user-switcher-title">
            Переключить пользователя
          </div>


          <!-- USERS -->
          <button
              v-for="user in users"
              :key="user.id"
              class="user-switcher-item"
              :class="{
                active:
                  user.id === activeUserId,
              }"
              @click="switchUser(user.id)"
          >

            <div class="user-switcher-avatar">
              {{ user.avatar }}
            </div>

            <div class="user-switcher-info">

              <strong>
                {{ user.name }}
              </strong>

              <span>
                {{ user.username }}
              </span>

            </div>

            <span
                v-if="
                  user.id === activeUserId
                "
                class="user-check"
            >
              ✓
            </span>

          </button>


          <!-- ADD USER -->
          <button
              class="add-user-button"
              @click.stop
          >
            ＋ Добавить пользователя
          </button>

        </div>

      </div>

    </aside>


    <!-- =========================
         MAIN CHAT
    ========================== -->

    <main class="chat">

      <!-- =========================
           CHAT HEADER
      ========================== -->

      <header
          v-if="activeChat"
          class="chat-header"
      >

        <!-- AVATAR -->
        <div class="header-avatar">

          {{ activeChat.avatar }}

          <span
              v-if="activeChat.online"
              class="online-dot"
          />

        </div>


        <!-- INFO -->
        <div class="header-info">

          <strong>
            {{ activeChat.name }}
          </strong>

          <span>
            {{
              activeChat.online
                  ? 'в сети'
                  : 'был(а) недавно'
            }}
          </span>

        </div>


        <!-- ACTIONS -->
        <div class="header-actions">

          <button
              class="header-button"
              title="Поиск"
          >
            ⌕
          </button>

          <button
              class="header-button"
              title="Меню"
          >
            ⋮
          </button>

        </div>

      </header>


      <!-- =========================
           MESSAGES
      ========================== -->

      <section class="messages">

        <div class="messages-background" />


        <div class="message-container">

          <!-- MESSAGE -->
          <div
              v-for="message in activeMessages"
              :key="message.id"
              class="message-row"
              :class="{
                mine:
                  message.sender === 'me',
              }"
          >

            <div class="message">

              <!-- IMAGE -->
              <img
                  v-if="message.image"
                  class="message-image"
                  :src="message.image"
                  :alt="
                    message.imageName ||
                    'Изображение'
                  "
              />


              <!-- TEXT -->
              <div
                  v-if="message.text"
                  class="message-text"
              >
                {{ message.text }}
              </div>


              <!-- META -->
              <div class="message-meta">

                <span>
                  {{ message.time }}
                </span>

                <span
                    v-if="
                      message.sender === 'me'
                    "
                    class="checks"
                >
                  ✓✓
                </span>

              </div>

            </div>

          </div>


          <!-- NO MESSAGES -->
          <div
              v-if="activeMessages.length === 0"
              class="empty-messages"
          >
            Нет сообщений
          </div>

        </div>

      </section>


      <!-- =========================
           COMPOSER
      ========================== -->

      <footer class="composer">


        <!-- =====================
             EMOJI
        ====================== -->

        <div class="emoji-wrapper">

          <button
              class="composer-button"
              title="Emoji"
              @click.stop="
                toggleEmojiPicker()
              "
          >
            😊
          </button>


          <!-- EMOJI PANEL -->
          <div
              v-if="emojiOpen"
              class="emoji-panel"
              @click.stop
          >

            <div class="emoji-header">

              <strong>
                Emoji
              </strong>

              <span>
                Популярные
              </span>

            </div>


            <div class="emoji-grid">

              <button
                  v-for="(
                    emoji, index
                  ) in visibleEmojis"
                  :key="
                    `${emoji}-${index}`
                  "
                  class="emoji-item"
                  @click="
                    addEmoji(emoji)
                  "
              >
                {{ emoji }}
              </button>

            </div>


            <div class="emoji-footer">
              При следующем запуске
              набор изменится
            </div>

          </div>

        </div>


        <!-- =====================
             FILE INPUT
        ====================== -->

        <input
            ref="fileInput"
            class="hidden-file-input"
            type="file"
            accept="
              image/png,
              image/jpeg,
              image/webp
            "
            @change="
              handleFileSelected
            "
        />


        <!-- =====================
             SELECTED FILE
        ====================== -->

        <div
            v-if="selectedFile"
            class="selected-file"
        >

          <!-- PREVIEW -->
          <img
              v-if="
                selectedFile.dataUrl
              "
              class="selected-file-preview"
              :src="
                selectedFile.dataUrl
              "
              alt="Предпросмотр"
          />


          <!-- FILE ICON -->
          <div
              v-else
              class="selected-file-icon"
          >
            📎
          </div>


          <!-- FILE INFO -->
          <div class="selected-file-info">

            <strong>
              {{ selectedFile.name }}
            </strong>

            <span>
              {{
                formatFileSize(
                    selectedFile.size,
                )
              }}
            </span>

          </div>


          <!-- REMOVE -->
          <button
              class="remove-file"
              title="Удалить"
              @click="
                removeSelectedFile()
              "
          >
            ×
          </button>

        </div>


        <!-- =====================
             TEXT INPUT
        ====================== -->

        <textarea
            v-model="messageText"
            placeholder="Написать сообщение..."
            rows="1"
            @keydown.enter="
              handleEnter
            "
        />


        <!-- =====================
             ATTACHMENT
        ====================== -->

        <button
            class="composer-button"
            title="Прикрепить изображение"
            @click="openFilePicker"
        >
          📎
        </button>


        <!-- =====================
             SEND
        ====================== -->

        <button
            class="send-button"
            :class="{
              ready:
                messageText.trim() ||
                selectedFile,
            }"
            :disabled="
              !messageText.trim() &&
              !selectedFile
            "
            @click="sendMessage"
        >
          ➤
        </button>

      </footer>

    </main>

  </div>
</template>

```css
<style>
* {
  box-sizing: border-box;
}

html,
body,
#app {
  width: 100%;
  height: 100%;
  margin: 0;
}

body {
  font-family:
      Inter,
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      sans-serif;

  background: #0e1014;
  color: #ffffff;

  overflow: hidden;
}

button,
input,
textarea {
  font: inherit;
}

button {
  border: 0;
}

/* =========================
   APP
========================= */

.app {
  width: 100%;
  height: 100vh;

  display: flex;

  background: #0e1014;
}

/* =========================
   SIDEBAR
========================= */

.sidebar {
  width: 340px;
  min-width: 340px;
  height: 100%;

  display: flex;
  flex-direction: column;

  background: #15171c;
  border-right: 1px solid #292c33;
}

.sidebar-header {
  height: 70px;

  display: flex;
  align-items: center;
  justify-content: space-between;

  padding: 0 18px;

  border-bottom: 1px solid #25282e;
}

.brand {
  display: flex;
  align-items: center;
  gap: 11px;

  font-size: 17px;
  font-weight: 800;
  letter-spacing: 1.5px;
}

.brand-logo {
  width: 38px;
  height: 38px;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 12px;

  background: linear-gradient(
      135deg,
      #367cff,
      #6d42ff
  );

  box-shadow:
      0 5px 20px rgba(60, 100, 255, 0.3);

  font-size: 18px;
  font-weight: 800;
}

.icon-button {
  width: 36px;
  height: 36px;

  border-radius: 10px;

  background: transparent;
  color: #89909c;

  font-size: 25px;

  cursor: pointer;

  transition: 0.15s;
}

.icon-button:hover {
  background: #24272e;
  color: white;
}

/* =========================
   SEARCH
========================= */

.search-container {
  padding: 14px 14px 10px;
}

.search-box {
  height: 42px;

  display: flex;
  align-items: center;

  gap: 9px;

  padding: 0 12px;

  border-radius: 11px;

  background: #22252b;
}

.search-icon {
  color: #858c98;
  font-size: 23px;
}

.search-box input {
  flex: 1;

  width: 100%;

  border: 0;
  outline: 0;

  background: transparent;

  color: white;
  font-size: 14px;
}

.search-box input::placeholder {
  color: #777e8a;
}

.clear-search {
  color: #8c929d;
  font-size: 20px;

  cursor: pointer;
}

/* =========================
   CHAT LIST
========================= */

.chat-list {
  flex: 1;

  overflow-y: auto;

  padding: 4px 8px;
}

.chat-list::-webkit-scrollbar {
  width: 5px;
}

.chat-list::-webkit-scrollbar-thumb {
  background: #343840;
  border-radius: 10px;
}

.chat-item {
  width: 100%;

  display: flex;
  align-items: center;

  gap: 12px;

  padding: 10px;

  border-radius: 12px;

  background: transparent;
  color: white;

  text-align: left;

  cursor: pointer;

  transition: 0.15s;
}

.chat-item:hover {
  background: #202329;
}

.chat-item.active {
  background: #2f6fe4;
}

.avatar,
.header-avatar,
.profile-avatar {
  position: relative;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;

  background: linear-gradient(
      135deg,
      #3c82f6,
      #7647ff
  );

  color: white;

  font-weight: 700;
}

.avatar {
  width: 52px;
  height: 52px;

  font-size: 18px;
}

.online-dot {
  position: absolute;

  right: 1px;
  bottom: 1px;

  width: 12px;
  height: 12px;

  border: 2px solid #15171c;

  border-radius: 50%;

  background: #32d583;
}

.chat-item.active .online-dot {
  border-color: #2f6fe4;
}

.chat-info {
  min-width: 0;
  flex: 1;
}

.chat-top,
.chat-bottom {
  display: flex;
  align-items: center;
}

.chat-top {
  justify-content: space-between;
  gap: 8px;

  margin-bottom: 4px;
}

.chat-name {
  font-size: 15px;
  font-weight: 650;
}

.chat-time {
  flex-shrink: 0;

  color: #777f8b;

  font-size: 11px;
}

.chat-item.active .chat-time {
  color: #dbe7ff;
}

.chat-bottom {
  justify-content: space-between;
  gap: 8px;
}

.last-message {
  min-width: 0;

  overflow: hidden;

  color: #858c97;

  font-size: 13px;

  text-overflow: ellipsis;
  white-space: nowrap;
}

.chat-item.active .last-message {
  color: #e1eaff;
}

.unread {
  min-width: 20px;
  height: 20px;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;

  background: #3984ff;

  color: white;

  font-size: 11px;
  font-weight: 700;
}

.empty-search {
  padding: 35px 10px;

  color: #777e8a;

  text-align: center;
  font-size: 14px;
}



.profile {
  position: relative;

  height: 72px;

  display: flex;
  align-items: center;

  gap: 11px;

  padding: 10px 14px;

  border-top: 1px solid #292c33;
}

.profile-main {
  min-width: 0;

  flex: 1;

  display: flex;
  align-items: center;

  gap: 11px;

  padding: 0;

  background: transparent;
  color: white;

  text-align: left;

  cursor: pointer;
}

.profile-main:hover {
  opacity: 0.9;
}

.profile-avatar {
  width: 42px;
  height: 42px;

  background: linear-gradient(
      135deg,
      #13b5ea,
      #246bfe
  );
}

.profile-info {
  min-width: 0;

  flex: 1;

  display: flex;
  flex-direction: column;

  gap: 3px;
}

.profile-info strong {
  overflow: hidden;

  font-size: 14px;

  text-overflow: ellipsis;
  white-space: nowrap;
}

.profile-info span {
  color: #32d583;

  font-size: 12px;
}

.profile-button {
  width: 32px;
  height: 36px;

  flex-shrink: 0;

  border-radius: 9px;

  background: transparent;

  color: #7e8591;

  font-size: 21px;

  cursor: pointer;
}

.profile-button:hover {
  background: #24272e;
  color: white;
}

/* =========================
   USER SWITCHER
========================= */

.user-switcher {
  position: absolute;

  left: 10px;
  bottom: 68px;

  width: 280px;

  padding: 8px;

  border: 1px solid #353942;

  border-radius: 14px;

  background: #1b1e24;

  box-shadow:
      0 15px 45px rgba(0, 0, 0, 0.55);

  z-index: 200;

  animation:
      userSwitcherOpen
      0.15s
      ease-out;
}

.user-switcher-title {
  padding: 8px 10px 10px;

  color: #777e8a;

  font-size: 11px;
  font-weight: 600;

  text-transform: uppercase;
}

.user-switcher-item {
  width: 100%;

  display: flex;
  align-items: center;

  gap: 10px;

  padding: 9px;

  border-radius: 10px;

  background: transparent;
  color: white;

  text-align: left;

  cursor: pointer;

  transition: 0.15s;
}

.user-switcher-item:hover {
  background: #25282e;
}

.user-switcher-item.active {
  background: #2f6fe4;
}

.user-switcher-avatar {
  width: 38px;
  height: 38px;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;

  background: linear-gradient(
      135deg,
      #3c82f6,
      #7647ff
  );

  color: white;

  font-size: 13px;
  font-weight: 700;
}

.user-switcher-info {
  min-width: 0;

  flex: 1;

  display: flex;
  flex-direction: column;

  gap: 2px;
}

.user-switcher-info strong {
  overflow: hidden;

  font-size: 13px;

  text-overflow: ellipsis;
  white-space: nowrap;
}

.user-switcher-info span {
  overflow: hidden;

  color: #8d95a2;

  font-size: 11px;

  text-overflow: ellipsis;
  white-space: nowrap;
}

.user-switcher-item.active
.user-switcher-info span {
  color: #dbe7ff;
}

.user-check {
  flex-shrink: 0;

  color: white;

  font-size: 17px;
  font-weight: 700;
}

.add-user-button {
  width: 100%;

  margin-top: 5px;

  padding: 10px;

  border-top: 1px solid #30343c;
  border-radius: 9px;

  background: transparent;

  color: #7eaaff;

  text-align: left;

  cursor: pointer;

  transition: 0.15s;
}

.add-user-button:hover {
  background: #25282e;
}

@keyframes userSwitcherOpen {
  from {
    opacity: 0;
    transform: translateY(8px);
  }

  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* =========================
   CHAT
========================= */

.chat {
  min-width: 0;
  height: 100%;

  flex: 1;

  display: flex;
  flex-direction: column;

  background: #101216;
}

/* =========================
   HEADER
========================= */

.chat-header {
  height: 70px;

  display: flex;
  align-items: center;

  padding: 0 18px;

  border-bottom: 1px solid #292c33;

  background: #17191e;
}

.header-avatar {
  width: 44px;
  height: 44px;

  margin-right: 12px;
}

.header-avatar .online-dot {
  border-color: #17191e;
}

.header-info {
  display: flex;
  flex-direction: column;

  gap: 4px;
}

.header-info strong {
  font-size: 15px;
}

.header-info span {
  color: #32d583;

  font-size: 12px;
}

.header-actions {
  margin-left: auto;

  display: flex;

  gap: 4px;
}

.header-button {
  width: 40px;
  height: 40px;

  border-radius: 10px;

  background: transparent;

  color: #8a919d;

  font-size: 22px;

  cursor: pointer;

  transition: 0.15s;
}

.header-button:hover {
  background: #25282e;
  color: white;
}

/* =========================
   MESSAGES
========================= */

.messages {
  position: relative;

  flex: 1;

  overflow-y: auto;
}

.messages-background {
  position: absolute;

  inset: 0;

  opacity: 0.25;

  background-image:
      radial-gradient(
          circle at 20% 30%,
          #2a3d65 0,
          transparent 25%
      ),
      radial-gradient(
          circle at 80% 70%,
          #332653 0,
          transparent 25%
      );
}

.message-container {
  position: relative;

  min-height: 100%;

  display: flex;
  flex-direction: column;
  justify-content: flex-end;

  padding: 25px 7%;
}

.message-row {
  display: flex;

  margin: 4px 0;
}

.message-row.mine {
  justify-content: flex-end;
}

.message {
  max-width: min(600px, 70%);

  padding: 9px 12px 7px;

  border-radius: 14px 14px 14px 4px;

  background: #24272e;

  box-shadow:
      0 2px 5px rgba(0, 0, 0, 0.15);
}

.message-row.mine .message {
  border-radius: 14px 14px 4px 14px;

  background: #2d6fdb;
}

.message-text {
  color: #f5f7fa;

  font-size: 14px;
  line-height: 1.45;

  white-space: pre-wrap;
  overflow-wrap: anywhere;
}

/* =========================
   MESSAGE IMAGE
========================= */

.message-image {
  display: block;

  width: 100%;
  max-width: 420px;
  max-height: 420px;

  margin: 0 0 7px;

  border-radius: 10px;

  object-fit: cover;

  background: #15171c;

  cursor: pointer;

  transition: opacity 0.15s;
}

.message-image:hover {
  opacity: 0.95;
}

.message-meta {
  display: flex;
  align-items: center;
  justify-content: flex-end;

  gap: 4px;

  margin-top: 3px;

  color: #8b929d;

  font-size: 10px;
}

.message-row.mine .message-meta {
  color: #c9dcff;
}

.checks {
  font-size: 11px;
}

.empty-messages {
  display: flex;
  align-items: center;
  justify-content: center;

  min-height: 200px;

  color: #777e8a;

  font-size: 14px;
}

/* =========================
   COMPOSER
========================= */

.composer {
  position: relative;

  min-height: 70px;

  display: flex;
  align-items: flex-end;

  gap: 8px;

  padding: 12px 18px;

  background: #17191e;

  border-top: 1px solid #292c33;
}

.composer textarea {
  flex: 1;

  min-height: 44px;
  max-height: 130px;

  padding: 12px 14px;

  resize: none;

  border: 0;
  outline: 0;

  border-radius: 13px;

  background: #24272e;

  color: white;

  font-size: 14px;
  line-height: 20px;
}

.composer textarea::placeholder {
  color: #777e8a;
}

.composer-button,
.send-button {
  width: 44px;
  height: 44px;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 12px;

  background: transparent;

  color: #8c939f;

  font-size: 20px;

  cursor: pointer;

  transition: 0.15s;
}

.composer-button:hover {
  background: #25282e;
  color: white;
}

.send-button {
  background: #24272e;

  color: #555b66;

  font-size: 18px;
}

.send-button.ready {
  background: #3279e6;

  color: white;

  box-shadow:
      0 4px 15px rgba(50, 121, 230, 0.25);
}

.send-button.ready:hover {
  background: #4287ed;
}

.send-button:disabled {
  cursor: default;
}

/* =========================
   EMOJI
========================= */

.emoji-wrapper {
  position: relative;
}

.emoji-panel {
  position: absolute;

  left: 0;
  bottom: 56px;

  width: 330px;
  height: 350px;

  padding: 14px;

  border: 1px solid #353942;

  border-radius: 16px;

  background: #1b1e24;

  box-shadow:
      0 15px 45px rgba(0, 0, 0, 0.55);

  z-index: 100;
}

.emoji-header {
  display: flex;
  align-items: center;
  justify-content: space-between;

  padding: 2px 4px 12px;

  border-bottom: 1px solid #2c2f36;
}

.emoji-header strong {
  font-size: 14px;
}

.emoji-header span {
  color: #777f8b;

  font-size: 11px;
}

.emoji-grid {
  height: 270px;

  display: grid;

  grid-template-columns:
    repeat(7, 1fr);

  align-content: start;

  gap: 4px;

  padding-top: 10px;

  overflow-y: auto;
}

.emoji-grid::-webkit-scrollbar {
  width: 5px;
}

.emoji-grid::-webkit-scrollbar-thumb {
  background: #3b3f48;

  border-radius: 10px;
}

.emoji-item {
  width: 38px;
  height: 38px;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 9px;

  background: transparent;

  font-size: 23px;

  cursor: pointer;

  transition:
      background 0.12s,
      transform 0.12s;
}

.emoji-item:hover {
  background: #2b2f37;

  transform: scale(1.12);
}

.emoji-footer {
  padding-top: 8px;

  color: #656c78;

  text-align: center;

  font-size: 10px;
}

/* =========================
   FILE
========================= */

.hidden-file-input {
  display: none;
}

.selected-file {
  position: absolute;

  left: 18px;
  right: 18px;
  bottom: 78px;

  display: flex;
  align-items: center;

  gap: 10px;

  padding: 9px 12px;

  border: 1px solid #343943;

  border-radius: 12px;

  background: #20232a;

  box-shadow:
      0 8px 25px rgba(0, 0, 0, 0.35);

  z-index: 50;
}

.selected-file-icon {
  width: 35px;
  height: 35px;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 9px;

  background: #2f6fe4;
}

.selected-file-preview {
  width: 48px;
  height: 48px;

  flex-shrink: 0;

  display: block;

  border-radius: 8px;

  object-fit: cover;

  background: #15171c;
}

.selected-file-info {
  min-width: 0;

  flex: 1;

  display: flex;
  flex-direction: column;

  gap: 2px;
}

.selected-file-info strong {
  overflow: hidden;

  color: #f2f4f7;

  font-size: 12px;

  text-overflow: ellipsis;
  white-space: nowrap;
}

.selected-file-info span {
  color: #7d8591;

  font-size: 10px;
}

.remove-file {
  width: 30px;
  height: 30px;

  flex-shrink: 0;

  border-radius: 8px;

  background: transparent;

  color: #8b929e;

  font-size: 20px;

  cursor: pointer;
}

.remove-file:hover {
  background: #30343c;

  color: white;
}

/* =========================
   SCROLLBARS
========================= */

.messages::-webkit-scrollbar {
  width: 6px;
}

.messages::-webkit-scrollbar-thumb {
  background: #343840;

  border-radius: 10px;
}

/* =========================
   RESPONSIVE
========================= */

@media (max-width: 750px) {
  .sidebar {
    width: 290px;
    min-width: 290px;
  }

  .message {
    max-width: 80%;
  }

  .emoji-panel {
    width: 300px;
  }

  .user-switcher {
    width: 270px;
  }
}

@media (max-width: 600px) {
  .sidebar {
    width: 90px;
    min-width: 90px;
  }

  .sidebar-header {
    justify-content: center;
  }

  .brand span,
  .search-container,
  .chat-info,
  .profile-info,
  .profile-button {
    display: none;
  }

  .chat-item {
    justify-content: center;

    padding: 10px 4px;
  }

  .profile {
    justify-content: center;
  }

  .profile-main {
    justify-content: center;
  }

  .message {
    max-width: 88%;
  }

  .message-image {
    max-width: 100%;
    max-height: 300px;
  }

  .emoji-panel {
    left: -5px;

    width: 290px;
  }

  .user-switcher {
    left: 5px;
    bottom: 65px;

    width: 270px;
  }

  .selected-file-preview {
    width: 42px;
    height: 42px;
  }
}
</style>

