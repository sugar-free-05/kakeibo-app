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
const allEntries = ref([]); // ★ すべてのデータを保持する変数
const isLoading = ref(true);
const errorMessage = ref('');
const showContent = ref(false);

// ★★★ 月別フィルター用の新しい変数 ★★★
const availableMonths = ref([]); // 選択肢となる年月のリスト
const selectedMonth = ref('');   // ユーザーが選択した年月

// ★★★ 選択された月のデータだけを絞り込む算出プロパティ ★★★
const filteredEntries = computed(() => {
  if (!selectedMonth.value) {
    return allEntries.value; // 何も選択されていなければ全データを返す
  }
  return allEntries.value.filter(entry => entry.date.startsWith(selectedMonth.value));
});

// ★★★ 合計金額は、絞り込まれたデータを元に計算する ★★★
const totalAmount = computed(() => {
  return filteredEntries.value.reduce((sum, entry) => sum + entry.amount, 0);
});

// ★★★ グラフも、絞り込まれたデータを元に計算する ★★★
const chartData = computed(() => {
    if (!filteredEntries.value || filteredEntries.value.length === 0) {
        return { labels: [], datasets: [{ data: [] }] };
    }
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

onMounted(async () => {
  try {
    isLoading.value = true;
    showContent.value = false;
    const { data, error } = await supabase.from('database').select('*').order('date', { ascending: false });
    if (error) throw error;
    allEntries.value = data; // ★ 全データを allEntries に保存

    // ★★★ 年月の選択肢を生成する処理 ★★★
    const months = new Set(data.map(entry => entry.date.slice(0, 7))); // 'YYYY-MM' の形式で取得
    availableMonths.value = Array.from(months);
    // 最新の月をデフォルトで選択状態にする
    if (availableMonths.value.length > 0) {
      selectedMonth.value = availableMonths.value[0];
    }

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
  <div class="list-section">
    <h2>入力履歴</h2>
    
    <div v-if="showContent">
      <!-- ★★★ 月別フィルターのドロップダウンを追加 ★★★ -->
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
      
      <!-- ★★★ v-forの対象を filteredEntries に変更 ★★★ -->
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

    <div v-if="isLoading">データを読み込んでいます...</div>
    <div v-if="errorMessage" class="error-box">{{ errorMessage }}</div>
  </div>
</template>

<style scoped>
/* ★★★ フィルター用のスタイルを追加 ★★★ */
.filter-container {
  margin-bottom: 20px;
}
.filter-container label {
  margin-right: 10px;
  font-weight: bold;
}
.filter-container select {
  padding: 5px;
  border-radius: 4px;
}

/* ... 他のスタイルは変更なし ... */
</style>