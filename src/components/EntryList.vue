<script setup>
import { ref, onMounted, computed } from 'vue';
import { createClient } from '@supabase/supabase-js';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Doughnut } from 'vue-chartjs';

ChartJS.register(ArcElement, Tooltip, Legend);

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const emit = defineEmits(['edit-entry']);

// --- 状態管理の変数を整理 ---
const allEntries = ref([]);
const availableMonths = ref([]);
const selectedMonth = ref('');
const isLoading = ref(true);
const errorMessage = ref('');

// --- データ取得と初期化を行う非同期関数 ---
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
    console.error('データ取得エラー:', error);
    errorMessage.value = 'データの読み込みに失敗しました。';
  } finally {
    isLoading.value = false;
  }
}

// --- onMountedでデータ取得関数を呼び出す ---
onMounted(fetchData);


// --- 算出プロパティ ---
const filteredEntries = computed(() => {
  if (!selectedMonth.value) return allEntries.value;
  return allEntries.value.filter(entry => entry.date.startsWith(selectedMonth.value));
});

const totalAmount = computed(() => {
  return filteredEntries.value.reduce((sum, entry) => sum + entry.amount, 0);
});

const chartData = computed(() => {
  if (!filteredEntries.value || filteredEntries.value.length === 0) return { labels: [], datasets: [{ data: [] }] };
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

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { position: 'top' } },
};

// --- メソッド ---
async function deleteEntry(id) { /* ... 変更なし ... */ }
function handleEditClick(entry) { /* ... 変更なし ... */ }
</script>

<template>
  <div class="list-section">
    <h2>入力履歴</h2>
    
    <div v-if="isLoading">データを読み込んでいます...</div>
    <div v-if="errorMessage" class="error-box">{{ errorMessage }}</div>
    
    <!-- ★★★ v-if を isLoading が終わった後に変更 ★★★ -->
    <div v-if="!isLoading && !errorMessage">
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
      
      <table v-if="filteredEntries.length > 0">
        <!-- ... thead, tbody は変更なし ... -->
      </table>
      
      <div v-if="filteredEntries.length === 0">対象のデータがありません。</div>
    </div>
  </div>
</template>

<style scoped>
/* ... */
.summary-box span { color: var(--primary-color); } /* 変数に変更 */
.edit-btn { 
  background-color: var(--edit-color); /* 変数に変更 */
}
.edit-btn:hover { 
  background-color: var(--edit-hover-color); /* 変数に変更 */
}
.delete-btn { 
  background-color: var(--danger-color); /* 変数に変更 */
}
.delete-btn:hover { 
  background-color: var(--danger-hover-color); /* 変数に変更 */
}
/* ... */
</style>