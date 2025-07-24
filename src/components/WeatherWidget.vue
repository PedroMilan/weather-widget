<template>
  <div :class="['widget', isDark ? 'dark' : 'light']">
    <!-- Input de pesquisa -->
    <input
      v-model="search"
      @input="onSearchInput"
      @keydown.down.prevent="highlightNext"
      @keydown.up.prevent="highlightPrev"
      @keydown.enter.prevent="selectHighlighted"
      placeholder="Buscar cidade..."
      class="search-input"
      autocomplete="off"
      ref="inputEl"
    />

    <!-- Lista de sugestões -->
    <ul v-if="suggestions.length > 0" class="suggestion-list">
      <li
        v-for="(item, idx) in suggestions"
        :key="`${item.lat}-${item.lon}`"
        @click="selectSuggestion(item)"
        :class="[
          'suggestion-item',
          highlightedIndex === idx ? 'highlighted' : '',
        ]"
      >
        {{ item.name }}, {{ item.state ? item.state + ", " : ""
        }}{{ item.country }}
      </li>
    </ul>

    <h2 class="title">Clima Atual</h2>

    <!-- Loading -->
    <div v-if="isLoading" class="loading">
      <svg
        class="spinner"
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
      >
        <circle
          class="spinner-bg"
          cx="12"
          cy="12"
          r="10"
          stroke="currentColor"
          stroke-width="4"
        />
        <path
          class="spinner-path"
          fill="currentColor"
          d="M4 12a8 8 0 018-8v4a4 4 0 00-4 4H4z"
        />
      </svg>
    </div>

    <!-- Dados de clima -->
    <div v-else-if="weather" class="weather-info">
      <img
        :src="`https://openweathermap.org/img/wn/${weather.weather[0].icon}@2x.png`"
        :alt="weather.weather[0].description"
        class="weather-icon"
      />
      <p class="temperature">{{ Math.round(weather.main.temp) }}°C</p>
      <p class="city">{{ weather.name }}</p>
      <p class="description">{{ weather.weather[0].description }}</p>
    </div>

    <p v-else class="error">Não foi possível obter a previsão do tempo.</p>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from "vue";

const search = ref("");
const suggestions = ref<any[]>([]);
const highlightedIndex = ref(-1);
const weather = ref<any>(null);
const isLoading = ref(false);
const isDark = ref(false);
const inputEl = ref<HTMLInputElement>();

const API_KEY = import.meta.env.VITE_WEATHER_API_KEY;

async function fetchWeather(lat: number, lon: number) {
  isLoading.value = true;
  try {
    const res = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&units=metric&lang=pt_br&appid=${API_KEY}`
    );
    if (!res.ok) throw new Error("Erro na API");
    weather.value = await res.json();
  } catch (e) {
    console.error("Erro ao buscar clima:", e);
    weather.value = null;
  } finally {
    isLoading.value = false;
  }
}

let debounceTimeout: ReturnType<typeof setTimeout> | null = null;
async function onSearchInput() {
  highlightedIndex.value = -1;
  if (debounceTimeout) clearTimeout(debounceTimeout);

  if (search.value.length < 2) {
    suggestions.value = [];
    return;
  }

  debounceTimeout = setTimeout(async () => {
    try {
      const res = await fetch(
        `https://api.openweathermap.org/geo/1.0/direct?q=${encodeURIComponent(
          search.value
        )}&limit=5&appid=${API_KEY}`
      );
      if (!res.ok) throw new Error("Erro no autocomplete");
      suggestions.value = await res.json();
    } catch (e) {
      console.error(e);
      suggestions.value = [];
    }
  }, 350);
}

function selectSuggestion(item: any) {
  search.value = `${item.name}${item.state ? ", " + item.state : ""}, ${
    item.country
  }`;
  suggestions.value = [];
  fetchWeather(item.lat, item.lon);
}

function highlightNext() {
  if (highlightedIndex.value < suggestions.value.length - 1) {
    highlightedIndex.value++;
  }
}
function highlightPrev() {
  if (highlightedIndex.value > 0) {
    highlightedIndex.value--;
  }
}
function selectHighlighted() {
  if (
    highlightedIndex.value >= 0 &&
    suggestions.value[highlightedIndex.value]
  ) {
    selectSuggestion(suggestions.value[highlightedIndex.value]);
  }
}

onMounted(() => {
  isDark.value = window.matchMedia("(prefers-color-scheme: dark)").matches;

  if ("geolocation" in navigator) {
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        fetchWeather(pos.coords.latitude, pos.coords.longitude);
      },
      () => {
        weather.value = {
          name: "São Paulo",
          main: { temp: 26.5 },
          weather: [{ icon: "01d", description: "céu limpo" }],
        };
      }
    );
  } else {
    weather.value = {
      name: "São Paulo",
      main: { temp: 26.5 },
      weather: [{ icon: "01d", description: "céu limpo" }],
    };
  }
});
</script>

<style scoped>
.widget {
  position: relative;
  max-width: 420px;
  min-width: 320px;
  margin: auto;
  padding: 1rem;
  border-radius: 1rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transition: background-color 0.5s;
  text-align: center;
}

.light {
  background-color: #ffffff;
  color: #333333;
}

.dark {
  background-color: #2d3748;
  color: #ffffff;
}

.search-input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border-radius: 9999px;
  border: 1px solid #ccc;
  margin-bottom: 2rem;
  outline: none;
  font-size: 0.9rem;
}

.dark .search-input {
  background-color: #4a5568;
  color: #ffffff;
  border-color: #718096;
}

.suggestion-list {
  position: absolute;
  top: 70px;
  left: 1rem;
  right: 1rem;
  background: #ffffff;
  border: 1px solid #ccc;
  border-radius: 8px;
  max-height: 12rem;
  overflow-y: auto;
  z-index: 10;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.2);
}

.dark .suggestion-list {
  background: #4a5568;
  color: #ffffff;
  border-color: #718096;
}

.suggestion-item {
  padding: 0.5rem 0.75rem;
  cursor: pointer;
}

.suggestion-item:hover,
.suggestion-item.highlighted {
  background-color: #cbd5e0;
}

.dark .suggestion-item:hover,
.dark .suggestion-item.highlighted {
  background-color: #2b6cb0;
}

.title {
  font-size: 1.125rem;
  font-weight: 600;
  margin-bottom: 0.5rem;
}

.loading {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 8rem;
}

.spinner {
  width: 1.5rem;
  height: 1.5rem;
  animation: spin 1s linear infinite;
  color: #718096;
}

.spinner-bg {
  opacity: 0.25;
}

.spinner-path {
  opacity: 0.75;
}

.weather-info {
  animation: fade-in 1s ease forwards;
}

.weather-icon {
  margin: auto;
  margin-bottom: 0.5rem;
}

.temperature {
  font-size: 2rem;
  font-weight: bold;
}

.city {
  font-size: 1rem;
  font-weight: 500;
}

.description {
  font-size: 0.875rem;
  margin-top: 0.25rem;
  text-transform: capitalize;
  color: #718096;
}

.dark .description {
  color: #e2e8f0;
}

.error {
  color: #e53e3e;
  font-size: 0.875rem;
}

@keyframes fade-in {
  to {
    opacity: 1;
  }
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>
