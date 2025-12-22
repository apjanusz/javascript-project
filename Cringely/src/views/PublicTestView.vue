<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRoute } from 'vue-router'
import api from '../services/api'
import { useAuthStore } from '../stores/auth'

import {
  Chart as ChartJS,
  ArcElement,
  Tooltip,
  Legend,
  BarElement,
  CategoryScale,
  LinearScale,
} from 'chart.js'
import { Doughnut, Bar } from 'vue-chartjs'

ChartJS.register(ArcElement, Tooltip, Legend, BarElement, CategoryScale, LinearScale)

const route = useRoute()
const accessCode = route.params.code
const auth = useAuthStore()

const test = ref(null)
const loading = ref(true)
const error = ref(null)
const guestName = ref('')
const isStarted = ref(false)
const answers = ref({})
const isSubmitting = ref(false)
const submissionResult = ref(null)
const leaderboard = ref([])

onMounted(async () => {
  try {
    const { data } = await api.get(`/tests/public/${accessCode}`)
    test.value = data

    if (test.value && test.value.Questions) {
      test.value.Questions.forEach((q) => {
        if (q.is_multiple_choice) {
          answers.value[q.id] = []
        }
      })
    }
  } catch (e) {
    error.value = 'Nie znaleziono testu lub jest on prywatny.'
  } finally {
    loading.value = false
  }
})

onMounted(() => {
  if (auth.isAuthenticated && auth.user) {
    guestName.value = auth.user.name || auth.user.email
  }
})

const startTest = async () => {
  // goscie nie maja limitow 
  if (!auth.isAuthenticated) {
    if (!guestName.value) {
      alert('Podaj swoje imię!')
      return
    }
    isStarted.value = true
    return
  }

  // sprawdzanie limitu
  try {
    await api.post(`/tests/check-access/${route.params.code}`, {})
    isStarted.value = true
  } catch (e) {
    if (e.response && e.response.data && e.response.data.error) {
      alert(e.response.data.error) 
    } else {
      alert('Wystąpił błąd podczas weryfikacji dostępu.')
    }
  }
}

const submitTest = async () => {
  if (!confirm('Czy na pewno chcesz zakończyć test?')) return

  isSubmitting.value = true
  try {
    const payload = {
      guest_name: guestName.value,
      answers: answers.value,
    }

    const { data } = await api.post(`/tests/solve/${accessCode}`, payload)
    submissionResult.value = data
    await fetchLeaderboard()
  } catch (e) {
    if (e.response && e.response.status === 403) {
      alert(e.response.data.error || 'Wyczerpano limit podejść.')
    } else {
      alert('Wystąpił błąd podczas wysyłania odpowiedzi.')
    }
    console.error(e)
  } finally {
    isSubmitting.value = false
  }
}

const fetchLeaderboard = async () => {
  try {
    const { data } = await api.get(`/tests/public/leaderboard/${accessCode}`)
    leaderboard.value = data
  } catch (e) {
    console.error('Błąd pobierania rankingu', e)
  }
}

// --- WYKRESY ---
const efficiencyChartData = computed(() => {
  if (!submissionResult.value || !submissionResult.value.stats) return null
  const s = submissionResult.value.stats
  return {
    labels: ['Poprawne', 'Błędne', 'Do oceny'],
    datasets: [
      {
        backgroundColor: ['#2ecc71', '#e74c3c', '#f39c12'],
        data: [s.correct, s.incorrect, s.pending],
      },
    ],
  }
})

const comparisonChartData = computed(() => {
  if (!submissionResult.value) return null
  const myScore = submissionResult.value.score
  const avg = submissionResult.value.stats?.averageScore || 0
  const max = submissionResult.value.maxPoints
  return {
    labels: ['Twój Wynik', 'Średnia', 'Max możliwy'],
    datasets: [
      {
        label: 'Punkty',
        backgroundColor: ['#3498db', '#95a5a6', '#2c3e50'],
        data: [myScore, avg, max],
        borderRadius: 5,
      },
    ],
  }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { position: 'bottom' } },
}

const getResultForQuestion = (qId) => {
  if (!submissionResult.value?.resultsDetails) return null
  return submissionResult.value.resultsDetails[qId]
}

const isOptionSelected = (qId, optId) => {
  const res = getResultForQuestion(qId)
  if (!res) return false

  if (Array.isArray(res.userAnswer)) {
    return res.userAnswer.map(String).includes(String(optId))
  }
  return String(res.userAnswer) === String(optId)
}

// --- POPRAWKA BŁĘDU ---
const isOptionCorrect = (qId, optId) => {
  const res = getResultForQuestion(qId)
  if (!res) return false

  // Checkbox (wiele poprawnych) - sprawdzamy tylko gdy tablica NIE jest pusta
  if (
    res.correctOptionIds &&
    Array.isArray(res.correctOptionIds) &&
    res.correctOptionIds.length > 0
  ) {
    return res.correctOptionIds.map(String).includes(String(optId))
  }
  // Fallback dla Radio (jednokrotny)
  return String(res.correctOptionId) === String(optId)
}

const toggleCheckbox = (questionId, optionId) => {
  if (!Array.isArray(answers.value[questionId])) {
    answers.value[questionId] = []
  }

  const idx = answers.value[questionId].indexOf(optionId)

  if (idx === -1) {
    answers.value[questionId].push(optionId)
  } else {
    answers.value[questionId].splice(idx, 1)
  }
}
</script>

<template>
  <div class="public-page">
    <div v-if="loading" class="center">Ładowanie testu...</div>
    <div v-else-if="error" class="center err">{{ error }}</div>

    <div v-else class="test-container">
      <div v-if="!isStarted && !submissionResult" class="public-start">
        <div class="public-card start-card">
          <h1 class="public-title">{{ test.title }}</h1>
          <p class="public-author">Autor: {{ test.User?.name || 'Nieznany' }}</p>
          <p v-iftest.description class="desc">{{ test.description }}</p>

          <div v-if="test.attempts_limit > 0" class="limit-info">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              class="icon icon-tabler icons-tabler-outline icon-tabler-info-circle"
            >
              <path stroke="none" d="M0 0h24v24H0z" fill="none" />
              <path d="M3 12a9 9 0 1 0 18 0a9 9 0 0 0 -18 0" />
              <path d="M12 9h.01" />
              <path d="M11 12h1v4h1" />
            </svg>
            <label>Ten test ma limit podejść: {{ test.attempts_limit }}</label>
          </div>

          <div class="guest-form">
            <template v-if="!auth.isAuthenticated">
              <label>Twoje imię / nick</label>
              <input v-model="guestName" placeholder="Jan Kowalski" />
            </template>
            <template v-else>
              <p class="logged-user">
                Rozwiązujesz jako:
                <strong style="font-weight: bold">{{ auth.user.name || auth.user.email }}</strong>
              </p>
            </template>
            <button class="btn-green start-btn" @click="startTest">Rozpocznij test</button>
          </div>
        </div>
      </div>

      <div v-else-if="submissionResult" class="result-card">
        <div class="result-header"><h1 class="score-title">Wyniki Testu</h1></div>

        <div v-if="submissionResult.requiresGrading" class="pending-box">
          <p class="pending-msg">Czekaj na ocenę zadań otwartych.</p>
        </div>

        <div class="score-header">
          <div class="score-circle" :class="{ partial: submissionResult.requiresGrading }">
            <span>{{ submissionResult.score }} / {{ submissionResult.maxPoints }}</span>
          </div>
          <div class="score-text">
            <h3>Dobra robota, {{ guestName }}! Oto Twoje statystyki:</h3>
          </div>
        </div>

        <div class="charts-row">
          <div class="tstats-chart-card" v-if="efficiencyChartData">
            <h4 class="tstats-chart-title">Twoja skuteczność</h4>
            <div class="chart-wrapper">
              <Doughnut :data="efficiencyChartData" :options="chartOptions" />
            </div>
          </div>
          <div class="tstats-chart-card" v-if="comparisonChartData">
            <h4 class="tstats-chart-title">Porównanie z innymi</h4>
            <div class="chart-wrapper">
              <Bar :data="comparisonChartData" :options="chartOptions" />
            </div>
          </div>
        </div>

        <div v-if="submissionResult.resultsDetails" class="review-section">
          <h3>Przegląd Odpowiedzi</h3>
          <div v-for="(q, index) in test.Questions" :key="q.id" class="review-item">
            <div class="review-header">
              <span class="q-number">#{{ index + 1 }}</span>
              <span class="q-text">{{ q.text }}</span>

              <span v-if="getResultForQuestion(q.id)?.isCorrect === true" class="status-badge ok"
                >Dobrze</span
              >
              <span
                v-else-if="getResultForQuestion(q.id)?.isCorrect === false"
                class="status-badge bad"
                >Źle</span
              >
              <span v-else class="status-badge wait">Do oceny</span>
            </div>

            <div class="review-body">
              <div v-if="q.question_type === 'ABC'">
                <div
                  v-for="opt in q.QuestionOptions"
                  :key="opt.id"
                  class="review-opt"
                  :class="{
                    'user-selected': isOptionSelected(q.id, opt.id),
                    'correct-answer': isOptionCorrect(q.id, opt.id),
                  }"
                >
                  <span v-if="isOptionSelected(q.id, opt.id) && isOptionCorrect(q.id, opt.id)">
                  </span>
                  <span v-else-if="isOptionSelected(q.id, opt.id)">❌</span>
                  <span v-else-if="isOptionCorrect(q.id, opt.id)">Poprawna: </span>

                  {{ opt.text }}
                </div>
              </div>

              <div v-else-if="q.question_type === 'FILL'">
                <p>
                  Twoja odpowiedź:
                  <strong>{{ getResultForQuestion(q.id)?.userAnswer || '(Brak)' }}</strong>
                </p>
                <p v-if="getResultForQuestion(q.id)?.isCorrect === false" class="correction-text">
                  Poprawna: <strong>{{ getResultForQuestion(q.id)?.correctText }}</strong>
                </p>
              </div>

              <div v-else>
                <p class="user-open-ans">{{ getResultForQuestion(q.id)?.userAnswer }}</p>
              </div>
            </div>
          </div>
        </div>

        <div class="leaderboard-box">
          <h3>🏆 Top 10 Najlepszych</h3>
          <ul v-if="leaderboard.length">
            <li
              v-for="(entry, idx) in leaderboard"
              :key="idx"
              :class="{ me: entry.guest_name === guestName }"
            >
              <span class="rank">#{{ idx + 1 }}</span>
              <span class="name">{{ entry.guest_name }}</span>
              <span class="points">{{ entry.score }} pkt</span>
            </li>
          </ul>
          <p v-else class="empty-list">Bądź pierwszy na liście!</p>
        </div>

        <button class="btn-back" @click="$router.push('/')">Wróć na stronę główną</button>
      </div>

      <div v-else class="solve-view">
        <div class="solve-header">
          <h1 class="solve-title">{{ test.title }}</h1>
          <span class="solve-user">
            Uczeń: <strong>{{ guestName }}</strong>
          </span>
        </div>

        <div v-for="(q, index) in test.Questions" :key="q.id" class="solve-question-card">
          <div class="solve-question-head">
            <span class="solve-question-index">{{ index + 1 }}</span>

            <div class="solve-question-meta">
              <span class="solve-question-text">{{ q.text }}</span>
              <span class="solve-question-points">{{ q.points }} pkt</span>
            </div>

            <span v-if="q.is_multiple_choice" class="solve-multi-badge"> Wielokrotny wybór </span>
          </div>

          <div v-if="q.question_type === 'ABC'" class="solve-options">
            <div v-for="opt in q.QuestionOptions" :key="opt.id" class="solve-option-row">
              <input
                v-if="q.is_multiple_choice"
                type="checkbox"
                :value="opt.id"
                v-model="answers[q.id]"
                class="solve-hidden-input"
              />

              <input
                v-else
                type="radio"
                :name="'q' + q.id"
                :value="opt.id"
                v-model="answers[q.id]"
                class="solve-hidden-input"
              />

              <div
                class="solve-option-box"
                :class="{
                  active: q.is_multiple_choice
                    ? answers[q.id]?.includes(opt.id)
                    : answers[q.id] === opt.id,
                }"
                @click="
                  q.is_multiple_choice ? toggleCheckbox(q.id, opt.id) : (answers[q.id] = opt.id)
                "
              >
                {{ opt.text }}
              </div>
            </div>
          </div>

          <div v-else-if="q.question_type === 'FILL'" class="solve-input">
            <input type="text" v-model="answers[q.id]" placeholder="Wpisz odpowiedź..." />
          </div>

          <div v-else-if="q.question_type === 'OPEN'" class="solve-textarea">
            <textarea v-model="answers[q.id]" placeholder="Twoja odpowiedź..."></textarea>
          </div>
        </div>

        <button class="solve-finish-btn" @click="submitTest" :disabled="isSubmitting">
          {{ isSubmitting ? 'Wysyłanie...' : 'Zakończ i wyślij' }}
        </button>
      </div>
    </div>
  </div>
</template>

