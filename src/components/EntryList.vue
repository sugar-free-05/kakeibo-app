<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { createClient } from '@supabase/supabase-js';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Doughnut } from 'vue-chartjs';

ChartJS.register(ArcElement, Tooltip, Legend);

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const emit = defineEmits(['edit-entry']);

const allEntries = ref([]);
const availableMonths = ref([]);
const selectedMonth = ref('');
const isLoading = ref(true);
const errorMessage = ref('');

async function fetchData() {
  try {
    isLoading.value = true;
    const { data, error } = await supabase.from('database').select('*').order('date', { ascending: false });
    if (error) throw error;
    allEntries.value = data;
    const months = new Set(data.map(entry => entry.date.slice(0, 7)));
    availableMonths.value = Array.from(months);
    if (availableMonths.value.length > 0) {
      selectedMonth.value = availableMonths.value[0];
    }
  } catch (error) {
    errorMessage.value = 'データの読み込みに失敗しました。';
  } finally {
    isLoading.value = false;
  }
}

onMounted(fetchData);

const filteredEntries = computed(() => {
  if (!selectedMonth.value) return allEntries.value;
  return allEntries.value.filter(entry => entry.date.startsWith(selectedMonth.value));
});

const totalAmount = computed(() => {
  return filteredEntries.value.reduce((sum, entry) => sum + entry.amount, 0);
});

const chartData = computed(() => {
  if (filteredEntries.value.length === 0) return { labels: [], datasets: [{ data: [] }] };
  const summary = filteredEntries.value.reduce((acc, entry) => {
    acc[entry.category] = (acc[entry.category] || 0) + entry.amount;
    return acc;
  }, {});
  const labels = Object.keys(summary);
  const data = Object.values(summary);
  const backgroundColors = labels.map((_, i) => `hsl(${i * 360 / labels.length}, 70%, 70%)`);
  return {
    labels: labels,
    datasets: [{ backgroundColor: backgroundColors, data: data }]
  };
});

const chartOptions = { /* ... 変更なし ... */ };
async function deleteEntry(id) { /* ... 変更なし ... */ }
function handleEditClick(entry) { /* ... 変更なし ... */ }
</script>

<template>
  <div class="list-section">
    <h2>入力履歴</h2>
    
    <div v-if="isLoading">データを読み込んでいます...</div>
    <div v-else-if="errorMessage" class="error-box">{{ errorMessage }}</div>
    
    <!-- ★★★ すべての v-if/v-show を取っ払った、最もシンプルな構造 ★★★ -->
    <div v-else>
      <div class="filter-container">
        <label for="month-filter">表示月:</label>
        <select id="month-filter" v-model="selectedMonth">
          <option value="">すべての月</option>
          <option v-for="month in availableMonths" :key="month" :value="month">
            {{ month }}
          </option>
        </select>
      </div>

      <div class="summary-box">
        <div class="summary-content">
          <div class="total-amount-container">
            <h3>合計支出: <span>{{ totalAmount.toLocaleString() }} 円</span></h3>
          </div>
          <div class="chart-container">
            <Doughnut :data="chartData" :options="chartOptions" />
          </div>
        </div>
      </div>
      
      <table>
        <thead>
          <!-- ... thead ... -->
        </thead>
        <tbody>
          <tr v-for="entry in filteredEntries" :key="entry.id">
            <!-- ... td ... -->
          </tr>
        </tbody>
      </table>
      
      <div v-if="filteredEntries.length === 0" class="no-data-message">
        対象のデータがありません。
      </div>
    </div>
  </div>
</template>

<style scoped>
/* ... */
.summary-content { display: flex; align-items: center; gap: 2rem; }
.total-amount-container { flex-grow: 1; }
/* ★★★ グラフのサイズを大きくする ★★★ */
.chart-container { 
  width: 220px; /* 180pxから220pxに変更 */
  height: 220px; /* 180pxから220pxに変更 */
}
/* ... */
</style>