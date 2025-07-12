<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { createClient } from '@supabase/supabase-js';
import { Chart as ChartJS, ArcElement, Tooltip, Legend } from 'chart.js';
import { Doughnut } from 'vue-chartjs';
import Papa from 'papaparse';

ChartJS.register(ArcElement, Tooltip, Legend);

// ===================================================================
// 1. STATE (すべての状態をここで管理)
// ===================================================================
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

// --- Auth State ---
const session = ref(null);
const authLoading = ref(false);
const email = ref('');
const password = ref('');

// --- Manual Form State ---
const editingId = ref(null);
const entryDate = ref(new Date().toISOString().slice(0, 10));
const content = ref('');
const amount = ref(null);
const selectedCategory = ref('');
const categories = ref([]);
const categoryRules = ref([]);
const formStatusMessage = ref('');
const isFormLoading = ref(false);
const isFormError = ref(false);

// --- Fixed Cost Form State ---
const targetMonth = ref(new Date().toISOString().slice(0, 7));
const isFixedCostLoading = ref(false);
const fixedCostStatusMessage = ref('');
const isFixedCostError = ref(false);

// --- CSV Upload Form State ---
const selectedFile = ref(null);
const isUploading = ref(false);
const uploadStatusMessage = ref('');
const isUploadError = ref(false);

// --- List State ---
const allEntries = ref([]);
const availableMonths = ref([]);
const selectedMonth = ref('');
const isListLoading = ref(true);
const listErrorMessage = ref('');

// ===================================================================
// 2. COMPUTED (状態から派生する値)
// ===================================================================
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
    acc[entry.category] = (acc[entry.category] || 0) + entry.amount; return acc;
  }, {});
  const labels = Object.keys(summary);
  const data = Object.values(summary);
  const backgroundColors = labels.map((_, i) => `hsl(${i * 360 / labels.length}, 70%, 70%)`);
  return { labels: labels, datasets: [{ backgroundColor: backgroundColors, data: data }] };
});
const chartOptions = { responsive: true, maintainAspectRatio: false, plugins: { legend: { position: 'top' } } };

// ===================================================================
// 3. METHODS (状態を変更する関数)
// ===================================================================
const handleLogin = async () => {
  try {
    authLoading.value = true;
    const { error } = await supabase.auth.signInWithPassword({ email: email.value, password: password.value });
    if (error) throw error;
  } catch (error) {
    alert(error.error_description || error.message);
  } finally {
    authLoading.value = false;
  }
};
const handleSignup = async () => {
  try {
    authLoading.value = true;
    const { error } = await supabase.auth.signUp({ email: email.value, password: password.value });
    if (error) throw error;
    alert('確認メールを送信しました！');
  } catch (error) {
    alert(error.error_description || error.message);
  } finally {
    authLoading.value = false;
  }
};
const fetchData = async () => {
  try {
    isListLoading.value = true;
    const { data, error } = await supabase.from('database').select('*').order('date', { ascending: false });
    if (error) throw error;
    allEntries.value = data;
    const months = new Set(data.map(entry => entry.date.slice(0, 7)));
    availableMonths.value = Array.from(months);
    if (availableMonths.value.length > 0 && !selectedMonth.value) {
      selectedMonth.value = availableMonths.value[0];
    }
  } catch (error) {
    listErrorMessage.value = 'データ読み込みに失敗しました。';
  } finally {
    isListLoading.value = false;
  }
};
const fetchCategories = async () => {
  try {
    const { data, error } = await supabase.from('category_rules').select('*');
    if (error) throw error;
    categoryRules.value = data;
    categories.value = [...new Set(data.map(item => item.category))];
  } catch (error) { console.error('カテゴリ取得エラー:', error); }
};
const handleSubmit = async () => {
  if (!entryDate.value || !content.value || !amount.value || !selectedCategory.value) {
    alert('すべての項目を入力してください。'); return;
  }
  isFormLoading.value = true;
  const entryData = { date: entryDate.value, content: content.value, amount: amount.value, category: selectedCategory.value };
  try {
    let error;
    if (editingId.value) {
      const { error: updateError } = await supabase.from('database').update(entryData).eq('id', editingId.value);
      error = updateError;
    } else {
      entryData.input_method = '手入力';
      const { error: insertError } = await supabase.from('database').insert([entryData]);
      error = insertError;
    }
    if (error) throw error;
    cancelEdit();
    await fetchData();
  } catch (error) {
    formStatusMessage.value = 'エラー: ' + error.message; isFormError.value = true;
  } finally {
    isFormLoading.value = false;
  }
};
const cancelEdit = () => {
  editingId.value = null;
  entryDate.value = new Date().toISOString().slice(0, 10);
  content.value = '';
  amount.value = null;
  selectedCategory.value = '';
};
const handleEditClick = (entry) => {
  editingId.value = entry.id;
  entryDate.value = entry.date;
  content.value = entry.content;
  amount.value = entry.amount;
  selectedCategory.value = entry.category;
  window.scrollTo({ top: 0, behavior: 'smooth' });
};
const deleteEntry = async (id) => {
  if (!confirm('本当に削除しますか？')) return;
  try {
    const { error } = await supabase.from('database').delete().eq('id', id);
    if (error) throw error;
    await fetchData();
  } catch (error) {
    alert('削除に失敗しました。');
  }
};
const handleRegisterFixedCosts = async () => {
  if (!targetMonth.value) { alert('年月を入力してください。'); return; }
  if (!confirm(`${targetMonth.value} 分の固定費を登録します。よろしいですか？`)) return;
  isFixedCostLoading.value = true;
  fixedCostStatusMessage.value = '処理を開始します...';
  isFixedCostError.value = false;
  try {
    const startDate = `${targetMonth.value}-01`;
    const year = parseInt(targetMonth.value.split('-')[0]), month = parseInt(targetMonth.value.split('-')[1]);
    const endDate = `${targetMonth.value}-${new Date(year, month, 0).getDate()}`;
    const { data: existing, error: checkError } = await supabase.from('database').select('id').eq('input_method', '固定費自動登録').gte('date', startDate).lte('date', endDate);
    if (checkError) throw checkError;
    if (existing.length > 0) {
      fixedCostStatusMessage.value = `${targetMonth.value} 分の固定費は既に追加されています。`;
      isFixedCostError.value = true; return;
    }
    const { data: costs, error: fetchError } = await supabase.from('fixed_costs').select('*');
    if (fetchError) throw fetchError;
    if (costs.length === 0) { fixedCostStatusMessage.value = '登録する固定費がありません。'; return; }
    const newEntries = costs.map(c => ({
      date: `${targetMonth.value}-${String(c.payment_day).padStart(2, '0')}`,
      content: c.content, amount: c.amount, category: c.category || '固定費', input_method: '固定費自動登録'
    }));
    const { error: insertError } = await supabase.from('database').insert(newEntries);
    if (insertError) throw insertError;
    fixedCostStatusMessage.value = `${newEntries.length}件の固定費を登録しました！`;
    await fetchData();
  } catch (error) {
    fixedCostStatusMessage.value = 'エラー: ' + error.message; isFixedCostError.value = true;
  } finally {
    isFixedCostLoading.value = false;
  }
};
const handleFileChange = (event) => {
  selectedFile.value = event.target.files[0];
  uploadStatusMessage.value = '';
};
const handleUpload = () => {
  if (!selectedFile.value) { alert('ファイルを選択してください。'); return; }
  isUploading.value = true;
  uploadStatusMessage.value = 'ファイルを解析中...';
  isUploadError.value = false;
  Papa.parse(selectedFile.value, {
    header: true, skipEmptyLines: true,
    complete: async (results) => {
      uploadStatusMessage.value = `${results.data.length} 件のデータを登録します...`;
      const newEntries = results.data.map(row => ({
        date: row.日付, content: row.内容, amount: parseInt(String(row.金額 || '0').replace(/,/g, '')),
        category: row.カテゴリ || '未分類', input_method: 'CSVアップロード'
      })).filter(e => e.date && e.content);
      if (newEntries.length === 0) {
        uploadStatusMessage.value = '登録できる有効なデータが見つかりませんでした。'; isUploadError.value = true;
        isUploading.value = false; return;
      }
      try {
        const { error } = await supabase.from('database').insert(newEntries);
        if (error) throw error;
        uploadStatusMessage.value = `${newEntries.length} 件のデータを登録しました！`; isUploadError.value = false;
        document.getElementById('file-upload').value = ''; selectedFile.value = null;
        await fetchData();
      } catch (error) {
        uploadStatusMessage.value = 'エラー: ' + error.message; isUploadError.value = true;
      } finally {
        isUploading.value = false;
      }
    }
  });
};

// ===================================================================
// 4. LIFECYCLE & WATCHERS (ライフサイクルと監視)
// ===================================================================
onMounted(() => {
  supabase.auth.getSession().then(({ data }) => { session.value = data.session; });
  supabase.auth.onAuthStateChange((_event, _session) => { session.value = _session; });
});
watch(session, (newSession) => {
  if (newSession) {
    fetchData();
    fetchCategories();
  } else {
    allEntries.value = [];
  }
}, { immediate: true });
watch(content, (newContent) => {
  if (!newContent || editingId.value) return;
  const foundRule = categoryRules.value.find(rule => newContent.toLowerCase().includes(rule.keyword.toLowerCase()));
  if (foundRule) { selectedCategory.value = foundRule.category; }
});
</script>

<template>
  <div class="app-container">
    <div v-if="session">
      <!-- ログイン後の画面 -->
      <header>
        <h1>Flowee</h1>
        <button @click="supabase.auth.signOut()" class="logout-btn">ログアウト</button>
      </header>
      <main>
        <div class="forms-grid">
          <div class="form-section">
            <h2>{{ editingId ? 'データを編集' : '手動で追加' }}</h2>
            <div class="form-grid">
              <div class="form-group">
                <label for="manual-date">日付</label>
                <input type="date" id="manual-date" v-model="entryDate" />
              </div>
              <div class="form-group">
                <label for="manual-amount">金額</label>
                <input type="number" id="manual-amount" placeholder="0" v-model="amount" />
              </div>
              <div class="form-group content-group">
                <label for="manual-content">内容</label>
                <input type="text" id="manual-content" placeholder="例：コンビニ" v-model="content" />
              </div>
              <div class="form-group category-group">
                <label for="manual-category">カテゴリ</label>
                <select id="manual-category" v-model="selectedCategory">
                  <option disabled value="">選択してください</option>
                  <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
                </select>
              </div>
            </div>
            <div class="button-group">
              <button @click="handleSubmit" :disabled="isFormLoading">{{ editingId ? '更新する' : '手動で追加' }}</button>
              <button v-if="editingId" @click="cancelEdit" class="cancel-btn">キャンセル</button>
            </div>
            <div v-if="formStatusMessage" class="status-box" :class="{ error: isFormError }">{{ formStatusMessage }}</div>
          </div>
          <div class="side-forms">
            <div class="form-section">
              <h2>ファイルで追加</h2>
              <p>CSVファイルをアップロードします。</p>
              <div class="form-group">
                <input type="file" id="file-upload" @change="handleFileChange" accept=".csv" />
              </div>
              <button @click="handleUpload" :disabled="isUploading" class="upload-btn">
                {{ isUploading ? '処理中...' : 'アップロード' }}
              </button>
              <div v-if="uploadStatusMessage" class="status-box" :class="{ error: isUploadError }">{{ uploadStatusMessage }}</div>
            </div>
            <div class="form-section">
              <h2>固定費の登録</h2>
              <div class="form-group">
                <label for="target-month">対象年月</label>
                <input type="month" id="target-month" v-model="targetMonth" />
              </div>
              <button @click="handleRegisterFixedCosts" :disabled="isFixedCostLoading" class="fixed-cost-btn">
                {{ isFixedCostLoading ? '処理中...' : `${targetMonth} 分を登録` }}
              </button>
              <div v-if="fixedCostStatusMessage" class="status-box" :class="{ error: isFixedCostError }">{{ fixedCostStatusMessage }}</div>
            </div>
          </div>
        </div>
        <div class="list-section">
          <h2>入力履歴</h2>
          <div v-if="isListLoading">データを読み込んでいます...</div>
          <div v-else-if="listErrorMessage" class="error-box">{{ listErrorMessage }}</div>
          <div v-else>
             <div class="list-header">
              <div class="filter-container">
                <label for="month-filter">表示月:</label>
                <select id="month-filter" v-model="selectedMonth">
                  <option value="">すべての月</option>
                  <option v-for="month in availableMonths" :key="month" :value="month">{{ month }}</option>
                </select>
              </div>
              <div class="summary-total">
                合計支出: <span>{{ totalAmount.toLocaleString() }} 円</span>
              </div>
            </div>
            <div v-if="filteredEntries.length > 0" class="chart-container">
              <Doughnut :data="chartData" :options="chartOptions" />
            </div>
            <table v-if="filteredEntries.length > 0">
              <thead>
                <tr>
                  <th>日付</th>
                  <th>内容</th>
                  <th>カテゴリ</th>
                  <th class="text-right">金額</th>
                  <th class="text-right">操作</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="entry in filteredEntries" :key="entry.id">
                  <td>{{ entry.date }}</td>
                  <td>{{ entry.content }}</td>
                  <td>{{ entry.category }}</td>
                  <td class="text-right amount">{{ entry.amount.toLocaleString() }} 円</td>
                  <td class="actions">
                    <button @click="handleEditClick(entry)" class="edit-btn">編集</button>
                    <button @click="deleteEntry(entry.id)" class="delete-btn">削除</button>
                  </td>
                </tr>
              </tbody>
            </table>
            <div v-if="filteredEntries.length === 0" class="no-data-message">対象のデータがありません。</div>
          </div>
        </div>
      </main>
    </div>
    <div v-else>
      <!-- ログイン画面 -->
      <div class="auth-form-container">
        <h1>Flowee</h1>
        <p>メールアドレスとパスワードでログインまたはサインアップしてください。</p>
        <div class="form-group">
          <label for="email">メールアドレス</label>
          <input id="email" type="email" placeholder="you@example.com" v-model="email" />
        </div>
        <div class="form-group">
          <label for="password">パスワード</label>
          <input id="password" type="password" v-model="password" />
        </div>
        <div class="button-group">
          <button @click="handleLogin" :disabled="authLoading">{{ authLoading ? '...' : 'ログイン' }}</button>
          <button @click="handleSignup" :disabled="authLoading">{{ authLoading ? '...' : 'サインアップ' }}</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --c-brand: #2a9d8f; --c-brand-light: #e9f5f3; --c-text-1: #2c3e50;
  --c-text-2: #475569; --c-bg: #f8f9fa; --c-bg-soft: #ffffff;
  --c-border: #e5e7eb; --c-danger: #ef4444; --c-edit: #3b82f6;
}
body {
  margin: 0; background-color: var(--c-bg);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: var(--c-text-1);
}
</style>

<style scoped>
.app-container, .auth-form-container { padding: 2rem; max-width: 1200px; margin: 0 auto; }
.auth-form-container { max-width: 400px; margin-top: 5rem; background: var(--c-bg-soft); border-radius: 12px; border: 1px solid var(--c-border); }
header { margin-bottom: 2rem; display: flex; justify-content: space-between; align-items: center; }
h1 { font-size: 2.5rem; font-weight: 700; color: var(--c-brand); margin: 0; }
.auth-form-container h1 { text-align: center; }
.auth-form-container p { color: var(--c-text-2); text-align: center; margin-bottom: 2rem; }
.logout-btn { padding: 0.5rem 1rem; background-color: var(--c-text-2); color: white; border: none; border-radius: 8px; cursor: pointer; }
main { display: grid; grid-template-columns: 1fr; gap: 2rem; }
.forms-grid { display: grid; grid-template-columns: 2fr 1fr; gap: 2rem; }
.side-forms { display: flex; flex-direction: column; gap: 2rem; }
.form-section, .list-section { background: var(--c-bg-soft); padding: 2rem; border-radius: 12px; border: 1px solid var(--c-border); }
h2 { margin-top: 0; margin-bottom: 1.5rem; font-size: 1.25rem; font-weight: 600; color: var(--c-text-1); padding-bottom: 1rem; border-bottom: 1px solid var(--c-border); }
.form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1rem 1.5rem; }
.content-group, .category-group { grid-column: 1 / 3; }
.form-group { margin-bottom: 1rem; }
label { display: block; margin-bottom: 0.5rem; font-weight: 500; color: var(--c-text-2); font-size: 0.9rem; }
input, select { width: 100%; padding: 0.75rem; border: 1px solid var(--c-border); border-radius: 8px; box-sizing: border-box; font-size: 1rem; }
.button-group { display: flex; gap: 1rem; margin-top: 1.5rem; }
.button-group button { flex-grow: 1; padding: 0.75rem; border: none; border-radius: 8px; font-size: 1rem; font-weight: 600; cursor: pointer; color: white; }
.button-group button:disabled { background-color: #ccc; }
.button-group button { background-color: var(--c-brand); }
.cancel-btn { background-color: var(--c-text-2); }
.upload-btn { background-color: var(--c-edit) !important; }
.fixed-cost-btn { background-color: #e67e22 !important; }
.status-box { margin-top: 1rem; padding: 0.75rem; border-radius: 8px; font-size: 0.9rem; }
.status-box.error { background-color: #fef2f2; color: #991b1b; }
.status-box:not(.error) { background-color: #e9f5f3; color: var(--c-brand); }
.list-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem; flex-wrap: wrap; gap: 1rem; }
.filter-container label { margin-right: 0.5rem; }
.summary-total { font-size: 1.25rem; font-weight: 600; }
.summary-total span { color: var(--c-brand); }
.chart-container { max-width: 250px; max-height: 250px; margin: 0 auto 2rem; }
table { width: 100%; border-collapse: collapse; }
th, td { border-bottom: 1px solid var(--c-border); padding: 1rem 0.5rem; text-align: left; }
th { font-size: 0.85rem; font-weight: 600; color: var(--c-text-2); }
tbody tr:hover { background-color: var(--c-bg); }
.text-right { text-align: right; }
.amount { font-weight: 500; }
.actions { display: flex; gap: 0.5rem; justify-content: flex-end; }
.actions button { padding: 0.4rem 0.8rem; border-radius: 6px; border:none; color:white; cursor:pointer; }
.edit-btn { background-color: var(--c-edit); }
.delete-btn { background-color: var(--c-danger); }
.no-data-message { text-align: center; padding: 3rem; color: var(--c-text-2); }
.error-box { color: var(--c-danger); }
@media (max-width: 960px) {
  .forms-grid { grid-template-columns: 1fr; }
}
</style>