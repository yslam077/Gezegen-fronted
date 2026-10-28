<script setup>
import { ref, computed, onMounted } from 'vue'

const API_URL = 'http://127.0.0.1:8000/api/v1'

const searchFrom = ref('')
const searchTo = ref('')
const searchDate = ref('')

const airports = ref([])
const flights = ref([])

const loading = ref(false)
const searched = ref(false)
const notification = ref({ text: '', isError: false })

const selectedFlight = ref(null)
const bookingForm = ref({
  passengers_count: 1,
  passengers: [
    { full_name: '', phone: '' }
  ]
})

onMounted(async () => {
  await fetchAirports()
  await fetchFlights()
})

const fetchAirports = async () => {
  try {
    const response = await fetch(`${API_URL}/airports`, {
      headers: { 'Accept': 'application/json' }
    })
    const result = await response.json()
    airports.value = Array.isArray(result) ? result : (result.data || [])
  } catch (err) {
    console.error('Ошибка загрузки аэропортов:', err)
  }
}

const fetchFlights = async () => {
  loading.value = true
  notification.value.text = ''
  
  try {
    const response = await fetch(`${API_URL}/flights`, {
      headers: { 'Accept': 'application/json' }
    })
    
    if (!response.ok) {
      throw new Error(`Ошибка сервера: ${response.status}`)
    }

    const result = await response.json()
    flights.value = Array.isArray(result) ? result : (result.data || [])
    searched.value = true
  } catch (err) {
    notification.value = {
      text: 'Не удалось загрузить рейсы с сервера Laravel.',
      isError: true
    }
  } finally {
    loading.value = false
  }
}

const getAirportDisplay = (airportField, airportId) => {
  if (typeof airportField === 'object' && airportField !== null) {
    return `${airportField.name} (${airportField.code}, г. ${airportField.city})`
  }
  if (airportId) {
    const found = airports.value.find(a => String(a.id) === String(airportId))
    if (found) return `${found.name} (${found.code}, г. ${found.city})`
  }
  return 'Не указано'
}

const filteredFlights = computed(() => {
  return flights.value.filter(flight => {
    // 1. Проверка наличия свободных мест
    const hasSeats = Number(flight.empty_seats) > 0

    const fromId = flight.from_airport?.id || flight.from_airport_id || ''
    const toId = flight.to_airport?.id || flight.to_airport_id || ''
    const flightDate = flight.departure_date || (flight.departure_time ? flight.departure_time.split(' ')[0] : '')

    const matchFrom = !searchFrom.value || String(fromId) === String(searchFrom.value)
    const matchTo = !searchTo.value || String(toId) === String(searchTo.value)
    const matchDate = !searchDate.value || flightDate === searchDate.value

    return hasSeats && matchFrom && matchTo && matchDate
  })
})

const openBooking = (flight) => {
  selectedFlight.value = flight
  bookingForm.value = {
    passengers_count: 1,
    passengers: [
      { full_name: '', phone: '' }
    ]
  }
}

const closeModal = () => {
  selectedFlight.value = null
}

const updatePassengersCount = (delta) => {
  let newCount = bookingForm.value.passengers_count + delta
  const maxSeats = selectedFlight.value ? selectedFlight.value.empty_seats : 1

  if (newCount < 1) newCount = 1
  if (newCount > maxSeats) newCount = maxSeats

  bookingForm.value.passengers_count = newCount

  if (bookingForm.value.passengers.length < newCount) {
    while (bookingForm.value.passengers.length < newCount) {
      bookingForm.value.passengers.push({ full_name: '', phone: '' })
    }
  } else if (bookingForm.value.passengers.length > newCount) {
    bookingForm.value.passengers = bookingForm.value.passengers.slice(0, newCount)
  }
}

const totalPrice = computed(() => {
  if (!selectedFlight.value || !selectedFlight.value.price) return 0
  return Number(selectedFlight.value.price) * bookingForm.value.passengers_count
})

const bookTicket = async () => {
  for (let i = 0; i < bookingForm.value.passengers.length; i++) {
    if (!bookingForm.value.passengers[i].full_name.trim()) {
      alert(`Пожалуйста, укажите ФИО для пассажира №${i + 1}`)
      return
    }
  }

  try {
    const response = await fetch(`${API_URL}/tickets`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
      },
      body: JSON.stringify({
        flight_id: selectedFlight.value.id,
        passengers_count: bookingForm.value.passengers_count,
        passengers: bookingForm.value.passengers
      })
    })

    const result = await response.json()

    if (response.ok) {
      notification.value = {
        text: 'Билеты успешно забронированы! Все пассажиры сохранены в базе.',
        isError: false
      }
      closeModal()
      await fetchFlights()
    } else {
      notification.value = {
        text: result.message || 'Ошибка бронирования',
        isError: true
      }
    }
  } catch (err) {
    notification.value = {
      text: 'Ошибка сети при отправке запроса.',
      isError: true
    }
  }
}
</script>

<template>
  <div class="max-w-4xl mx-auto p-6 font-sans text-slate-800">
    <header class="mb-8 text-center">
      <h1 class="text-3xl font-extrabold text-indigo-600">Billet gözlemek we bronlamak</h1>
      <p class="text-slate-500 mt-1">Biilet saýlaş we maglumatynyzy ýazyň</p>
    </header>

    <div class="bg-white p-6 rounded-2xl shadow-md border border-slate-100 mb-8">
      <div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
        <div>
          <label class="block text-sm font-medium text-slate-700 mb-1">Nireden</label>
          <select 
            v-model="searchFrom" 
            class="w-full px-4 py-2 border border-slate-300 rounded-xl focus:ring-2 focus:ring-indigo-500 focus:outline-none bg-white"
          >
            <option value="">Şäheri saýlan</option>
            <option v-for="airport in airports" :key="airport.id" :value="airport.id">
              {{ airport.name }} ({{ airport.code }})
            </option>
          </select>
        </div>

        <div>
          <label class="block text-sm font-medium text-slate-700 mb-1">Niräk</label>
          <select 
            v-model="searchTo" 
            class="w-full px-4 py-2 border border-slate-300 rounded-xl focus:ring-2 focus:ring-indigo-500 focus:outline-none bg-white"
          >
            <option value="">Şäheri saýlan</option>
            <option v-for="airport in airports" :key="airport.id" :value="airport.id">
              {{ airport.name }} ({{ airport.code }})
            </option>
          </select>
        </div>

        <div>
          <label class="block text-sm font-medium text-slate-700 mb-1">Uçuş wagty</label>
          <input 
            v-model="searchDate" 
            type="date" 
            class="w-full px-4 py-2 border border-slate-300 rounded-xl focus:ring-2 focus:ring-indigo-500 focus:outline-none"
          />
        </div>
      </div>

      <button 
        @click="fetchFlights" 
        :disabled="loading"
        class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-3 rounded-xl transition cursor-pointer"
      >
        {{ loading ? 'Garaşyn...' : 'Maglumat täzelemk' }}
      </button>
    </div>

    <div 
      v-if="notification.text" 
      class="mb-6 p-4 rounded-xl text-sm border font-medium"
      :class="notification.isError ? 'bg-red-50 text-red-700 border-red-200' : 'bg-emerald-50 text-emerald-700 border-emerald-200'"
    >
      {{ notification.text }}
    </div>

    <div v-if="searched">
      <h2 class="text-xl font-bold mb-4">Elýeter billetlar ({{ filteredFlights.length }})</h2>

      <div v-if="filteredFlights.length > 0" class="space-y-4">
        <div 
          v-for="flight in filteredFlights" 
          :key="flight.id" 
          class="bg-white p-5 rounded-2xl shadow-sm border border-slate-200 flex flex-col md:flex-row justify-between items-start md:items-center gap-4"
        >
          <div>
            <div class="flex items-center gap-2 mb-2">
              <span class="font-bold text-lg text-indigo-600">Reýs: {{ flight.flight_number }} ({{ flight.airline }})</span>
              <span class="bg-indigo-50 text-indigo-700 text-xs px-2.5 py-1 rounded-full font-medium">
                Boş orunlar {{ flight.empty_seats }}
              </span>
            </div>
            
            <div class="text-slate-700 space-y-1">
              <p class="text-sm">
                ✈️ <span class="font-semibold">Nireden</span> {{ getAirportDisplay(flight.from_airport, flight.from_airport_id) }}
              </p>
              <p class="text-sm">
                🛬 <span class="font-semibold">Niräk</span> {{ getAirportDisplay(flight.to_airport, flight.to_airport_id) }}
              </p>
              <p class="text-xs text-slate-400">
                📅 Sene: {{ flight.departure_date || 'Berilmek' }} | 🕒 Ucus: {{ flight.departure_time }} | Gonmak: {{ flight.arrival_time }}
              </p>
              <p class="text-xs text-slate-500">
                ⏱ Wagt: {{ flight.duration }}
              </p>
              <p v-if="flight.price" class="text-base font-bold text-emerald-600 mt-1">
                💰 1 adamlyk orun bahasy: ${{ flight.price }}
              </p>
            </div>
          </div>

          <button 
            @click="openBooking(flight)" 
            class="px-5 py-2.5 rounded-xl font-medium text-white transition cursor-pointer bg-emerald-600 hover:bg-emerald-700"
          >
            Bron edmek
          </button>
        </div>
      </div>

      <div v-else class="text-center py-12 bg-white rounded-2xl border border-slate-100 text-slate-400">
        Boş orunlar tapylmady
      </div>
    </div>

    <div 
      v-if="selectedFlight" 
      class="fixed inset-0 bg-slate-900/40 backdrop-blur-sm flex items-center justify-center p-4 z-50 overflow-y-auto"
    >
      <div class="bg-white w-full max-w-lg p-6 rounded-2xl shadow-2xl border border-slate-100 my-8">
        <h3 class="text-xl font-bold mb-1">Billet tasyklamak</h3>
        
        <div class="bg-indigo-50/60 p-3 rounded-xl border border-indigo-100 mb-4 text-sm text-indigo-900">
          <p>Reýs<span class="font-semibold">{{ selectedFlight.flight_number }}</span></p>
          <p>1 adamlyk orun bahasy<span class="font-bold text-emerald-600">${{ selectedFlight.price || 0 }}</span></p>
          <p>Boş orunlar<span class="font-bold">{{ selectedFlight.empty_seats }}</span></p>
        </div>

        <div class="bg-slate-50 p-4 rounded-xl border border-slate-200 mb-4 flex items-center justify-between">
          <div>
            <span class="font-medium text-sm text-slate-700 block">Orun ýeri</span>
            <span class="text-xs text-slate-400">Ütgetmek üçin + - basyň</span>
          </div>
          <div class="flex items-center gap-3">
            <button 
              @click="updatePassengersCount(-1)" 
              :disabled="bookingForm.passengers_count <= 1"
              class="w-9 h-9 rounded-lg bg-white border border-slate-300 flex items-center justify-center font-bold text-slate-700 hover:bg-slate-100 disabled:opacity-50 cursor-pointer"
            >
              -
            </button>
            <span class="font-bold text-lg w-6 text-center">{{ bookingForm.passengers_count }}</span>
            <button 
              @click="updatePassengersCount(1)" 
              :disabled="bookingForm.passengers_count >= selectedFlight.empty_seats"
              class="w-9 h-9 rounded-lg bg-white border border-slate-300 flex items-center justify-center font-bold text-slate-700 hover:bg-slate-100 disabled:opacity-50 cursor-pointer"
            >
              +
            </button>
          </div>
        </div>

        <div class="space-y-4 max-h-64 overflow-y-auto pr-1">
          <div 
            v-for="(passenger, index) in bookingForm.passengers" 
            :key="index"
            class="p-4 rounded-xl border border-slate-200 bg-slate-50/50 space-y-3"
          >
            <h4 class="text-xs font-bold uppercase tracking-wider text-indigo-600">
              Ýolagçy No{{ index + 1 }}
            </h4>
            
            <div>
              <label class="block text-xs font-medium text-slate-700 mb-1">Ýolagçyn A.F.O</label>
              <input 
                v-model="passenger.full_name" 
                type="text" 
                placeholder="Ýolagçyn A.F.O" 
                class="w-full px-3 py-2 text-sm border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:outline-none bg-white"
              />
            </div>

            <div>
              <label class="block text-xs font-medium text-slate-700 mb-1">Telefon nomer</label>
              <input 
                v-model="passenger.phone" 
                type="text" 
                placeholder="+993..." 
                class="w-full px-3 py-2 text-sm border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:outline-none bg-white"
              />
            </div>
          </div>
        </div>

        <div v-if="totalPrice > 0" class="mt-4 pt-3 border-t border-slate-100 flex items-center justify-between">
          <span class="font-medium text-slate-600">Jemi baha({{ bookingForm.passengers_count }} ýer):</span>
          <span class="text-xl font-extrabold text-emerald-600">${{ totalPrice }}</span>
        </div>

        <div class="flex justify-end gap-3 mt-6">
          <button 
            @click="closeModal" 
            class="px-4 py-2 rounded-xl border border-slate-300 text-slate-700 hover:bg-slate-50 cursor-pointer font-medium text-sm"
          >
            Otmen
          </button>
          <button 
            @click="bookTicket" 
            class="px-5 py-2 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white font-medium text-sm cursor-pointer shadow-md shadow-indigo-200"
          >
            Tassyklamak(${{ totalPrice }})
          </button>
        </div>
      </div>
    </div>
  </div>
</template>