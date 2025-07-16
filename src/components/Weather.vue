<template>
  <div class="container">
    <h1>Weather Forecast</h1>

    <input
        v-model="query"
        placeholder="Search city..."
        @keydown.enter="searchCities"
    />
    <button @click="searchCities">Search</button>

    <div v-if="locations.length">
      <p>Select a location:</p>
      <ul>
        <li v-for="loc in locations" :key="loc.id">
          <button @click="selectLocation(loc)">
            {{ loc.name }}, {{ loc.country }}
          </button>
        </li>
      </ul>
    </div>
    </div>

    <div v-if="weather" class="forecast">

      <div v-if="selectedLocation" class="location-info">
        <p><strong>Weather in </strong> {{ selectedLocation.name }}, {{ selectedLocation.country }}</p>
      </div>

      <div class="current-weather">
        <h2>Current Weather</h2>
        <p>Temperature: {{ weather.current.temperature_2m }}°C</p>
        <p>Wind Speed: {{ weather.current.wind_speed_10m }} km/h</p>
        <p>Humidity: {{ weather.current.relative_humidity_2m }}%</p>
        <p>Precipitation: {{ weather.current.precipitation }}%</p>
        <p>Condition: {{ weatherCodeMap[weather.current.weather_code] || "Unknown" }}</p>
      </div>

      <h3>Hourly Forecast (Today)</h3>
      <div class="scroll-wrapper">
        <div class="scroll-row" ref="scrollRow">
          <div
              v-for="(entry, i) in todayHourlyData"
              :key="i"
              :class="['tile', entry.isNow ? 'current' : '']"
              :ref="entry.isNow ? 'currentTile' : null"
          >
            <div>{{ entry.localHour }}</div>
            <div>{{ entry.temp }}°C</div>
            <div><strong>Hum: </strong>{{ entry.humidity }}%</div>
            <div><strong>Rain: </strong>{{ entry.precip }}%</div>
            <div><strong>Wind: </strong>{{ entry.wind }} km/h</div>
            <div><strong>Cond: </strong>{{ entry.codeDesc }}</div>
          </div>
        </div>
      </div>

      <div class="daily-forecast" v-if="dailyForecastData.length">
        <h3>Daily Forecast (7 Days)</h3>
        <div class="day-tile" v-for="(day, i) in dailyForecastData" :key="i">
          <div>{{ day.date }}</div>
          <div>Max: {{ day.tempMax }}°C</div>
          <div>Min: {{ day.tempMin }}°C</div>
          <div>Rain: {{ day.precip }}%</div>
          <div>{{ day.codeDesc }}</div>
        </div>
      </div>

    </div>
</template>

<script setup>
import { ref, computed, nextTick, watch } from 'vue'
import { weatherCodeMap } from '@/utils/wmoCodes.js'

const query = ref('')
const locations = ref([])
const weather = ref(null)
const scrollRow = ref(null)
const currentTile = ref(null)
const selectedLocation = ref(null)
const dailyWeather = ref(null)

const searchCities = async () => {
  if (!query.value) return
  const res = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(query.value)}&count=5&language=en&format=json`)
  const data = await res.json()
  locations.value = data.results || []
}

const selectLocation = async (loc) => {
  selectedLocation.value = loc
  locations.value = []

  const today = new Date().toISOString().split('T')[0]
  //const tomorrow = new Date(Date.now() + 86400000).toISOString().split('T')[0]

  const url = `https://api.open-meteo.com/v1/forecast?latitude=${loc.latitude}&longitude=${loc.longitude}&current=relative_humidity_2m,temperature_2m,precipitation,wind_speed_10m,weather_code&hourly=temperature_2m,relative_humidity_2m,precipitation_probability,weathercode,windspeed_10m&start_date=${today}&end_date=${today}&timezone=UTC`

  const urlDaily = `https://api.open-meteo.com/v1/forecast?latitude=${loc.latitude}&longitude=${loc.longitude}&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max,weathercode&timezone=UTC`

  const [resHourly, resDaily] = await Promise.all([
    fetch(url),
    fetch(urlDaily),
  ])

  weather.value = await resHourly.json()
  dailyWeather.value = await resDaily.json()
}

const todayHourlyData = computed(() => {
  if (!weather.value || !weather.value.hourly) return []

  const { temperature_2m, time, relative_humidity_2m, precipitation_probability, weathercode, windspeed_10m } = weather.value.hourly

  return time.map((utc, i) => {
    const date = new Date(utc + "Z")
    return {
      time: date,
      localHour: date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }),
      isNow: isSameHourNow(date),
      temp: temperature_2m[i],
      humidity: relative_humidity_2m?.[i],
      precip: precipitation_probability?.[i],
      wind: windspeed_10m?.[i],
      code: weathercode?.[i],
      codeDesc: weatherCodeMap[weathercode?.[i]] || "Unknown"
    }
  })
})

const dailyForecastData = computed(() => {
  if (!dailyWeather.value?.daily) return []

  const { time, temperature_2m_max, temperature_2m_min, precipitation_probability_max, weathercode } = dailyWeather.value.daily

  return time.map((dateStr, i) => {
    const date = new Date(dateStr + "T00:00:00Z")
    return {
      date: date.toLocaleDateString([], { weekday: 'short', day: 'numeric', month: 'short' }),
      tempMax: temperature_2m_max[i],
      tempMin: temperature_2m_min[i],
      precip: precipitation_probability_max[i],
      code: weathercode[i],
      codeDesc: weatherCodeMap[weathercode[i]] || 'Unknown'
    }
  })
})

watch(weather, async () => {
  await nextTick()
  const tile = scrollRow.value?.querySelector('.tile.current')
  if (tile) {
    tile.scrollIntoView({ behavior: 'smooth', inline: 'center', block: 'nearest' })
  }
})

const isSameHourNow = (date) => {
  const now = new Date()
  return date.getHours() === now.getHours() &&
      date.getDate() === now.getDate()
}
</script>

<style>
.container {
  margin: 0 auto;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  gap: 2rem;
  max-width: 50%;
}

@media (max-width: 1024px) {
  .container {
    max-width: 90%;
  }
}

.forecast {
  margin: 0 auto;
  display: flex;
  flex-direction: column;
  text-align: center;
}

.location-info {
  text-align: center;
  font-size: 2rem;
  margin-bottom: 1rem;
}

.current-weather {
  margin-bottom: 1rem;
  text-align: center;
  font-size: 1rem;
}

.scroll-wrapper {
  overflow-x: auto;
  max-width: 100%;
  margin: auto;
}

.scroll-row {
  display: flex;
  gap: 0.5rem;
  padding: 1rem 0;
  max-width: 100%;
  min-width: max-content;
}

.tile {
  flex: 0 0 auto;
  min-width: 100px;
  max-width: 100px;
  word-break: break-word;
  padding: 0.75rem;
  border-radius: 8px;
  text-align: center;
  background: #eee;
  font-size: 0.75rem;
  color: black;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.tile.current {
  background: #007acc;
  color: white;
  font-weight: bold;
}

strong {
  font-weight: bold;
}

.daily-forecast {
  display: flex;
  flex-direction: column;
  justify-content: center;
  gap: 1rem;
  margin-top: 3rem;
}

.day-tile {
  color: black;
  background: #f4f4f4;
  border-radius: 10px;
  padding: 0.75rem;
  min-width: 200px;
  max-width: 200px;
  font-size: 1rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.25rem;
  margin: auto;
}
</style>