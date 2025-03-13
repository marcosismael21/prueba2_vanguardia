<!-- App.vue -->
<template>
  <div class="currency-converter">
    <h1>Conversor de Monedas en Tiempo Real</h1>
    
    <div class="error-message" v-if="error">
      {{ error }}
    </div>
    
    <div class="converter-container">
      <div class="input-group">
        <label for="amount">Cantidad</label>
        <input 
          type="number" 
          id="amount" 
          v-model="amount" 
          placeholder="Ingrese cantidad" 
          min="0"
          @input="calculateConversion"
        />
      </div>
      
      <div class="currencies-container">
        <div class="select-group">
          <label for="fromCurrency">De</label>
          <select id="fromCurrency" v-model="fromCurrency" @change="calculateConversion">
            <option v-for="currency in currencies" :key="currency" :value="currency">
              {{ currency }}
            </option>
          </select>
        </div>
        
        <button class="swap-button" @click="swapCurrencies">
          <span>⇄</span>
        </button>
        
        <div class="select-group">
          <label for="toCurrency">A</label>
          <select id="toCurrency" v-model="toCurrency" @change="calculateConversion">
            <option v-for="currency in currencies" :key="currency" :value="currency">
              {{ currency }}
            </option>
          </select>
        </div>
      </div>
      
      <div class="favorite-currency" v-if="favoriteCurrency">
        <button 
          class="favorite-button" 
          @click="useDefaultConversion"
          title="Usar configuración favorita"
        >
          Convertir a {{ favoriteCurrency }} (Favorito)
        </button>
      </div>
      
      <button 
        class="convert-button" 
        @click="calculateConversion" 
        :disabled="isLoading"
      >
        {{ isLoading ? 'Convirtiendo...' : 'Convertir' }}
      </button>
      
      <div class="result-container" v-if="conversionResult">
        <h2>Resultado</h2>
        <div class="result">
          <span>{{ amount }} {{ fromCurrency }} =</span>
          <span class="converted-amount">{{ conversionResult.toFixed(2) }} {{ toCurrency }}</span>
        </div>
        <div class="rate-info">
          <span>1 {{ fromCurrency }} = {{ conversionRate.toFixed(6) }} {{ toCurrency }}</span>
          <span>Última actualización: {{ lastUpdateTime }}</span>
        </div>
        <div class="favorite-actions">
          <button 
            class="set-favorite-button" 
            @click="saveFavoriteCurrency"
            title="Guardar como conversión favorita"
          >
            {{ favoriteCurrency === toCurrency ? 'Favorita ★' : 'Guardar como favorita ☆' }}
          </button>
        </div>
      </div>
    </div>
    
    <!-- Toggle para mostrar/ocultar el gráfico -->
    <div class="toggle-chart-button">
      <button @click="showChart = !showChart">
        {{ showChart ? 'Ocultar gráfico' : 'Mostrar gráfico de variación' }}
      </button>
    </div>
    
    <!-- Componente de gráfico -->
    <div v-if="showChart" class="chart-container">
      <h2>Evolución del Tipo de Cambio</h2>
      <canvas ref="chartCanvas" height="250"></canvas>
      <div class="chart-info">
        <p>Mostrando {{ fromCurrency }} a {{ toCurrency }} durante los últimos 7 días</p>
      </div>
    </div>
    
    <div class="history-container" v-if="conversionHistory.length > 0">
      <h2>Historial de Conversiones</h2>
      <ul class="history-list">
        <li v-for="(item, index) in conversionHistory" :key="index" class="history-item">
          <div class="history-date">{{ formatDate(item.date) }}</div>
          <div class="history-conversion">
            {{ item.amount }} {{ item.fromCurrency }} = 
            {{ item.result.toFixed(2) }} {{ item.toCurrency }}
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import Chart from 'chart.js/auto';

export default {
  name: 'CurrencyConverter',
  data() {
    return {
      amount: 1,
      fromCurrency: 'EUR',
      toCurrency: 'USD',
      currencies: [],
      conversionRate: 0,
      conversionResult: null,
      conversionHistory: [],
      error: null,
      isLoading: false,
      rates: {},
      lastUpdateTime: '',
      favoriteCurrency: null,
      showChart: false,
      chart: null,
      apiKey: 'd04e912d091c6415dd982dffa759c360'
    }
  },
  watch: {
    showChart(newVal) {
      if (newVal) {
        this.$nextTick(() => {
          this.renderChart();
        });
      }
    },
    fromCurrency() {
      if (this.showChart) {
        this.renderChart();
      }
    },
    toCurrency() {
      if (this.showChart) {
        this.renderChart();
      }
    }
  },
  mounted() {
    this.fetchCurrencies();
    
    // Cargar historial del localStorage
    const savedHistory = localStorage.getItem('conversionHistory');
    if (savedHistory) {
      this.conversionHistory = JSON.parse(savedHistory);
    }
    
    // Cargar moneda favorita del localStorage
    const savedFavorite = localStorage.getItem('favoriteCurrency');
    if (savedFavorite) {
      this.favoriteCurrency = savedFavorite;
    }
  },
  methods: {
    async fetchCurrencies() {
      this.isLoading = true;
      this.error = null;
      
      try {
        // Usando exchangeratesapi.io con tu API key
        const response = await fetch(`http://api.exchangeratesapi.io/v1/latest?access_key=${this.apiKey}`);
        const data = await response.json();
        
        if (data.success) {
          this.rates = data.rates;
          this.currencies = Object.keys(data.rates);
          this.lastUpdateTime = new Date(data.timestamp * 1000).toLocaleString();
          this.calculateConversion();
        } else {
          throw new Error(data.error?.info || 'No se pudieron obtener las tasas de cambio');
        }
      } catch (error) {
        this.error = `Error al cargar datos: ${error.message}`;
        console.error('Error fetching currency data:', error);
        
        // Usar datos de respaldo en caso de error
        this.useFallbackData();
      } finally {
        this.isLoading = false;
      }
    },
    
    useFallbackData() {
      // Datos de respaldo en caso de que la API falle
      this.currencies = ['USD', 'EUR', 'GBP', 'JPY', 'CAD', 'AUD', 'CHF', 'CNY', 'MXN'];
      this.rates = {
        USD: 1.1,
        EUR: 1,
        GBP: 0.85,
        JPY: 162.5,
        CAD: 1.47,
        AUD: 1.67,
        CHF: 0.97,
        CNY: 7.9,
        MXN: 18.6
      };
      this.lastUpdateTime = 'Usando datos de respaldo';
    },
    
    calculateConversion() {
      if (!this.amount || !this.fromCurrency || !this.toCurrency || !this.rates) {
        return;
      }
      
      try {
        // Calcular tasa de conversión
        // Nota: Para exchangeratesapi.io, la moneda base gratuita es EUR
        let rate;
        
        if (this.fromCurrency === 'EUR') {
          rate = this.rates[this.toCurrency];
        } else if (this.toCurrency === 'EUR') {
          rate = 1 / this.rates[this.fromCurrency];
        } else {
          // Convertir a través de EUR como moneda base
          rate = this.rates[this.toCurrency] / this.rates[this.fromCurrency];
        }
        
        this.conversionRate = rate;
        this.conversionResult = this.amount * rate;
        
        // Guardar en el historial
        this.addToHistory();
      } catch (error) {
        this.error = 'Error al calcular la conversión';
        console.error('Calculation error:', error);
      }
    },
    
    swapCurrencies() {
      [this.fromCurrency, this.toCurrency] = [this.toCurrency, this.fromCurrency];
      this.calculateConversion();
    },
    
    addToHistory() {
      // Añadir al historial
      const historyItem = {
        amount: this.amount,
        fromCurrency: this.fromCurrency,
        toCurrency: this.toCurrency,
        rate: this.conversionRate,
        result: this.conversionResult,
        date: new Date()
      };
      
      // Limitar el historial a las últimas 10 conversiones
      this.conversionHistory.unshift(historyItem);
      if (this.conversionHistory.length > 10) {
        this.conversionHistory.pop();
      }
      
      // Guardar en localStorage
      localStorage.setItem('conversionHistory', JSON.stringify(this.conversionHistory));
    },
    
    formatDate(dateString) {
      const date = new Date(dateString);
      return date.toLocaleString();
    },
    
    saveFavoriteCurrency() {
      this.favoriteCurrency = this.toCurrency;
      localStorage.setItem('favoriteCurrency', this.toCurrency);
    },
    
    useDefaultConversion() {
      if (this.favoriteCurrency) {
        this.toCurrency = this.favoriteCurrency;
        this.calculateConversion();
      }
    },
    
    renderChart() {
      if (this.chart) {
        this.chart.destroy();
      }
      
      // Generar datos simulados para el gráfico
      // (la versión gratuita de exchangeratesapi.io no incluye datos históricos)
      const days = 7;
      const labels = [];
      const data = [];
      
      for (let i = days; i >= 0; i--) {
        const date = new Date();
        date.setDate(date.getDate() - i);
        labels.push(date.toLocaleDateString());
        
        // Generar variación aleatoria para simular datos históricos
        const baseRate = this.conversionRate;
        const randomVariation = (Math.random() * 4 - 2) / 100; // ±2%
        data.push(baseRate * (1 + randomVariation));
      }
      
      const ctx = this.$refs.chartCanvas.getContext('2d');
      this.chart = new Chart(ctx, {
        type: 'line',
        data: {
          labels: labels,
          datasets: [{
            label: `${this.fromCurrency} a ${this.toCurrency}`,
            data: data,
            borderColor: '#4a90e2',
            backgroundColor: 'rgba(74, 144, 226, 0.1)',
            borderWidth: 2,
            tension: 0.3,
            fill: true
          }]
        },
        options: {
          responsive: true,
          plugins: {
            tooltip: {
              callbacks: {
                label: function(context) {
                  return `Tasa: ${context.raw.toFixed(6)}`;
                }
              }
            }
          },
          scales: {
            y: {
              ticks: {
                callback: function(value) {
                  return value.toFixed(4);
                }
              }
            }
          }
        }
      });
    }
  }
}
</script>

<style>
.currency-converter {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

h1 {
  text-align: center;
  color: #333;
  margin-bottom: 30px;
}

h2 {
  color: #555;
  font-size: 1.2rem;
  margin-bottom: 15px;
}

.error-message {
  background-color: #ffebee;
  color: #c62828;
  padding: 12px;
  border-radius: 4px;
  margin-bottom: 20px;
  text-align: center;
}

.converter-container {
  background-color: #fff;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
}

.input-group {
  margin-bottom: 20px;
}

.input-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: bold;
  color: #555;
}

input, select {
  width: 100%;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
  box-sizing: border-box;
}

input:focus, select:focus {
  border-color: #4a90e2;
  outline: none;
}

.currencies-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 15px;
  margin-bottom: 20px;
}

.select-group {
  flex: 1;
}

.swap-button {
  background-color: #f0f0f0;
  border: none;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 20px;
  margin-top: 25px;
  transition: background-color 0.3s;
}

.swap-button:hover {
  background-color: #e0e0e0;
}

.convert-button, .favorite-button, .set-favorite-button {
  width: 100%;
  padding: 14px;
  background-color: #4a90e2;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.3s;
  margin-bottom: 20px;
}

.convert-button:hover, .favorite-button:hover, .set-favorite-button:hover {
  background-color: #3a80d2;
}

.convert-button:disabled {
  background-color: #a0c4f0;
  cursor: not-allowed;
}

.favorite-button {
  background-color: #5c6bc0;
  margin-bottom: 15px;
}

.favorite-button:hover {
  background-color: #4c5ab0;
}

.set-favorite-button {
  background-color: #ff9800;
  margin: 10px 0 0 0;
  font-size: 14px;
  padding: 10px;
}

.set-favorite-button:hover {
  background-color: #f57c00;
}

.result-container {
  background-color: #f9f9f9;
  padding: 20px;
  border-radius: 6px;
  text-align: center;
}

.result {
  margin-bottom: 10px;
  font-size: 18px;
}

.converted-amount {
  font-weight: bold;
  font-size: 22px;
  color: #4a90e2;
  margin-left: 8px;
}

.rate-info {
  font-size: 14px;
  color: #777;
  display: flex;
  flex-direction: column;
  gap: 5px;
  margin-bottom: 10px;
}

.toggle-chart-button {
  text-align: center;
  margin-bottom: 20px;
}

.toggle-chart-button button {
  background-color: #4caf50;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 20px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.toggle-chart-button button:hover {
  background-color: #43a047;
}

.chart-container {
  background-color: #fff;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
  margin-bottom: 30px;
}

.chart-info {
  text-align: center;
  font-size: 14px;
  color: #777;
  margin-top: 15px;
}

.history-container {
  background-color: #fff;
  padding: 25px;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.history-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.history-item {
  padding: 15px;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.history-item:last-child {
  border-bottom: none;
}

.history-date {
  color: #777;
  font-size: 14px;
}

.history-conversion {
  font-weight: bold;
}

@media (max-width: 600px) {
  .currencies-container {
    flex-direction: column;
  }
  
  .swap-button {
    margin: 10px 0;
    transform: rotate(90deg);
  }
  
  .history-item {
    flex-direction: column;
    align-items: flex-start;
    gap: 5px;
  }
}
</style>