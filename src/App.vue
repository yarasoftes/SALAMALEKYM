<script setup lang="ts">
import { open } from '@tauri-apps/plugin-dialog'

import {
  computed,
  nextTick,
  onMounted,
  onUnmounted,
  ref,
  watch,
} from 'vue'

/* =========================================================
   TYPES
========================================================= */

interface Message {
  id: number
  text: string
  sender: 'me' | 'other'
  senderId?: string
  time: string
  image?: string
  imageName?: string
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

interface SavedState {
  messages: Record<number, Message[]>
  chats: Chat[]
  currentUserId?: string
}

/* =========================================================
   STATE
========================================================= */

const DB_NAME = 'yaraasx-messenger'
const DB_VERSION = 1
const DB_STORE = 'state'
const DB_KEY = 'main'

const accounts = [
  { id: 'yaraasx', name: 'Yaraasx', username: '@yaraasx', avatar: 'Y' },
  { id: 'oleg', name: 'Oleg', username: '@oleg', avatar: 'O' },
] as const

const currentUserId = ref<string>('yaraasx')
const accountMenuOpen = ref(false)

let persistenceWatcher: (() => void) | null = null

const search = ref('')
const messageText = ref('')

const currentUser = computed(() =>
  accounts.find(account => account.id === currentUserId.value) ?? accounts[0],
)

const otherUser = computed(() =>
  accounts.find(account => account.id !== currentUserId.value) ?? accounts[1],
)

/* В локальном режиме есть одна общая переписка между двумя аккаунтами. */
const CONVERSATION_ID = 1
const activeChatId = ref(1)

const emojiOpen = ref(false)

const fileInput =
    ref<HTMLInputElement | null>(null)

const selectedFile =
    ref<SelectedFile | null>(null)

/* =========================================================
   CHATS
========================================================= */

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
    name: 'Yaraasx',
    username: '@yaraasx',
    avatar: 'Y',
    online: true,
    lastMessage: 'Это мои сообщения',
    lastTime: '12:40',
    unread: 0,
  },
])

/* =========================================================
   MESSAGES
========================================================= */

const messages =
    ref<Record<number, Message[]>>({
      1: [
        {
          id: 1,
          text: 'Привет!',
          sender: 'other',
          senderId: 'oleg',
          time: '12:39',
        },

        {
          id: 2,
          text: 'Привет! Как дела?',
          sender: 'me',
          senderId: 'yaraasx',
          time: '12:40',
        },

        {
          id: 3,
          text: 'Отлично 😎 А у тебя?',
          sender: 'other',
          senderId: 'oleg',
          time: '12:41',
        },
      ],

      2: [
        {
          id: 4,
          text: 'Это чат с самим собой',
          sender: 'me',
          time: '12:40',
        },
      ],
    })

/* =========================================================
   EMOJI
========================================================= */

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

const visibleEmojis =
    ref<string[]>([])

/* =========================================================
   EMOJI HELPERS
========================================================= */

function shuffle<T>(array: T[]): T[] {
  const result = [...array]

  for (
      let i = result.length - 1;
      i > 0;
      i--
  ) {
    const j = Math.floor(
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

/* =========================================================
   COMPUTED
========================================================= */

const activeChat = computed(() => {
  return chats.value.find(chat => chat.id === activeChatId.value)
})

const activeMessages = computed(() => {
  // В локальном режиме оба аккаунта смотрят на одну и ту же
  // переписку. Для отображения `me/other` ниже используется senderId.
  return messages.value[CONVERSATION_ID] ?? []
})

const filteredChats = computed(() => {
  const value =
      search.value
          .toLowerCase()
          .trim()

  const availableChats = chats.value.filter(
      chat => chat.id !== (currentUserId.value === 'yaraasx' ? 2 : 1),
  )

  if (!value) {
    return availableChats
  }

  return availableChats.filter(
      chat =>
          chat.name.toLowerCase().includes(value) ||
          chat.username.toLowerCase().includes(value),
  )
})

/* =========================================================
   ACCOUNT SWITCH
========================================================= */

function toggleAccountMenu() {
  accountMenuOpen.value = !accountMenuOpen.value
}

async function switchAccount(userId: string) {
  if (userId === currentUserId.value) {
    accountMenuOpen.value = false
    return
  }

  currentUserId.value = userId
  activeChatId.value = otherUser.value.id === 'oleg' ? 1 : 2
  accountMenuOpen.value = false
  emojiOpen.value = false
  selectedFile.value = null
  messageText.value = ''

  if (fileInput.value) {
    fileInput.value.value = ''
  }

  await saveState()

  nextTick(() => scrollMessagesToBottom())
}

/* =========================================================
   CHAT SWITCH
========================================================= */

function selectChat(chatId: number) {
  activeChatId.value = chatId

  const chat = chats.value.find(
      item => item.id === chatId,
  )

  if (chat) {
    chat.unread = 0
  }

  emojiOpen.value = false

  selectedFile.value = null

  messageText.value = ''

  if (fileInput.value) {
    fileInput.value.value = ''
  }

  nextTick(() => {
    scrollMessagesToBottom()
  })
}

/* =========================================================
   SCROLL
========================================================= */

function scrollMessagesToBottom() {
  nextTick(() => {
    const container =
        document.querySelector(
            '.messages',
        ) as HTMLElement | null

    if (!container) {
      return
    }

    container.scrollTop =
        container.scrollHeight
  })
}

/* =========================================================
   EMOJI
========================================================= */

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

/* =========================================================
   FILE PICKER
========================================================= */

async function openFilePicker() {
  /*
   * Сначала пробуем системный Tauri picker.
   */

  try {
    const selected =
        await open({
          multiple: false,

          filters: [
            {
              name: 'Images',

              extensions: [
                'png',
                'jpg',
                'jpeg',
                'webp',
                'gif',
              ],
            },
          ],
        })

    /*
     * Если пользователь выбрал путь,
     * здесь можно подключить чтение
     * файла через Tauri FS.
     *
     * Пока оставляем fallback
     * на обычный input.
     */

    if (selected) {
      fileInput.value?.click()
    }
  } catch {
    /*
     * Если Tauri dialog недоступен,
     * используем обычный input.
     */

    fileInput.value?.click()
  }
}

/* =========================================================
   FILE SELECT
========================================================= */

function handleFileSelected(
    event: Event,
) {
  const input =
      event.target as HTMLInputElement

  const file =
      input.files?.[0]

  if (!file) {
    return
  }

  if (!file.type.startsWith('image/')) {
    input.value = ''
    return
  }

  const reader =
      new FileReader()

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

/* =========================================================
   REMOVE FILE
========================================================= */

function removeSelectedFile() {
  selectedFile.value = null

  if (fileInput.value) {
    fileInput.value.value = ''
  }
}

/* =========================================================
   FILE SIZE
========================================================= */

function formatFileSize(
    bytes: number,
): string {
  if (bytes < 1024) {
    return `${bytes} Б`
  }

  if (
      bytes <
      1024 * 1024
  ) {
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

/* =========================================================
   SEND MESSAGE
========================================================= */

function sendMessage() {
  const text =
      messageText.value.trim()

  if (
      !text &&
      !selectedFile.value
  ) {
    return
  }

  if (!messages.value[CONVERSATION_ID]) {
    messages.value[CONVERSATION_ID] = []
  }

  const now =
      new Date().toLocaleTimeString(
          [],
          {
            hour: '2-digit',
            minute: '2-digit',
          },
      )

  const image =
      selectedFile.value?.dataUrl

  const imageName =
      selectedFile.value?.name

  messages.value[CONVERSATION_ID].push({
    id: Date.now(),

    text,

    sender: 'me',

    senderId: currentUserId.value,

    time: now,

    image,

    imageName,
  })

  const chat =
      chats.value.find(
          item =>
              item.id ===
              activeChatId.value,
      )

  if (chat) {
    chat.lastMessage =
        selectedFile.value
            ? `📎 ${selectedFile.value.name}`
            : text

    chat.lastTime = now
  }

  messageText.value = ''

  removeSelectedFile()

  emojiOpen.value = false

  nextTick(() => {
    scrollMessagesToBottom()
  })

  void saveState()
}

/* =========================================================
   ENTER
========================================================= */

function handleEnter(
    event: KeyboardEvent,
) {
  if (event.shiftKey) {
    return
  }

  event.preventDefault()

  sendMessage()
}

/* =========================================================
   OUTSIDE CLICK
========================================================= */

function handleOutsideClick(
    event: MouseEvent,
) {
  const target =
      event.target as HTMLElement

  if (!target.closest('.emoji-wrapper')) {
    emojiOpen.value = false
  }

  if (!target.closest('.profile')) {
    accountMenuOpen.value = false
  }
}

/* =========================================================
   ESCAPE
========================================================= */

function handleEscape(
    event: KeyboardEvent,
) {
  if (
      event.key === 'Escape'
  ) {
    emojiOpen.value = false
  }
}

/* =========================================================
   INDEXED DB
========================================================= */

function openDatabase(): Promise<IDBDatabase> {
  return new Promise(
      (resolve, reject) => {
        const request =
            indexedDB.open(
                DB_NAME,
                DB_VERSION,
            )

        request.onupgradeneeded =
            () => {
              const db =
                  request.result

              if (
                  !db.objectStoreNames.contains(
                      DB_STORE,
                  )
              ) {
                db.createObjectStore(
                    DB_STORE,
                )
              }
            }

        request.onsuccess = () => {
          resolve(
              request.result,
          )
        }

        request.onerror = () => {
          reject(
              request.error,
          )
        }
      },
  )
}

/* =========================================================
   SAVE STATE
========================================================= */

async function saveState() {
  try {
    const db =
        await openDatabase()

    const transaction =
        db.transaction(
            DB_STORE,
            'readwrite',
        )

    const store =
        transaction.objectStore(
            DB_STORE,
        )

    const state: SavedState = {
      messages:
      messages.value,

      chats:
      chats.value,

      currentUserId: currentUserId.value,
    }

    store.put(
        state,
        DB_KEY,
    )

    await new Promise<void>(
        (
            resolve,
            reject,
        ) => {
          transaction.oncomplete =
              () => {
                resolve()
              }

          transaction.onerror =
              () => {
                reject(
                    transaction.error,
                )
              }

          transaction.onabort =
              () => {
                reject(
                    transaction.error,
                )
              }
        },
    )

    db.close()
  } catch (error) {
    console.error(
        'Ошибка сохранения данных:',
        error,
    )
  }
}

/* =========================================================
   LOAD STATE
========================================================= */

async function loadState() {
  try {
    const db =
        await openDatabase()

    const transaction =
        db.transaction(
            DB_STORE,
            'readonly',
        )

    const store =
        transaction.objectStore(
            DB_STORE,
        )

    const saved =
        await new Promise<
            SavedState | undefined
        >(
            (
                resolve,
                reject,
            ) => {
              const request =
                  store.get(
                      DB_KEY,
                  )

              request.onsuccess =
                  () => {
                    resolve(
                        request.result as
                            | SavedState
                            | undefined,
                    )
                  }

              request.onerror =
                  () => {
                    reject(
                        request.error,
                    )
                  }
            },
        )

    db.close()

    if (!saved) {
      return
    }

    if (saved.currentUserId) {
      currentUserId.value = saved.currentUserId
    }


    if (
        saved.messages
    ) {
      messages.value =
          saved.messages
    }

    if (saved.chats) {
      chats.value = saved.chats
    }

    if (saved.currentUserId === 'yaraasx' || saved.currentUserId === 'oleg') {
      currentUserId.value = saved.currentUserId
    }

    // Миграция старых сообщений: они были сохранены относительно Yaraasx.
    const legacyMessages = messages.value[CONVERSATION_ID] ?? []
    legacyMessages.forEach(message => {
      if (message.senderId == null) {
        message.senderId = message.sender === 'me' ? 'yaraasx' : 'oleg'
      }
    })

    // Сохраняем единственный чат между двумя локальными аккаунтами.
    chats.value = [
      {
        id: 1,
        name: 'Oleg',
        username: '@oleg',
        avatar: 'O',
        online: true,
        lastMessage: messages.value[CONVERSATION_ID]?.at(-1)?.text || 'Начните переписку',
        lastTime: messages.value[CONVERSATION_ID]?.at(-1)?.time || '',
        unread: 0,
      },
      {
        id: 2,
        name: 'Yaraasx',
        username: '@yaraasx',
        avatar: 'Y',
        online: true,
        lastMessage: messages.value[CONVERSATION_ID]?.at(-1)?.text || 'Начните переписку',
        lastTime: messages.value[CONVERSATION_ID]?.at(-1)?.time || '',
        unread: 0,
      },
    ]

    activeChatId.value = currentUserId.value === 'yaraasx' ? 1 : 2
  } catch (error) {
    console.error(
        'Ошибка загрузки данных:',
        error,
    )
  }
}

/* =========================================================
   MOUNT
========================================================= */

onMounted(
    async () => {
      generateRandomEmojis()

      await loadState()

      persistenceWatcher =
          watch(
              [messages, chats],
              () => {
                void saveState()
              },
              {
                deep: true,
              },
          )

      try {
        if (
            navigator.storage &&
            navigator.storage.persist
        ) {
          await navigator.storage.persist()
        }
      } catch {
        /*
         * Ничего страшного.
         */
      }

      document.addEventListener(
          'click',
          handleOutsideClick,
      )

      document.addEventListener(
          'keydown',
          handleEscape,
      )

      await nextTick()

      scrollMessagesToBottom()
    },
)

/* =========================================================
   UNMOUNT
========================================================= */

onUnmounted(() => {
  if (
      persistenceWatcher
  ) {
    persistenceWatcher()

    persistenceWatcher =
        null
  }

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

<template>
  <div class="app">

    <!-- =====================================================
         SIDEBAR
    ====================================================== -->

    <aside class="sidebar">

      <div class="sidebar-header">

        <div class="brand">

          <div class="brand-logo">
            Y
          </div>

          <span>
            YARAASX
          </span>

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
              chat.id ===
              activeChatId,
          }"
            @click="
            selectChat(chat.id)
          "
        >

          <div class="avatar">

            {{ chat.avatar }}

            <span
                v-if="chat.online"
                class="online-dot"
            />

          </div>

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

              <span
                  class="last-message"
              >
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

        <div
            v-if="
            filteredChats.length === 0
          "
            class="empty-search"
        >
          Ничего не найдено
        </div>

      </div>

      <!-- PROFILE / ACCOUNT SWITCHER -->

      <div class="profile">

        <button
            class="profile-main"
            type="button"
            @click.stop="toggleAccountMenu"
        >

          <div class="profile-avatar">
            {{ currentUser.avatar }}
            <span class="profile-online-dot" />
          </div>

          <div class="profile-info">
            <strong>{{ currentUser.name }}</strong>
            <span>{{ currentUser.username }} · online</span>
          </div>

        </button>

        <button
            class="profile-button"
            type="button"
            title="Сменить аккаунт"
            @click.stop="toggleAccountMenu"
        >
          ⇅
        </button>

        <div v-if="accountMenuOpen" class="account-switcher">
          <div class="account-switcher-title">Переключить аккаунт</div>

          <button
              v-for="user in accounts"
              :key="user.id"
              type="button"
              class="account-option"
              :class="{ active: user.id === currentUserId }"
              @click.stop="switchAccount(user.id)"
          >
            <span class="account-option-avatar">{{ user.avatar }}</span>
            <span class="account-option-info">
              <strong>{{ user.name }}</strong>
              <small>{{ user.username }}</small>
            </span>
            <span v-if="user.id === currentUserId" class="account-check">✓</span>
          </button>
        </div>

      </div>

    </aside>

    <!-- =====================================================
         CHAT
    ====================================================== -->

    <main class="chat">

      <!-- HEADER -->

      <header
          v-if="activeChat"
          class="chat-header"
      >

        <div class="header-avatar">

          {{ activeChat.avatar }}

          <span
              v-if="
              activeChat.online
            "
              class="online-dot"
          />

        </div>

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

      <!-- MESSAGES -->

      <section
          class="messages"
      >

        <div
            class="
            messages-background
          "
        />

        <div
            class="
            message-container
          "
        >

          <div
              v-for="
              message in activeMessages
            "
              :key="message.id"
              class="message-row"
              :class="{
              mine:
                (message.senderId ?? (message.sender === 'me' ? 'yaraasx' : 'oleg')) ===
                currentUserId,
            }"
          >

            <div class="message">

              <!-- IMAGE -->

              <img
                  v-if="
                  message.image
                "
                  class="
                  message-image
                "
                  :src="
                  message.image
                "
                  :alt="
                  message.imageName ||
                  'Изображение'
                "
              />

              <!-- TEXT -->

              <div
                  v-if="
                  message.text
                "
                  class="
                  message-text
                "
              >
                {{ message.text }}
              </div>

              <!-- META -->

              <div
                  class="
                  message-meta
                "
              >

                <span>
                  {{ message.time }}
                </span>

                <span
                    v-if="
                    (message.senderId ?? (message.sender === 'me' ? 'yaraasx' : 'oleg')) ===
                    currentUserId
                  "
                    class="checks"
                >
                  ✓✓
                </span>

              </div>

            </div>

          </div>

        </div>

      </section>

      <!-- ===================================================
           COMPOSER
      ==================================================== -->

      <footer
          class="composer"
      >

        <!-- EMOJI -->

        <div
            class="emoji-wrapper"
        >

          <button
              class="
              composer-button
            "
              title="Emoji"
              @click.stop="
              toggleEmojiPicker()
            "
          >
            😊
          </button>

          <div
              v-if="emojiOpen"
              class="
              emoji-panel
            "
              @click.stop
          >

            <div
                class="
                emoji-header
              "
            >

              <strong>
                Emoji
              </strong>

              <span>
                Популярные
              </span>

            </div>

            <div
                class="
                emoji-grid
              "
            >

              <button
                  v-for="(
                  emoji,
                  index
                ) in visibleEmojis"
                  :key="
                  `${emoji}-${index}`
                "
                  class="
                  emoji-item
                "
                  @click="
                  addEmoji(emoji)
                "
              >
                {{ emoji }}
              </button>

            </div>

            <div
                class="
                emoji-footer
              "
            >
              При следующем запуске
              набор изменится
            </div>

          </div>

        </div>

        <!-- FILE INPUT -->

        <input
            ref="fileInput"
            class="
            hidden-file-input
          "
            type="file"
            accept="image/*"
            @change="
            handleFileSelected
          "
        />

        <!-- SELECTED FILE -->

        <div
            v-if="selectedFile"
            class="
            selected-file
          "
        >

          <div
              class="
              selected-file-preview
            "
          >

            <img
                :src="
                selectedFile.dataUrl
              "
                :alt="
                selectedFile.name
              "
            />

          </div>

          <div
              class="
              selected-file-info
            "
          >

            <strong>
              {{
                selectedFile.name
              }}
            </strong>

            <span>
              {{
                formatFileSize(
                    selectedFile.size,
                )
              }}
            </span>

          </div>

          <button
              class="
              remove-file
            "
              title="Удалить"
              @click="
              removeSelectedFile()
            "
          >
            ×
          </button>

        </div>

        <!-- TEXT -->

        <textarea
            v-model="messageText"
            placeholder="
            Написать сообщение...
          "
            rows="1"
            @keydown.enter="
            handleEnter
          "
        />

        <!-- ATTACHMENT -->

        <button
            class="
            composer-button
          "
            title="
            Прикрепить изображение
          "
            @click="
            openFilePicker
          "
        >
          📎
        </button>

        <!-- SEND -->

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
            @click="
            sendMessage
          "
        >
          ➤
        </button>

      </footer>

    </main>

  </div>
</template>

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

/* =========================================================
   APP
========================================================= */

.app {
  width: 100%;
  height: 100vh;

  display: flex;

  background: #0e1014;
}

/* =========================================================
   SIDEBAR
========================================================= */

.sidebar {
  width: 340px;
  min-width: 340px;
  height: 100%;

  display: flex;
  flex-direction: column;

  background: #15171c;

  border-right:
      1px solid #292c33;
}

.sidebar-header {
  height: 70px;

  display: flex;
  align-items: center;
  justify-content: space-between;

  padding: 0 18px;

  border-bottom:
      1px solid #25282e;
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

  background:
      linear-gradient(
          135deg,
          #367cff,
          #6d42ff
      );

  box-shadow:
      0 5px 20px
      rgba(
          60,
          100,
          255,
          0.3
      );

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

/* =========================================================
   SEARCH
========================================================= */

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

/* =========================================================
   CHAT LIST
========================================================= */

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

/* =========================================================
   AVATARS
========================================================= */

.avatar,
.header-avatar,
.profile-avatar {
  position: relative;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;

  background:
      linear-gradient(
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

  border:
      2px solid #15171c;

  border-radius: 50%;

  background: #32d583;
}

.chat-item.active
.online-dot {
  border-color: #2f6fe4;
}

/* =========================================================
   CHAT INFO
========================================================= */

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

.chat-item.active
.chat-time {
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

.chat-item.active
.last-message {
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

/* =========================================================
   PROFILE
========================================================= */

.profile {
  height: 72px;

  display: flex;
  align-items: center;

  gap: 11px;

  padding: 10px 14px;

  border-top:
      1px solid #292c33;
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

.profile-avatar {
  width: 42px;
  height: 42px;

  background:
      linear-gradient(
          135deg,
          #13b5ea,
          #246bfe
      );

  transition:
      transform 0.15s;
}

.profile-main:hover
.profile-avatar {
  transform: scale(1.04);
}

.profile-online-dot {
  position: absolute;

  right: 0;
  bottom: 0;

  width: 12px;
  height: 12px;

  border:
      2px solid #15171c;

  border-radius: 50%;

  background: #32d583;
}

.profile-info {
  min-width: 0;

  flex: 1;

  display: flex;
  flex-direction: column;

  gap: 3px;
}

.profile-info strong {
  font-size: 14px;
}

.profile-info span {
  color: #32d583;

  font-size: 12px;
}

.profile-button {
  width: 36px;
  height: 36px;

  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

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

/* =========================================================
   CHAT
========================================================= */

.chat {
  min-width: 0;

  height: 100%;

  flex: 1;

  display: flex;
  flex-direction: column;

  background: #101216;
}

/* =========================================================
   HEADER
========================================================= */

.chat-header {
  height: 70px;

  display: flex;
  align-items: center;

  padding: 0 18px;

  border-bottom:
      1px solid #292c33;

  background: #17191e;
}

.header-avatar {
  width: 44px;
  height: 44px;

  margin-right: 12px;
}

.header-avatar
.online-dot {
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

/* =========================================================
   MESSAGES
========================================================= */

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
  max-width:
      min(600px, 70%);

  padding: 9px 12px 7px;

  border-radius:
      14px 14px 14px 4px;

  background: #24272e;

  box-shadow:
      0 2px 5px
      rgba(
          0,
          0,
          0,
          0.15
      );
}

.message-row.mine
.message {
  border-radius:
      14px 14px 4px 14px;

  background: #2d6fdb;
}

.message-text {
  color: #f5f7fa;

  font-size: 14px;

  line-height: 1.45;

  white-space: pre-wrap;

  overflow-wrap: anywhere;
}

/* =========================================================
   MESSAGE IMAGE
========================================================= */

.message-image {
  display: block;

  width: auto;

  max-width: 420px;

  max-height: 420px;

  margin-bottom: 7px;

  border-radius: 11px;

  object-fit: contain;

  background: #15171c;
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

.message-row.mine
.message-meta {
  color: #c9dcff;
}

.checks {
  font-size: 11px;
}

/* =========================================================
   COMPOSER
========================================================= */

.composer {
  position: relative;

  min-height: 70px;

  display: flex;
  align-items: flex-end;

  gap: 8px;

  padding: 12px 18px;

  background: #17191e;

  border-top:
      1px solid #292c33;
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
      0 4px 15px
      rgba(
          50,
          121,
          230,
          0.25
      );
}

.send-button.ready:hover {
  background: #4287ed;
}

.send-button:disabled {
  cursor: default;
}

/* =========================================================
   EMOJI
========================================================= */

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

  border:
      1px solid #353942;

  border-radius: 16px;

  background: #1b1e24;

  box-shadow:
      0 15px 45px
      rgba(
          0,
          0,
          0,
          0.55
      );

  z-index: 100;
}

.emoji-header {
  display: flex;
  align-items: center;
  justify-content: space-between;

  padding: 2px 4px 12px;

  border-bottom:
      1px solid #2c2f36;
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

/* =========================================================
   FILE
========================================================= */

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

  border:
      1px solid #343943;

  border-radius: 12px;

  background: #20232a;

  box-shadow:
      0 8px 25px
      rgba(
          0,
          0,
          0,
          0.35
      );

  z-index: 50;
}

.selected-file-preview {
  width: 45px;
  height: 45px;

  flex-shrink: 0;

  overflow: hidden;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 9px;

  background: #2f6fe4;
}

.selected-file-preview img {
  width: 100%;
  height: 100%;

  object-fit: cover;
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

/* =========================================================
   SCROLLBARS
========================================================= */

.messages::-webkit-scrollbar {
  width: 6px;
}

.messages::-webkit-scrollbar-thumb {
  background: #343840;

  border-radius: 10px;
}


.account-switcher {
  position: absolute;
  bottom: 76px;
  left: 12px;
  right: 12px;
  padding: 8px;
  background: #20232a;
  border: 1px solid #343840;
  border-radius: 14px;
  z-index: 20;
  box-shadow: 0 12px 30px rgba(0,0,0,.28);
}

.profile { position: relative; }
.account-switcher-title { padding: 8px 10px; color: #9da3b0; font-size: 12px; }
.account-option { display:flex; align-items:center; gap:10px; width:100%; border:0; background:transparent; color:#fff; padding:10px; border-radius:9px; cursor:pointer; text-align:left; }
.account-option:hover, .account-option.active { background:#343840; }
.account-option-avatar { display:grid; place-items:center; width:30px; height:30px; border-radius:50%; background:#5865f2; font-weight:700; }
.account-check { margin-left:auto; color:#7ee2a8; }

/* =========================================================
   RESPONSIVE
========================================================= */

@media (max-width: 750px) {
  .sidebar {
    width: 290px;
    min-width: 290px;
  }

  .message {
    max-width: 80%;
  }

  .message-image {
    max-width: 300px;
  }

  .emoji-panel {
    width: 300px;
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

  .message {
    max-width: 88%;
  }

  .message-image {
    max-width: 240px;
  }

  .emoji-panel {
    left: -5px;

    width: 290px;
  }
}
</style>
