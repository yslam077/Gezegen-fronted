<!-- <script setup>
import { ref, computed, onMounted } from 'vue';

const cities = ref([]);
const hotels = ref([]); 
const searchRating = ref('');

const fetchCities = async () => {
    try {
        const response = await fetch('http://127.0.0.1:8000/api/v1/cities', {
            headers: { 'Accept': 'application/json' }
        });
        const data = await response.json();
        cities.value = data.data || data;
    } catch (error) {
        console.error('Error fetching cities:', error);
    }

    try {
        const response = await fetch('http://127.0.0.1:8000/api/hotels', {
            headers: { 'Accept': 'application/json' }
        });
        const data = await response.json();
        hotels.value = data.data || data;
    } catch (error) {
        console.error('Error fetching hotels:', error);
    }
};

const filteredHotels = computed(() => {
    if (!searchRating.value) {
        return hotels.value;
    }
    return hotels.value.filter(hotel => 
        String(hotel.rating).includes(String(searchRating.value))
    );
});

onMounted(fetchCities);
</script>

<template>
    <RouterLink to="/cities">SAher</RouterLink>
    <div class="max-w-4xl mx-auto p-6">
        <div class="relative max-w-sm mb-4">
            <span class="pointer-events-none absolute inset-y-0 left-3 flex items-center text-gray-400">
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="size-4" aria-hidden="true"><circle cx="11" cy="11" r="8"/><path d="m21 21-4.3-4.3"/></svg>
            </span>
            <input 
                type="text" 
                v-model="searchRating" 
                placeholder="Search" 
                class="w-full rounded-lg border border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 outline-none placeholder:text-gray-400 focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 dark:border-gray-600 dark:bg-gray-800 dark:text-white dark:placeholder:text-gray-500 pl-10" 
            />
        </div>

        <div class="text-2xl font-bold mb-4">Hotels</div>
        
        <div class="grid gap-4 md:grid-cols-2">
            <div v-for="hotel in filteredHotels" :key="hotel.id" class="bg-white p-4 rounded shadow border dark:bg-gray-800 dark:border-gray-700 dark:text-white">
                <p><strong>ID:</strong> {{ hotel.id }}</p>
                <p><strong>Ady:</strong> {{ hotel.name }}</p>
                <p><strong>Rating:</strong> {{ hotel.rating }}</p>
            </div>
        </div>
    </div>
    <RouterView/>
</template> -->