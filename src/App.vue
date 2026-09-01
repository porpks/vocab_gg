<template>
  <div class="app-container">
    <div v-if="currentScreen === 'loading'" class="screen active">
      <div class="loading-bar">
        <span class="header-title">Loading...</span>
      </div>
    </div>

    <div v-else-if="currentScreen === 'main'" class="screen active">
      <div class="menu-container">
        <div class="menu-card" @click="startExercise('new')">
          <h2>New Vocab</h2>
          <span class="count-badge">{{ activeNewVocabList.length }} words</span>
        </div>
        <div class="menu-card" @click="startExercise('remember')">
          <h2>Remember Vocab</h2>
          <span class="count-badge">{{ rememberVocabList.length }} words</span>
        </div>
        <div class="menu-card" @click="startExercise('review')">
          <h2>Review Vocab</h2>
          <span class="count-badge">{{ reviewVocabList.length }} words</span>
        </div>
      </div>
    </div>

    <div v-else-if="currentScreen === 'exercise'" class="screen active">
      <button class="back-btn" @click="showScreen('main')">← Back to Menu</button>
      <div class="exercise-card">
        <div class="header-bar">
          <span>vocab {{ currentIndex + 1 }}/{{ currentList.length }}</span>
          <span class="header-title">{{ modeTitle }}</span>
          <span>{{ progressPercent }}%</span>
        </div>

        <div v-if="currentItem" class="vocab-card">
          <div class="pos-tag">{{ currentItem[KEY_POS] || "Part of speech" }}</div>
          <div class="word-title">{{ currentItem[KEY_MEANING] }}</div>
          <div class="hint-text">{{ hintText }}</div>
        </div>

        <div :class="['feedback', feedbackType]">{{ feedbackMessage }}</div>

        <div class="input-group">
          <input 
            type="text" 
            v-model="userInput" 
            class="input-field" 
            placeholder="Typing..." 
            autocomplete="off"
            :disabled="inputDisabled"
            @keydown.enter="handleSubmit"
          >
          <button 
            :class="['btn btn-submit', { active: userInput.trim().length > 0 }]" 
            @click="handleSubmit"
            :disabled="isProcessing || userInput.trim().length === 0"
          >
            Submit
          </button>
        </div>

        <div class="bottom-controls">
          <button class="btn" @click="handlePass" :disabled="isProcessing">Pass</button>
          <button class="btn" @click="handleAnswer" :disabled="isProcessing">Answer</button>
          <button class="btn" @click="handleNote" :disabled="isProcessing">Note</button>
        </div>
      </div>
    </div>

    <div v-else-if="currentScreen === 'summary'" class="screen active">
      <div class="exercise-card">
        <div class="header-bar">
          <span class="header-title">Session Summary</span>
        </div>

        <div class="summary-score-card">
          <div>Score (Correct Submissions)</div>
          <div class="score-number">{{ correctList.length }} / {{ currentList.length }}</div>
        </div>

        <div class="summary-section">
          <h3>Passed Vocab</h3>
          <div class="word-list">
            <span v-if="passedList.length === 0" class="empty-text">None</span>
            <span v-for="(item, idx) in passedList" :key="idx" class="word-badge">{{ item[KEY_WORD] }}</span>
          </div>
        </div>

        <div class="summary-section">
          <h3>Revealed Answers</h3>
          <div class="word-list">
            <span v-if="answeredList.length === 0" class="empty-text">None</span>
            <span v-for="(item, idx) in answeredList" :key="idx" class="word-badge">{{ item[KEY_WORD] }}</span>
          </div>
        </div>

        <div class="summary-section">
          <h3>Noted Vocab</h3>
          <div class="word-list">
            <span v-if="notedList.length === 0" class="empty-text">None</span>
            <span v-for="(item, idx) in notedList" :key="idx" class="word-badge">{{ item[KEY_WORD] }}</span>
          </div>
        </div>

        <button class="btn btn-submit active" style="width: 100%; margin-top: 10px;" @click="showScreen('main')">
          Return to Main Menu
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

// --- Constants ---
const SHEET_NEW_VOCAB = "New Vocab";
const SHEET_REMEMBER_VOCAB = "Remember Vocab";
const SHEET_REVIEW_VOCAB = "Review Vocab";
const KEY_WORD = "Word";
const KEY_POS = "Parts of Speech";
const KEY_MEANING = "Definition";

const BASE_ENDPOINT_URL = "https://sheetdb.io/api/v1/avvuri7rkjp0b"; 
const NEW_VOCAB_CSV_URL = `${BASE_ENDPOINT_URL}?sheet=${SHEET_NEW_VOCAB}`; 
const REMEMBER_VOCAB_CSV_URL = `${BASE_ENDPOINT_URL}?sheet=${SHEET_REMEMBER_VOCAB}`; 
const REVIEW_VOCAB_CSV_URL = `${BASE_ENDPOINT_URL}?sheet=${SHEET_REVIEW_VOCAB}`; 

// --- Reactive State ---
const currentScreen = ref('loading'); // 'loading', 'main', 'exercise', 'summary'
const baseNewVocabList = ref([]);
const rememberVocabList = ref([]);
const reviewVocabList = ref([]);

const currentList = ref([]);
const currentIndex = ref(0);
const currentMode = ref('new');
const isProcessing = ref(false);

const correctList = ref([]);
const passedList = ref([]);
const answeredList = ref([]);
const notedList = ref([]);

const userInput = ref('');
const feedbackMessage = ref('');
const feedbackType = ref(''); // 'correct', 'wrong', 'info'
const inputDisabled = ref(false);

// --- Computed Properties ---
const activeNewVocabList = computed(() => {
  return baseNewVocabList.value.filter(item => 
    !rememberVocabList.value.some(r => r[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase()) &&
    !reviewVocabList.value.some(rv => rv[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase())
  );
});

const currentItem = computed(() => currentList.value[currentIndex.value]);

const modeTitle = computed(() => {
  const titles = { new: 'New Vocab', remember: 'Remember Vocab', review: 'Review Vocab' };
  return titles[currentMode.value];
});

const progressPercent = computed(() => {
  if (currentList.value.length === 0) return 0;
  return Math.round((currentIndex.value / currentList.value.length) * 100);
});

const hintText = computed(() => {
  if (!currentItem.value) return '';
  return currentItem.value[KEY_WORD].substring(0, 2) + "...";
});


// --- Core Methods ---
function showScreen(screen) {
  currentScreen.value = screen;
}

function startExercise(mode) {
  currentMode.value = mode;
  currentIndex.value = 0;
  isProcessing.value = false;

  if (mode === 'new') {
    const available = activeNewVocabList.value;
    currentList.value = shuffleArray([...available]).slice(0, 25);
  } else if (mode === 'remember') {
    currentList.value = [...rememberVocabList.value];
  } else {
    currentList.value = [...reviewVocabList.value];
  }

  if (currentList.value.length === 0) {
    alert("No words available in this section!");
    return;
  }

  correctList.value = [];
  passedList.value = [];
  answeredList.value = [];
  notedList.value = [];

  showScreen('exercise');
  resetQuestionUI();
}

function resetQuestionUI() {
  userInput.value = "";
  inputDisabled.value = false;
  feedbackMessage.value = "";
  feedbackType.value = "";
}

function loadNextQuestion() {
  if (currentIndex.value + 1 >= currentList.value.length) {
    showScreen('summary');
  } else {
    currentIndex.value++;
    resetQuestionUI();
  }
}

// --- List Management ---
async function saveStorage(prevRememberData = [], prevReviewData = []) {
  if (rememberVocabList.value.length > prevRememberData.length) {
    await addToGoogleSheet(REMEMBER_VOCAB_CSV_URL, rememberVocabList.value, prevRememberData);
  } else if (rememberVocabList.value.length < prevRememberData.length) {
    await removeFromGoogleSheet(REMEMBER_VOCAB_CSV_URL, rememberVocabList.value, prevRememberData);
  }

  if (reviewVocabList.value.length > prevReviewData.length) {
    await addToGoogleSheet(REVIEW_VOCAB_CSV_URL, reviewVocabList.value, prevReviewData);
  } else if (reviewVocabList.value.length < prevReviewData.length) {
    await removeFromGoogleSheet(REVIEW_VOCAB_CSV_URL, reviewVocabList.value, prevReviewData);
  }
}

function moveToReview(item) {
  const prevRememberData = [...rememberVocabList.value];
  const prevReviewData = [...reviewVocabList.value];
  
  rememberVocabList.value = rememberVocabList.value.filter(v => v[KEY_WORD].toLowerCase() !== item[KEY_WORD].toLowerCase());
  
  if (!reviewVocabList.value.some(v => v[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase())) {
    reviewVocabList.value.push(item);
  }
  saveStorage(prevRememberData, prevReviewData);
}

function moveToRemember(item) {
  const prevRememberData = [...rememberVocabList.value];
  const prevReviewData = [...reviewVocabList.value];
  
  if (!rememberVocabList.value.some(v => v[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase()) &&
      !reviewVocabList.value.some(v => v[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase())) {
    rememberVocabList.value.push(item);
    saveStorage(prevRememberData, prevReviewData);
  }
}

function demoteFromReviewToRemember(item) {
  const prevRememberData = [...rememberVocabList.value];
  const prevReviewData = [...reviewVocabList.value];
  
  reviewVocabList.value = reviewVocabList.value.filter(v => v[KEY_WORD].toLowerCase() !== item[KEY_WORD].toLowerCase());
  
  if (!rememberVocabList.value.some(v => v[KEY_WORD].toLowerCase() === item[KEY_WORD].toLowerCase())) {
    rememberVocabList.value.push(item);
  }
  saveStorage(prevRememberData, prevReviewData);
}

// --- Action Handlers ---
function handleSubmit() {
  if (isProcessing.value || !userInput.value.trim()) return;

  const input = userInput.value.trim().toLowerCase();
  const answer = currentItem.value[KEY_WORD].toLowerCase();

  if (input === answer) {
    feedbackMessage.value = "Correct!";
    feedbackType.value = "correct";
    correctList.value.push(currentItem.value);

    moveToReview(currentItem.value);

    isProcessing.value = true;
    setTimeout(() => {
      isProcessing.value = false;
      loadNextQuestion();
    }, 600);
  } else {
    feedbackMessage.value = "Incorrect. Try again!";
    feedbackType.value = "wrong";
  }
}

function handlePass() {
  if (isProcessing.value) return;
  const item = currentItem.value;
  passedList.value.push(item);
  
  if (currentMode.value === 'new') moveToRemember(item);
  else if (currentMode.value === 'review') demoteFromReviewToRemember(item);

  loadNextQuestion();
}

function handleAnswer() {
  if (isProcessing.value) return;
  isProcessing.value = true;

  const item = currentItem.value;
  userInput.value = item[KEY_WORD];
  inputDisabled.value = true;
  
  feedbackMessage.value = "Revealing answer (3s)...";
  feedbackType.value = "info";

  answeredList.value.push(item);

  if (currentMode.value === 'new') moveToRemember(item);
  else if (currentMode.value === 'review') demoteFromReviewToRemember(item);

  setTimeout(() => {
    isProcessing.value = false;
    loadNextQuestion();
  }, 3000);
}

function handleNote() {
  if (isProcessing.value) return;
  const item = currentItem.value;
  notedList.value.push(item);
  
  if (currentMode.value === 'new') moveToRemember(item);
  else if (currentMode.value === 'review') demoteFromReviewToRemember(item);

  loadNextQuestion();
}

// --- Utility / API ---
function shuffleArray(array) {
  const arr = [...array];
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

async function fetchGoogleSheet(csvUrl) {
  if (!csvUrl) return null;
  try {
    const response = await fetch(csvUrl);
    const text = await response.text();
    const json = JSON.parse(text);
    return Array.isArray(json) ? json : null;
  } catch (e) {
    console.error("Error fetching Google Sheet data:", e);
    return null;
  }
}

async function addToGoogleSheet(csvUrl, data, prevdata = []) {
  if (!csvUrl || !Array.isArray(data) || data.length === 0) return null;

  const existingData = prevdata.length > 0 ? prevdata : (await fetchGoogleSheet(csvUrl) || []);
  const uniqueData = data.filter(item => !existingData.some(existingItem => existingItem[KEY_WORD] === item[KEY_WORD]));
  
  if(uniqueData.length === 0) return null;

  try {
    const response = await fetch(csvUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(uniqueData)
    });
    return await response.json();
  } catch (e) {
    console.error("Error saving to Google Sheet:", e);
    return null;
  }
}

async function removeFromGoogleSheet(csvUrl, data, prevdata = []) {
  if (!csvUrl || !Array.isArray(data)) return null;

  const existingData = prevdata.length > 0 ? prevdata : (await fetchGoogleSheet(csvUrl) || []);
  const filteredData = data.length > 0 ? existingData.filter(item => !data.some(removeItem => removeItem[KEY_WORD] === item[KEY_WORD])) : existingData;
  
  const results = [];
  for (const item of filteredData) {
    try {
      const endpoint = csvUrl.replace(BASE_ENDPOINT_URL, `${BASE_ENDPOINT_URL}/${KEY_WORD}/${item[KEY_WORD]}`);
      const response = await fetch(endpoint, {
        method: 'DELETE',
        headers: { 'Content-Type': 'application/json' },
      });
      const result = await response.json();
      results.push(result);
    } catch (e) {
      console.error("Error removing from Google Sheet:", e);
    }
  }
  return results;
}

// --- Initialization ---
onMounted(async () => {
  const fetchedNew = await fetchGoogleSheet(NEW_VOCAB_CSV_URL);
  if (fetchedNew) baseNewVocabList.value = fetchedNew;
  
  const fetchedRemember = await fetchGoogleSheet(REMEMBER_VOCAB_CSV_URL);
  if (fetchedRemember) rememberVocabList.value = fetchedRemember;

  const fetchedReview = await fetchGoogleSheet(REVIEW_VOCAB_CSV_URL);
  if (fetchedReview) reviewVocabList.value = fetchedReview;

  showScreen('main');
});
</script>

<style>
:root {
  --bg-color: #f0fdf4;
  --card-bg: #ffffff;
  --text-main: #0f172a;
  --text-muted: #64748b;
  --primary-blue: #2563eb;
  --primary-green: #10b981;
  --border-color: #e2e8f0;
  --shadow-sm: 0 4px 16px rgba(15, 23, 42, 0.04);
  --shadow-md: 0 10px 30px rgba(15, 23, 42, 0.08);
}

.app-container {
  background-color: var(--bg-color);
  color: var(--text-main);
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

/* Screen Management */
.screen {
  width: 100%;
  max-width: 520px;
}

/* Main Menu Screen */
.menu-container {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.menu-card {
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 20px;
  padding: 32px 24px;
  text-align: center;
  box-shadow: var(--shadow-sm);
  cursor: pointer;
  transition: all 0.2s ease;
}

.menu-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-md);
  border-color: var(--primary-green);
}

.menu-card h2 {
  font-size: 26px;
  font-weight: 500;
  color: var(--text-main);
  margin: 0;
}

.menu-card .count-badge {
  display: inline-block;
  margin-top: 6px;
  font-size: 13px;
  color: var(--text-muted);
}

/* Exercise & Summary Screen */
.exercise-card {
  background: var(--card-bg);
  border-radius: 24px;
  padding: 28px;
  box-shadow: var(--shadow-md);
  border: 1px solid var(--border-color);
  position: relative;
}

.header-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 15px;
  color: var(--text-muted);
  margin-bottom: 24px;
}

.header-title {
  font-weight: 600;
  color: var(--text-main);
  font-size: 16px;
}

.vocab-card {
  border: 1px solid var(--border-color);
  border-radius: 16px;
  padding: 40px 20px;
  text-align: center;
  margin-bottom: 24px;
  background: #fafafa;
}

.pos-tag {
  font-size: 14px;
  color: var(--text-muted);
  margin-bottom: 12px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.word-title {
  font-size: 34px;
  font-weight: 600;
  color: var(--text-main);
  margin-bottom: 16px;
}

.hint-text {
  font-size: 16px;
  color: var(--text-muted);
  letter-spacing: 2px;
  min-height: 24px;
}

.input-group {
  display: flex;
  gap: 10px;
  margin-bottom: 16px;
}

.input-field {
  flex: 1;
  padding: 12px 20px;
  border: 1px solid var(--border-color);
  border-radius: 30px;
  font-size: 16px;
  outline: none;
  transition: border 0.2s;
}

.input-field:focus {
  border-color: var(--primary-blue);
}

.btn {
  border: 1px solid var(--border-color);
  background: var(--card-bg);
  border-radius: 30px;
  padding: 10px 20px;
  font-size: 14px;
  font-weight: 500;
  color: var(--text-main);
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn:hover:not(:disabled) {
  background: #f8fafc;
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-submit {
  padding: 10px 24px;
}

.btn-submit.active {
  background: var(--primary-blue);
  color: white;
  border-color: var(--primary-blue);
}

.bottom-controls {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.back-btn {
  background: none;
  border: none;
  color: var(--text-muted);
  cursor: pointer;
  font-size: 14px;
  margin-bottom: 12px;
  display: inline-block;
  padding: 0;
}

.back-btn:hover {
  color: var(--text-main);
}

.feedback {
  text-align: center;
  font-size: 14px;
  font-weight: 500;
  margin-bottom: 12px;
  min-height: 20px;
}
.feedback.correct { color: var(--primary-green); }
.feedback.wrong { color: #ef4444; }
.feedback.info { color: var(--primary-blue); }

/* Summary Screen Specific Styles */
.summary-score-card {
  text-align: center;
  padding: 20px;
  background: #fafafa;
  border-radius: 16px;
  margin-bottom: 20px;
  border: 1px solid var(--border-color);
}

.score-number {
  font-size: 42px;
  font-weight: 700;
  color: var(--primary-green);
  margin-top: 4px;
}

.summary-section {
  margin-bottom: 20px;
}

.summary-section h3 {
  font-size: 15px;
  color: var(--text-muted);
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.word-list {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.word-badge {
  background: #f1f5f9;
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 14px;
  color: var(--text-main);
  border: 1px solid var(--border-color);
}

.empty-text {
  font-size: 13px;
  color: var(--text-muted);
  font-style: italic;
}

.loading-bar {
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>