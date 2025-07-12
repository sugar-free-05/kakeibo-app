<script setup>
import { ref } from 'vue';
import { createClient } from '@supabase/supabase-js';

// ★★★ 環境変数からSupabaseの情報を読み込むように変更 ★★★
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const targetMonth = ref(new Date().toISOString().slice(0, 7));
const isLoading = ref(false);
const statusMessage = ref('');
const isError = ref(false);

async function handleRegisterFixedCosts() {
  if (!targetMonth.value) {
    alert('年月を入力してください。');
    return;
  }
  if (!confirm(`${targetMonth.value} 分の固定費を登録します。よろしいですか？`)) {
    return;
  }
  isLoading.value = true;
  statusMessage.value = '処理を開始します...';
  isError.value = false;

  try {
    statusMessage.value = '登録済みデータがないか確認中...';
    const startDate = `${targetMonth.value}-01`;
    const year = parseInt(targetMonth.value.split('-')[0]);
    const month = parseInt(targetMonth.value.split('-')[1]);
    const lastDay = new Date(year, month, 0).getDate();
    const endDate = `${targetMonth.value}-${lastDay}`;

    const { data: existingEntries, error: checkError } = await supabase
      .from('database')
      .select('id')
      .eq('input_method', '固定費自動登録')
      .gte('date', startDate)
      .lte('date', endDate);

    if (checkError) throw checkError;
    if (existingEntries.length > 0) {
      isError.value = true;
      statusMessage.value = `${targetMonth.value} 分の固定費は既に追加されています。`;
      isLoading.value = false;
      return;
    }

    statusMessage.value = '固定費リストを取得中...';
    const { data: fixedCosts, error: fetchError } = await supabase.from('fixed_costs').select('*');
    if (fetchError) throw fetchError;
    if (fixedCosts.length === 0) {
      statusMessage.value = '登録する固定費がありません。';
      isLoading.value = false;
      return;
    }

    const newEntries = fixedCosts.map(cost => {
      const paymentDate = `${targetMonth.value}-${String(cost.payment_day).padStart(2, '0')}`;
      return {
        date: paymentDate,
        content: cost.content,
        amount: cost.amount,
        category: cost.category || '固定費', 
        input_method: '固定費自動登録'
      };
    });
    
    statusMessage.value = 'データベースに登録中...';
    const { error: insertError } = await supabase.from('database').insert(newEntries);
    if (insertError) throw insertError;

    statusMessage.value = `${targetMonth.value} 分の固定費 (${newEntries.length}件) を登録しました！ページをリロードして確認してください。`;
    isError.value = false;

  } catch (error) {
    console.error('固定費登録エラー:', error);
    statusMessage.value = 'エラーが発生しました: ' + (error.message || JSON.stringify(error));
    isError.value = true;
  } finally {
    isLoading.value = false;
  }
}
</script>

<template>
  <!-- <template>部分は変更ありません -->
  <div class="form-section">
    <h2>固定費の登録</h2>
    <div class="form-group">
      <label for="target-month">対象年月</label>
      <input type="month" id="target-month" v-model="targetMonth" />
    </div>
    <button @click="handleRegisterFixedCosts" :disabled="isLoading">
      {{ isLoading ? '登録処理中...' : `${targetMonth} 分の固定費を登録` }}
    </button>
    <div v-if="statusMessage" class="status-box" :class="{ error: isError }">
      {{ statusMessage }}
    </div>
  </div>
</template>

<style scoped>
/* ... */
button { 
  background-color: var(--secondary-color); /* 変数に変更 */
}
button:hover { 
  background-color: var(--secondary-hover-color); /* 変数に変更 */
}
/* ... */
</style>