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
/* <style>部分は変更ありません */
.form-section { border: 1px solid #eee; padding: 20px; border-radius: 8px; }
.form-group { margin-bottom: 15px; }
label { display: block; margin-bottom: 5px; font-weight: bold; }
input, select { width: 95%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
.button-group { display: flex; gap: 10px; }
.button-group button { flex-grow: 1; padding: 10px; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; color: white; background-color: #4285F4; }
.button-group button:hover { opacity: 0.9; }
.button-group button:disabled { background-color: #aaa; cursor: not-allowed; }
.cancel-btn { background-color: #777; }
.status-box { margin-top: 15px; padding: 10px; border-radius: 4px; font-weight: bold; }
.status-box.error { background-color: #f8d7da; color: #721c24; }
.status-box:not(.error) { background-color: #d4edda; color: #155724; }
</style>