<script setup>
import { ref, onMounted, computed, watch, nextTick } from 'vue';
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
    console.error('データ取得エラー:', error);
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

async function deleteEntry(id) {
  if (!confirm('このデータを本当に削除しますか？')) return;
  try {
    const { error } = await supabase.from('database').delete().eq('id', id);
    if (error) throw error;
    location.reload();
  } catch (error) {
    console.error('削除エラー:', error);
    alert('データの削除に失敗しました。');
  }
}

function handleEditClick(entry) {
  emit('edit-entry', entry);
}
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
      
      <!-- ★★★ v-if を filteredEntries.length に変更 ★★★ -->
      <table v-if="filteredEntries.length > 0">
        <thead>
          <tr>
            <th>日付</th>
            <th>内容</th>
            <th>カテゴリ</th>
            <th>金額</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="entry in filteredEntries" :key="entry.id">
            <td>{{ entry.date }}</td>
            <td>{{ entry.content }}</td>
            <td>{{ entry.category }}</td>
            <td class="amount">{{ entry.amount.toLocaleString() }} 円</td>
            <td class="actions">
              <button @click="handleEditClick(entry)" class="edit-btn">編集</button>
              <button @click="deleteEntry(entry.id)" class="delete-btn">削除</button>
            </td>
          </tr>
        </tbody>
      </table>
      
      <div v-if="filteredEntries.length === 0">対象のデータがありません。</div>
    </div>
  </div>
</template>

<style scoped>
/* スタイル部分は変更なし */
.list-section {
  margin-top: 3rem;
  padding-top: 3rem;
  border-top: 1px solid var(--c-border);
}
h2 {
  font-size: 1.5rem;
  margin-top: 0;
  margin-bottom: 1.5rem;
  color: var(--c-text-2);
  font-weight: 600;
}
.filter-container {
  margin-bottom: 1.5rem;
}
.filter-container label {
  margin-right: 0.75rem;
  font-weight: 600;
  color: var(--c-text-2);
}
.filter-container select {
  padding: 0.5rem;
  border-radius: 8px;
  border: 1px solid var(--c-border);
  font-size: 0.9rem;
}
.summary-box {
  background-color: #fafafa;
  padding: 1.5rem;
  margin-bottom: 2rem;
  border-radius: 12px;
}
.summary-box h3 { margin: 0; font-size: 1.25rem; }
.summary-box span { font-size: 1.75rem; color: var(--c-brand); font-weight: 600; }
.summary-content { display: flex; align-items: center; gap: 2rem; }
.total-amount-container { flex-grow: 1; }
.chart-container { width: 180px; height: 180px; }
table {
  width: 100%;
  border-collapse: collapse;
}
th, td {
  border-bottom: 1px solid var(--c-border);
  padding: 1rem 0.5rem;
  text-align: left;
}
th { font-size: 0.85rem; color: var(--c-text-2); font-weight: 600; }
tbody tr:hover { background-color: #f8f9fa; }
.amount { text-align: right; font-weight: 500; }
.actions { display: flex; gap: 0.5rem; justify-content: flex-end; }
.actions button {
  padding: 0.4rem 0.8rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  color: white;
  font-size: 0.85rem;
  transition: opacity 0.2s;
}
.actions button:hover { opacity: 0.85; }
.edit-btn { background-color: var(--c-edit); }
.delete-btn { background-color: var(--c-danger); }
.error-box { color: var(--c-danger); font-weight: 600; }
</style>