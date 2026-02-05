<template>
  <div class="page">
    <div class="card">
      <img
        src="/assets/images/me.webp"
        alt="Изображение"
        class="avatar"
      />
      <div v-if="mode === 'choice'">
        <h1>Погода</h1>
        <p>Выберите способ определения города:</p>
        <button @click="mode = 'manual'">
          Ввести город вручную
        </button>
        <button @click="chooseAuto" :disabled="loading">
          Определить автоматически
        </button>
        <p v-if="error" class="error">
          {{ error }}
        </p>
      </div>

      <div v-else>
        <h1>Погода</h1>

        <button @click="mode = 'choice'">
          Изменить способ выбора города
        </button>

        <div>
          <label>
            Город:
            <input
              v-model="searchCity"
              @keypress="handleKeyPress"
              type="text"
              placeholder="Введите название города..."
              :disabled="loading"
            />
          </label>
          <button
            @click="handleSearch"
            :disabled="loading"
          >
            Поиск
          </button>
          <button
            @click="getCurrentLocation"
            :disabled="loading"
          >
            Определить автоматически
          </button>
        </div>

        <p v-if="error" class="error">
          {{ error }}
        </p>

        <p v-if="loading" class="loading">
          Загрузка данных...
        </p>

        <div v-if="weatherData && !loading">
          <h2>{{ weatherData.name }}</h2>
          <p>Температура: {{ Math.round(weatherData.main.temp) }} °C</p>
          <p>Описание: {{ weatherData.weather?.[0]?.description || '' }}</p>
          <p>Ощущается как: {{ Math.round(weatherData.main.feels_like) }} °C</p>
          <p>Влажность: {{ weatherData.main.humidity }}%</p>
          <p>Скорость ветра: {{ weatherData.wind?.speed || 0 }} м/с</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import type { WeatherData } from './types'

const weatherData = ref<WeatherData | null>(null)
const mode = ref<'choice' | 'manual' | 'auto'>('choice')
const loading = ref(false)
const error = ref<string | null>(null)
const searchCity = ref('')

const API_KEY = import.meta.env.VITE_OPENWEATHER_API_KEY
const API_BASE_URL = 'https://api.openweathermap.org/data/2.5/weather'

const getWeatherByCoords = async (lat: number, lon: number) => {
  try {
    loading.value = true
    error.value = null
    
    const response = await fetch(
      `${API_BASE_URL}?lat=${lat}&lon=${lon}&appid=${API_KEY}&units=metric&lang=ru`
    )
    
    if (!response.ok) {
      throw new Error('ошибка при запросе')
    }
    
    const data = await response.json()
    weatherData.value = data
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'ошибка'
  } finally {
    loading.value = false
  }
}

const chooseAuto = () => {
  mode.value = 'auto'
  getCurrentLocation()
}

const getWeatherByCity = async (city: string) => {
  if (!city.trim()) {
    error.value = 'введите название города'
    return
  }
  
  try {
    loading.value = true
    error.value = null
    
    const response = await fetch(
      `${API_BASE_URL}?q=${encodeURIComponent(city)}&appid=${API_KEY}&units=metric&lang=ru`
    )
    
    if (!response.ok) {
      if (response.status === 404) {
        throw new Error('город не найден')
      }
      throw new Error('ошибка при запросе')
    }
    
    const data = await response.json()
    weatherData.value = data
    searchCity.value = ''
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'ошибка при запросе'
  } finally {
    loading.value = false
  }
}

const getCurrentLocation = () => {
  if (!navigator.geolocation) {
    error.value = 'не получилось определить вашу позицию'
    return
  }
  
  loading.value = true
  error.value = null
  
  navigator.geolocation.getCurrentPosition(
    (position) => {
      getWeatherByCoords(position.coords.latitude, position.coords.longitude)
    },
    (err) => {
      loading.value = false
      error.value = 'не получилось определить вашу позицию, используйте поиск по городу.'
    }
  )
}

const handleSearch = () => {
  if (searchCity.value.trim()) {
    getWeatherByCity(searchCity.value.trim())
  }
}

const handleKeyPress = (event: KeyboardEvent) => {
  if (event.key === 'Enter') {
    handleSearch()
  }
}
</script>

<style scoped>
div {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.page {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 20px;
}

.card {
  width: 100%;
  max-width: 420px;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 16px;
}

.avatar {
  display: block;
  width: 260px;
  height: 260px;
  object-fit: cover;
  border-radius: 8px;
  margin: 0 auto 12px;
}

h1 {
  margin: 0 0 12px;
  font-size: 20px;
}

button {
  padding: 6px 10px;
  margin: 4px 4px 4px 0;
  border-radius: 4px;
  border: 1px solid #ccc;
  background: #f6f6f6;
  cursor: pointer;
}

button:disabled {
  opacity: 0.6;
  cursor: default;
}

input[type="text"] {
  padding: 6px 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
  margin-left: 4px;
  min-width: 180px;
}

p {
  margin: 6px 0;
}

p:nth-of-type(1) {
  margin-top: 0;
}

.error {
  color: #b00020;
}

.loading {
  font-size: 14px;
  color: #555;
}
</style>
