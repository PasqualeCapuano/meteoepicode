<template>
	<v-container>
		<v-card class="mx-auto" max-width="800" elevation="3">
			<v-card-text>
				<v-row align="center" class="search-row">
					<v-col cols="12">
						<v-autocomplete
							v-model="selectedCity"
							:items="cities"
							:loading="isSearching"
							:search-input.sync="searchQuery"
							@update:search="searchCities"
							@update:model-value="onCitySelect"
							density="comfortable"
							variant="outlined"
							label="Inserisci il nome della città"
							hide-details
							class="search-input"
							item-title="name"
							item-value="id"
							return-object
							clearable
							hide-no-data
							:menu-props="{ maxHeight: 300 }"
						>
							<template v-slot:item="{ props, item }">
								<v-list-item v-bind="props">
									<v-list-item-subtitle>
										{{ item.raw.country }}
									</v-list-item-subtitle>
								</v-list-item>
							</template>
						</v-autocomplete>
					</v-col>
				</v-row>

				<v-alert
					v-if="error"
					type="error"
					variant="tonal"
					:text="error"
					class="mb-4 mt-4"
				></v-alert>

				<div v-if="isLoading" class="d-flex justify-center mt-6">
					<v-progress-circular
						indeterminate
						color="#9c27b0"
						size="50"
					></v-progress-circular>
				</div>

				<div v-if="weatherData && !isLoading" class="weather-container">
					<v-card
						variant="outlined"
						class="weather-card mt-6"
					>
						<v-card-title class="d-flex align-center purple-gradient">
							<span class="weather-title">
								📍 {{ currentCity }}
							</span>
						</v-card-title>

						<v-card-text class="pa-4">
							<v-list>
								<v-list-item
									v-for="(item, index) in weatherItems"
									:key="index"
									class="weather-item mb-2"
								>
									<template v-slot:prepend>
										<span class="weather-emoji">{{ item.emoji }}</span>
									</template>
									<v-list-item-title class="weather-text">
										{{ item.label }}: {{ item.value }}
									</v-list-item-title>
								</v-list-item>
							</v-list>
						</v-card-text>
					</v-card>
					<img 
						:src="getWeatherImage" 
						:alt="weatherData.weatherCode"
						class="weather-background"
					/>
				</div>
			</v-card-text>
		</v-card>
	</v-container>
</template>

<script>
export default {
	name: "Meteo",
	data() {
		return {
			searchQuery: "",
			selectedCity: null,
			cities: [],
			currentCity: "",
			weatherData: null,
			error: null,
			isLoading: false,
			isSearching: false,
			searchTimeout: null,
		};
	},
	computed: {
		weatherItems() {
			if (!this.weatherData) return [];
			return [
				{
					emoji: "🌡️",
					label: "Temperatura",
					value: `${this.weatherData.temperature}°C`
				},
				{
					emoji: this.getWeatherEmoji(this.weatherData.weatherCode),
					label: "Condizioni",
					value: this.weatherData.weatherCode
				},
				{
					emoji: "💨",
					label: "Velocità del vento",
					value: `${this.weatherData.windSpeed} km/h`
				},
				{
					emoji: "💧",
					label: "Umidità",
					value: `${this.weatherData.humidity}%`
				}
			];
		},
		getWeatherImage() {
			if (!this.weatherData) return '';
			
			const weatherType = this.weatherData.weatherCode.toLowerCase();
			
			if (weatherType.includes('sereno') || weatherType.includes('prevalentemente sereno')) {
				return new URL('../assets/weather/sunny.png', import.meta.url).href;
			} else if (weatherType.includes('nuvoloso') || weatherType.includes('nebbia')) {
				return new URL('../assets/weather/cloudy.png', import.meta.url).href;
			} else if (weatherType.includes('pioggia') || weatherType.includes('pioviggine') || 
						weatherType.includes('temporale') || weatherType.includes('neve')) {
				return new URL('../assets/weather/rainy.png', import.meta.url).href;
			}
			
			// Default to sunny
			return new URL('../assets/weather/sunny.png', import.meta.url).href;
		}
	},
	methods: {
		async searchCities(query) {
			if (!query) {
				this.cities = [];
				return;
			}

			if (this.searchTimeout) {
				clearTimeout(this.searchTimeout);
			}

			this.searchTimeout = setTimeout(async () => {
				try {
					this.isSearching = true;
					const geocodingUrl = new URL('https://geocoding-api.open-meteo.com/v1/search');
					geocodingUrl.searchParams.append('name', query);
					geocodingUrl.searchParams.append('count', '5');
					geocodingUrl.searchParams.append('language', 'it');
					geocodingUrl.searchParams.append('format', 'json');

					const response = await fetch(geocodingUrl);
					const data = await response.json();

					if (data.results) {
						this.cities = data.results.map(city => ({
							name: city.name,
							country: `${city.admin1 ? `${city.admin1}, ` : ''}${city.country}`,
							id: `${city.latitude},${city.longitude}`,
							latitude: city.latitude,
							longitude: city.longitude
						}));
					}
				} catch (error) {
					console.error('Error fetching cities:', error);
				} finally {
					this.isSearching = false;
				}
			}, 300);
		},

		onCitySelect(city) {
			if (city) {
				this.searchLocation(city);
			}
		},

		async searchLocation(city = this.selectedCity) {
			if (!city) {
				this.error = "Per favore seleziona una città";
				return;
			}

			this.isLoading = true;
			this.error = null;
			this.weatherData = null;

			try {
				const { latitude, longitude } = city;
				this.currentCity = `${city.name}, ${city.country}`;

				const weatherUrl = new URL('https://api.open-meteo.com/v1/forecast');
				weatherUrl.searchParams.append('latitude', latitude);
				weatherUrl.searchParams.append('longitude', longitude);
				weatherUrl.searchParams.append('current', 'temperature_2m,relative_humidity_2m,weather_code,wind_speed_10m');

				const weatherResponse = await fetch(weatherUrl);

				if (!weatherResponse.ok) {
					throw new Error('Errore nel recupero dei dati meteo');
				}

				const weatherData = await weatherResponse.json();

				if (!weatherData.current) {
					throw new Error('Dati meteo non validi');
				}

				this.weatherData = {
					temperature: weatherData.current.temperature_2m,
					humidity: weatherData.current.relative_humidity_2m,
					weatherCode: this.getWeatherDescription(weatherData.current.weather_code),
					windSpeed: weatherData.current.wind_speed_10m,
				};
			} catch (err) {
				console.error('Errore meteo:', err);
				this.error = err.message || "Errore nel recupero dei dati meteo";
			} finally {
				this.isLoading = false;
			}
		},
		getWeatherDescription(code) {
			const weatherCodes = {
				0: "Cielo sereno",
				1: "Prevalentemente sereno",
				2: "Parzialmente nuvoloso",
				3: "Nuvoloso",
				45: "Nebbia",
				48: "Nebbia con brina",
				51: "Pioviggine leggera",
				53: "Pioviggine moderata",
				55: "Pioviggine intensa",
				61: "Pioggia leggera",
				63: "Pioggia moderata",
				65: "Pioggia forte",
				71: "Neve leggera",
				73: "Neve moderata",
				75: "Neve forte",
				95: "Temporale"
			};
			return weatherCodes[code] || "Sconosciuto";
		},
		getWeatherEmoji(description) {
			const emojiMap = {
				'Cielo sereno': '☀️',
				'Prevalentemente sereno': '🌤️',
				'Parzialmente nuvoloso': '⛅',
				'Nuvoloso': '☁️',
				'Nebbia': '🌫️',
				'Nebbia con brina': '🌫️',
				'Pioviggine leggera': '🌦️',
				'Pioviggine moderata': '🌧️',
				'Pioviggine intensa': '🌧️',
				'Pioggia leggera': '🌦️',
				'Pioggia moderata': '🌧️',
				'Pioggia forte': '⛈️',
				'Neve leggera': '🌨️',
				'Neve moderata': '��️',
				'Neve forte': '❄️',
				'Temporale': '⛈️'
			};
			return emojiMap[description] || '❓';
		}
	},
};
</script>

<style scoped>
.search-row {
	margin-bottom: 16px;
}

.search-input {
	border-radius: 8px;
}

.purple-gradient {
	color: #4a148c !important;
	padding: 16px !important;
	border-bottom: 2px solid #000000;
}

.weather-container {
	position: relative;
	min-height: 300px;
	display: flex;
	align-items: center;
	gap: 30px;
}

.weather-background {
	position: static;
	height: 250px;
	width: auto;
	opacity: 0.9;
	order: 2;
	transform: none;
}

.weather-card {
	flex: 1;
	max-width: 500px;
	background: white !important;
	backdrop-filter: none;
	order: 1;
	border: 2px solid #000000 !important;
	border-radius: 12px;
	overflow: hidden;
}

.weather-title {
	font-size: 1.5rem;
	font-weight: 600;
	text-shadow: none;
}

.weather-item {
	background-color: #f5f5f5;
	border-radius: 8px;
	margin-bottom: 8px;
	transition: all 0.3s ease;
}

.weather-item:hover {
	background-color: #f3e5f5;
	transform: translateX(4px);
}

.weather-emoji {
	font-size: 1.8rem;
	margin-right: 12px;
}

.weather-text {
	font-size: 1.1rem;
	color: #424242;
}

:deep(.v-btn.purple) {
	background-color: #9c27b0 !important;
	color: white !important;
	box-shadow: 0 3px 5px rgba(156, 39, 176, 0.2);
}

:deep(.v-btn.purple:hover) {
	background-color: #7b1fa2 !important;
	box-shadow: 0 4px 8px rgba(156, 39, 176, 0.3);
	transform: translateY(-1px);
}

:deep(.v-text-field .v-input__append) {
	padding-top: 6px !important;
	margin-inline-start: 0px !important;
}

:deep(.v-autocomplete .v-field__input) {
	padding-top: 0;
}

:deep(.v-list-item-subtitle) {
	font-size: 0.8rem;
	color: #666;
}

/* Hide the default dropdown icon */
:deep(.v-field__append-inner .v-icon) {
	display: none !important;
}
</style>
