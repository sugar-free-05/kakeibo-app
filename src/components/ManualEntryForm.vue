<script setup>
import { ref, onMounted, watch } from 'vue';
import { createClient } from '@supabase/supabase-js';

const props = defineProps({
  entryToEdit: Object
});
const emit = defineEmits(['cancel-edit']);
const editingId = ref(null);

// ★★★ 環境変数からSupabaseの情報を読み込むように変更 ★★★
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const entryDate = ref('');
const content = ref('');
const amount = ref(null);
const selectedCategory = ref('');
const categories = ref([]);
const categoryRules = ref([]);
const statusMessage = ref('');
const isLoading = ref(false);
const isError = ref(false);

watch(() => props.entryToEdit, (newEntry) => {
  if (newEntry) {
    editingId.value = newEntry.id;
    entryDate.value = newEntry.date;
    content.value = newEntry.content;
    amount.value = newEntry.amount;
    selectedCategory.value = newEntry.category;
  }
});

onMounted(async () => {
  try {
    const { data, error } = await supabase.from('category_rules').select('*');
    if (error) throw error; 
    categoryRules.value = data;
    const uniqueCategories = [...new Set(data.map(item => item.category))];
    categories.value = uniqueCategories;
  } catch (error) {
    console.error('カテゴリの取得に失敗しました:', error);
    statusMessage.value = 'カテゴリの読み込みに失敗しました。';
    isError.value = true;
  }
});

watch(content, (newContent) => {
  if (!newContent) return;
  const lowerNewContent = newContent.toLowerCase();
  const foundRule = categoryRules.value.find(rule => 
    lowerNewContent.includes(rule.keyword.toLowerCase())
  );
  if (foundRule) {
    selectedCategory.value = foundRule.category;
  }
});

function cancelEdit() {
  editingId.value = null;
  entryDate.value = '';
  content.value = '';
  amount.value = null;
  selectedCategory.value = '';
  statusMessage.value = '';
  isError.value = false;
  emit('cancel-edit');
}

async function handleSubmit() {
  if (!entryDate.value || !content.value || !amount.value || !selectedCategory.value) {
    statusMessage.value = 'すべての項目を入力してください。';
    isError.value = true;
    return;
  }
  isLoading.value = true;
  statusMessage.value = editingId.value ? '更新処理中...' : '追加処理中...';
  isError.value = false;

  const entryData = {
    date: entryDate.value,
    content: content.value,
    amount: amount.value,
    category: selectedCategory.value,
  };

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
    location.reload();
  } catch (error) {
    statusMessage.value = 'エラーが発生しました: ' + error.message;
    isError.value = true;
  } finally {
    isLoading.value = false;
  }
}
</script>

<template>
  <!-- <template>部分は変更ありません -->
  <div class="form-section">
    <h2>{{ editingId ? 'データを編集' : '手動で追加' }}</h2>
    <div class="form-group">
      <label for="manual-date">日付</label>
      <input type="date" id="manual-date" v-model="entryDate" />
    </div>
    <div class="form-group">
      <label for="manual-content">内容</label>
      <input type="text" id="manual-content" placeholder="例：コンビニ" v-model="content" />
    </div>
    <div class="form-group">
      <label for="manual-amount">金額</label>
      <input type="number" id="manual-amount" placeholder="例：500" v-model="amount" />
    </div>
    <div class="form-group">
      <label for="manual-category">カテゴリ</label>
      <select id="manual-category" v-model="selectedCategory">
        <option disabled value="">カテゴリを選択してください</option>
        <option v-for="cat in categories" :key="cat" :value="cat">{{ cat }}</option>
      </select>
    </div>
    <div class="button-group">
      <button @click="handleSubmit" :disabled="isLoading">{{ editingId ? '更新する' : '手動で追加' }}</button>
      <button v-if="editingId" @click="cancelEdit" class="cancel-btn">キャンセル</button>
    </div>
    <div v-if="statusMessage" class="status-box" :class="{ error: isError }">{{ statusMessage }}</div>
  </div>
</template>

<style scoped>
.form-section {
  padding: 1.5rem;
  border: 1px solid var(--c-border);
  border-radius: 12px;
  width: 450px;
}
h2 {
  margin-top: 0;
  margin-bottom: 1.5rem;
  font-size: 1.5rem;
  color: var(--c-text-1);
}
.form-group {
  margin-bottom: 1.25rem;
}
label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 600;
  color: var(--c-text-2);
}
input, select {
  width: 100%;
  padding: 0.75rem;
  border: 1px solid var(--c-border);
  border-radius: 8px;
  box-sizing: border-box; /* paddingを含めて幅を計算 */
  font-size: 1rem;
}
.button-group {
  display: flex;
  gap: 1rem;
  margin-top: 1.5rem;
}
.button-group button {
  flex-grow: 1;
  padding: 0.75rem;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  color: white;
  background-color: var(--c-brand);
  transition: background-color 0.2s;
}
.button-group button:hover {
  background-color: #248a7d;
}
.button-group button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
.cancel-btn {
  background-color: var(--c-text-2);
}
.cancel-btn:hover {
  background-color: #3a4657;
}
.status-box { margin-top: 1rem; padding: 0.75rem; border-radius: 8px; }
.status-box.error { background-color: #fef2f2; color: #991b1b; }
.status-box:not(.error) { background-color: var(--c-brand-light); color: var(--c-brand); }
</style>