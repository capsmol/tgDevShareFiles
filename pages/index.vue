<template>
  <div class="container mx-auto p-6 max-w-2xl">
    <div class="bg-white rounded-lg shadow-md p-8">
      <div class="space-y-6">
        <div class="grid grid-cols-2 gap-4">
          <div class="space-y-2">
            <h3 class="text-lg font-medium text-gray-700">Prod bot key</h3>
            <UInput 
              v-model="prodBotKey"
              class="w-full"
              placeholder="Введите ключ production бота"
            />
          </div>

          <div class="space-y-2">
            <h3 class="text-lg font-medium text-gray-700">Dev bot key</h3>
            <UInput 
              v-model="devBotKey"
              class="w-full"
              placeholder="Введите ключ development бота"
            />
          </div>
        </div>

        <div class="space-y-2">
          <h3 class="text-lg font-medium text-gray-700">Chat ID</h3>
          <UInput 
            v-model="chatId"
            class="w-full"
            placeholder="Введите ID чата"
          />
        </div>

        <div class="pt-4 border-t">
          <div class="flex items-center gap-4">
            <input 
              type="file" 
              class="block w-full text-sm text-gray-500
                file:mr-4 file:py-2 file:px-4
                file:rounded-md file:border-0
                file:text-sm file:font-semibold
                file:bg-primary-50 file:text-primary-700
                hover:file:bg-primary-100"
              @change="handleFileChange"
              accept="image/*"
            />
            <UButton 
              color="primary"
              class="px-6"
              :loading="isLoading"
              @click="sendFile"
            >
              Get ids
            </UButton>
          </div>
        </div>
      </div>

      <div v-if="uploadResults.length" class="mt-8 space-y-6">
        <h3 class="text-xl font-semibold text-gray-800">Результаты загрузки</h3>
        
        <div v-for="(result, index) in uploadResults" :key="index" class="border rounded-lg p-4">
          <div class="flex items-center gap-2 mb-3">
            <span class="font-medium text-gray-700">Бот:</span>
            <span class="text-black">@{{ result.result.from.username }}</span>
          </div>
          
          <div class="space-y-4">
            <div class="grid grid-cols-2 gap-4 text-sm">
              <div>
                <span class="font-medium text-gray-700">Message ID:</span>
                <span class="ml-2 text-black">{{ result.result.message_id }}</span>
              </div>
              <div>
                <span class="font-medium text-gray-700">Дата:</span>
                <span class="ml-2 text-black">{{ new Date(result.result.date * 1000).toLocaleString() }}</span>
              </div>
            </div>

            <div class="space-y-2">
              <div class="font-medium text-gray-700">Доступные размеры файла:</div>
              <div class="grid grid-cols-2 gap-3">
                <div v-for="photo in result.result.photo" 
                     :key="photo.file_unique_id"
                     class="text-sm p-2 bg-gray-50 rounded text-black">
                  <div>Размер: {{ photo.width }}x{{ photo.height }}</div>
                  <div>Размер файла: {{ (photo.file_size / 1024).toFixed(1) }} KB</div>
                  <div class="mt-2">
                    <div class="font-medium mb-1">FILE_ID:</div>
                    <div class="flex gap-2 items-center">
                      <input 
                        type="text" 
                        :value="photo.file_id" 
                        readonly
                        class="flex-1 text-xs p-1 bg-white border rounded focus:outline-none focus:border-primary-500"
                      />
                      <UButton
                        size="xs"
                        color="gray"
                        icon="i-heroicons-clipboard"
                        square
                        @click="copyToClipboard(photo.file_id)"
                        title="Копировать в буфер обмена"
                      />
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'

interface TelegramPhoto {
  file_id: string
  file_unique_id: string
  file_size: number
  width: number
  height: number
}

interface TelegramResponse {
  ok: boolean
  result: {
    message_id: number
    from: {
      id: number
      is_bot: boolean
      first_name: string
      username: string
    }
    chat: {
      id: number
      first_name: string
      last_name: string
      username: string
      type: string
    }
    date: number
    photo: TelegramPhoto[]
  }
}

const STORAGE_KEYS = {
  PROD_BOT_KEY: 'prodBotKey',
  DEV_BOT_KEY: 'devBotKey',
  CHAT_ID: 'chatId'
} as const

const prodBotKey = ref('')
const devBotKey = ref('')
const chatId = ref('')
const selectedFile = ref<File | null>(null)
const isLoading = ref(false)
const uploadResults = ref<TelegramResponse[]>([])

function initializeFromStorage() {
  prodBotKey.value = localStorage.getItem(STORAGE_KEYS.PROD_BOT_KEY) || ''
  devBotKey.value = localStorage.getItem(STORAGE_KEYS.DEV_BOT_KEY) || ''
  chatId.value = localStorage.getItem(STORAGE_KEYS.CHAT_ID) || ''
}

watch(prodBotKey, (newValue) => {
  localStorage.setItem(STORAGE_KEYS.PROD_BOT_KEY, newValue)
})

watch(devBotKey, (newValue) => {
  localStorage.setItem(STORAGE_KEYS.DEV_BOT_KEY, newValue)
})

watch(chatId, (newValue) => {
  localStorage.setItem(STORAGE_KEYS.CHAT_ID, newValue)
})

onMounted(() => {
  initializeFromStorage()
})

function handleFileChange(event: Event) {
  const input = event.target as HTMLInputElement
  if (input.files && input.files.length > 0) {
    selectedFile.value = input.files[0]
  }
}

async function sendFileToBot(botKey: string, file: File): Promise<TelegramResponse> {
  const formData = new FormData()
  formData.append('chat_id', chatId.value)
  formData.append('photo', file, file.name)

  try {
    const response = await fetch(`https://api.telegram.org/bot${botKey}/sendPhoto`, {
      method: 'POST',
      body: formData
    })
    
    if (!response.ok) {
      const errorData = await response.json()
      throw new Error(`Telegram API error: ${errorData.description || response.statusText}`)
    }
    
    return await response.json()
  } catch (error) {
    console.error(`Ошибка при отправке файла: ${error}`)
    throw error
  }
}

async function sendFile() {
  if (!selectedFile.value || !chatId.value || !prodBotKey.value || !devBotKey.value) {
    alert('Please fill in all fields and select a file')
    return
  }

  isLoading.value = true
  uploadResults.value = []
  
  try {
    const results = await Promise.all([
      sendFileToBot(prodBotKey.value, selectedFile.value),
      sendFileToBot(devBotKey.value, selectedFile.value)
    ])
    
    uploadResults.value = results
  } catch (error) {
    alert('Error sending file')
    console.error(error)
  } finally {
    isLoading.value = false
  }
}

async function copyToClipboard(text: string) {
  try {
    await navigator.clipboard.writeText(text)
  } catch (err) {
    console.error('Failed to copy text: ', err)
  }
}
</script>
