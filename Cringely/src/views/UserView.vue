<script setup>
import { ref, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import api from '../services/api'
import { useAuthStore } from '../stores/auth'
import '../assets/main.css'


const auth = useAuthStore()
const router = useRouter()

if (!auth.isAuthenticated) {
  router.push('/login')
}

const activeTab = ref('profile')

const formData = ref({
  name: '',
  description: '',
  birth_date: '',
  interests: '',
  avatar: '',
})
const saving = ref(false)
const message = ref('')

const history = ref([])
const loadingHistory = ref(false)

watch(
  () => auth.user,
  (newUser) => {
    if (newUser) {
      formData.value = { ...newUser }
    }
  },
  { immediate: true },
)

onMounted(() => {
  if (auth.isAuthenticated) {
    fetchHistory()
  }
})

const fetchHistory = async () => {
  loadingHistory.value = true
  try {
    const { data } = await api.get('/auth/history')
    history.value = data
  } catch (e) {
    console.error(e)
  } finally {
    loadingHistory.value = false
  }
}

// obsluga wyboru pliku
const handleFileUpload = (event) => {
  const file = event.target.files[0]
  if (!file) return

  if (file.size > 2 * 1024 * 1024) {
    alert('Plik jest za duży (max 2MB)')
    return
  }

  const reader = new FileReader()
  reader.onload = (e) => {
    formData.value.avatar = e.target.result 
  }
  reader.readAsDataURL(file)
}

const saveProfile = async () => {
  saving.value = true
  message.value = ''
  try {
    const { data } = await api.put('/auth/me', formData.value)
    auth.setUser(data) 
    message.value = 'Zapisano zmiany'
  } catch (e) {
    console.error(e)
    alert('Błąd zapisu. Upewnij się, że plik nie jest za duży.')
  } finally {
    saving.value = false
  }
}
</script>

<template>
  <div class="user-page">
    <!-- header -->
    <div class="user-header">
      <div class="user-avatar">
        <img
          v-if="formData.avatar"
          :src="formData.avatar"
          alt="Avatar"
          class="user-avatar-img"
        />
        <span v-else class="user-avatar-placeholder">
          {{ formData.name?.charAt(0) || 'U' }}
        </span>
      </div>

      <div class="user-header-info">
        <h1 class="user-name">{{ auth.user?.name || 'Użytkownik' }}</h1>
        <p class="user-email">{{ auth.user?.email }}</p>
      </div>
    </div>

    <!-- tabs -->
    <div class="user-tabs">
      <button
        class="user-tab-btn"
        :class="{ active: activeTab === 'profile' }"
        @click="activeTab = 'profile'"
      >
        Mój Profil
      </button>

      <button
        class="user-tab-btn"
        :class="{ active: activeTab === 'history' }"
        @click="activeTab = 'history'"
      >
        Historia Testów
      </button>
    </div>

    <!-- profil -->
    <div v-if="activeTab === 'profile'" class="user-tab-content">
      <div class="user-form-card">
        <div class="user-form-group">
          <label>Imię / Nick</label>
          <input v-model="formData.name" type="text" />
        </div>

        <div class="user-form-group">
          <label>Data urodzenia</label>
          <input v-model="formData.birth_date" type="date" />
        </div>

        <div class="user-form-group">
          <label>O sobie</label>
          <textarea v-model="formData.description"></textarea>
        </div>

        <div class="user-form-group">
          <label>Zainteresowania</label>
          <input v-model="formData.interests" />
        </div>

        <div class="user-form-group">
          <label>Zmień avatar</label>
          <input type="file" accept="image/*" @change="handleFileUpload" />
        </div>

        <button
          class="user-save-btn"
          :disabled="saving"
          @click="saveProfile"
        >
          {{ saving ? 'Zapisywanie…' : 'Zapisz zmiany' }}
        </button>

        <p v-if="message" class="user-success-msg">{{ message }}</p>
      </div>
    </div>

    <!-- historia -->
    <div v-else class="user-tab-content">
      <div v-if="loadingHistory" class="user-loading">Ładowanie…</div>
      <div v-else-if="history.length === 0" class="user-empty">
        Brak rozwiązanych testów.
      </div>

      <div v-else class="user-history-list">
        <div
          v-for="item in history"
          :key="item.id"
          class="user-history-item"
        >
          <div class="user-history-info">
            <h3>{{ item.Test?.title || 'Test usunięty' }}</h3>
            <small>
              Rozwiązano:
              {{ new Date(item.started_at).toLocaleDateString() }}
            </small>
          </div>

          <div class="user-history-score">{{ item.score }} pkt</div>
        </div>
      </div>
    </div>
  </div>
</template>
