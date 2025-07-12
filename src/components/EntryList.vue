<script setup>
// ★ nextTick を追加インポートします
import { ref, onMounted, computed, watch, nextTick } from 'vue';
import { createClient } from '@supabase/supabase-js';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Doughnut } from 'vue-chartjs';

ChartJS.register(ArcElement, Tooltip, Legend);

const supabaseUrl = 'https://jequjznjvmdbbxtwbmhs.supabase.co';
const supabaseKey = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImplcXVqem5qdm1kYmJ4dHdibWhzIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTIyNzIxOTEsImV4cCI6MjA2Nzg0ODE5MX0.uA4lOCIWUciDOswqcf-g7cdZ8PbTvwnTdmoaAOaW5NE';
const supabase = createClient(supabaseUrl, supabaseKey);

const emit = defineEmits(['edit-entry']);
const entries = ref([]);
const isLoading = ref(true);
const errorMessage = ref('');

// ★★★ グラフとリストの表示を制御する新しい変数 ★★★
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
    showContent.value = false; // ★ 最初は非表示にしておく
    const { data, error } = await supabase.from('database').select('*').order('date', { ascending: false });
    if (error) throw error;
    entries.value = data;

    // ★★★ データを取得した後、少し待ってから表示する ★★★
    await nextTick(); // DOMの更新を待つ
    showContent.value = true; // 表示をONにする

  } catch (error) {
    console.error('データ取得エラー:', error);
    errorMessage.value = 'データの読み込みに失敗しました。';
  } finally {
    isLoading.value = false;
  }
});

async function deleteEntry(id) { /* ... 変更なし ... */ }
function handleEditClick(entry) { /* ... 変更なし ... */ }
</script>

<template>
  <div class="list-section">
    <h2>入力履歴</h2>
    
    <!-- ★★★ showContentがtrueの時だけ、中身全体を表示する ★★★ -->
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

    <!-- 読み込み中とエラーの表示 -->
    <div v-if="isLoading">データを読み込んでいます...</div>
    <div v-if="errorMessage" class="error-box">{{ errorMessage }}</div>
  </div>
</template>

<style scoped>
/* スタイル部分は変更なし */
</style>