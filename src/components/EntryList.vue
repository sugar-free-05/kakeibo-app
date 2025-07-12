<script setup>
import { ref, onMounted, computed, watch, nextTick } from 'vue';
import { createClient } from '@supabase/supabase-js';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Doughnut } from 'vue-chartjs';

ChartJS.register(ArcElement, Tooltip, Legend);

// ★★★ 環境変数からSupabaseの情報を読み込むように変更 ★★★
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const emit = defineEmits(['edit-entry']);
const entries = ref([]);
const isLoading = ref(true);
const errorMessage = ref('');
const showContent = ref(false);
const chartData = ref({ labels: [], datasets: [{ backgroundColor: [], data: [] }] });
const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { position: 'top' } },
};

const totalAmount = computed(() => {
  return entries.value.reduce((sum, entry) => sum + entry.amount, 0);
});

watch(entries, (newEntries) => {
  if (!newEntries || newEntries.length === 0) {
    chartData.value = { labels: [], datasets: [{ data: [] }] };
    return;
  }
  const summary = newEntries.reduce((acc, entry) => {
    acc[entry.category] = (acc[entry.category] || 0) + entry.amount;
    return acc;
  }, {});
  const labels = Object.keys(summary);
  const data = Object.values(summary);
  const backgroundColors = labels.map((_, i) => `hsl(${i * 360 / labels.length}, 70%, 70%)`);
  chartData.value = {
    labels: labels,
    datasets: [{ backgroundColor: backgroundColors, data: data }]
  };
}, { deep: true });

onMounted(async () => {
  try {
    isLoading.value = true;
    showContent.value = false;
    const { data, error } = await supabase.from('database').select('*').order('date', { ascending: false });
    if (error) throw error;
    entries.value = data;
    await nextTick();
    showContent.value = true;
  } catch (error) {
    console.error('データ取得エラー:', error);
    errorMessage.value = 'データの読み込みに失敗しました。';
  } finally {
    isLoading.value = false;
  }
});

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
  <!-- <template>部分は変更ありません -->
  <div class="list-section">
    <h2>入力履歴</h2>
    <div v-if="showContent">
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
      <table v-if="entries.length > 0">
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
          <tr v-for="entry in entries" :key="entry.id">
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
      <div v-if="entries.length === 0">まだデータがありません。</div>
    </div>
    <div v-if="isLoading">データを読み込んでいます...</div>
    <div v-if="errorMessage" class="error-box">{{ errorMessage }}</div>
  </div>
</template>

<style scoped>
/* <style>部分は変更ありません */
.list-section { margin-top: 40px; }
.error-box { color: red; font-weight: bold; }
.summary-box { background-color: #f2f2f2; border: 1px solid #ddd; padding: 15px; margin-bottom: 20px; border-radius: 8px; }
.summary-box h3 { margin: 0; font-size: 1.2em; }
.summary-box span { font-size: 1.5em; color: #42b883; margin-left: 10px; }
.summary-content { display: flex; align-items: center; gap: 20px; }
.total-amount-container { flex-grow: 1; }
.chart-container { width: 200px; height: 200px; }
table { width: 100%; border-collapse: collapse; margin-top: 15px; }
th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
th { background-color: #f2f2f2; }
tr:nth-child(even) { background-color: #f9f9f9; }
.amount { text-align: right; font-weight: bold; }
.actions { display: flex; gap: 5px; }
.edit-btn { background-color: #4285F4; color: white; border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; }
.edit-btn:hover { background-color: #357ae8; }
.delete-btn { background-color: #e53935; color: white; border: none; padding: 5px 10px; border-radius: 4px; cursor: pointer; }
.delete-btn:hover { background-color: #c62828; }
</style>