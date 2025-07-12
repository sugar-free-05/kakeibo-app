<script setup>
import { ref } from 'vue';
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = 'https://jequjznjvmdbbxtwbmhs.supabase.co';
const supabaseKey = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImplcXVqem5qdm1kYmJ4dHdibWhzIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTIyNzIxOTEsImV4cCI6MjA2Nzg0ODE5MX0.uA4lOCIWUciDOswqcf-g7cdZ8PbTvwnTdmoaAOaW5NE';
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
    // ★★★ 1. 日付の範囲を正しく指定して、登録済みチェックを行う ★★★
    statusMessage.value = '登録済みデータがないか確認中...';
    
    // 対象月の初日 (YYYY-MM-01)
    const startDate = `${targetMonth.value}-01`;
    // 対象月の末日を計算
    const year = parseInt(targetMonth.value.split('-')[0]);
    const month = parseInt(targetMonth.value.split('-')[1]);
    const lastDay = new Date(year, month, 0).getDate(); // 月の最終日を取得
    const endDate = `${targetMonth.value}-${lastDay}`;

    const { data: existingEntries, error: checkError } = await supabase
      .from('database')
      .select('id')
      .eq('input_method', '固定費自動登録')
      .gte('date', startDate) // ★ gte: startDate 以降
      .lte('date', endDate);   // ★ lte: endDate 以前

    if (checkError) throw checkError;

    if (existingEntries.length > 0) {
      isError.value = true;
      statusMessage.value = `${targetMonth.value} 分の固定費は既に追加されています。`;
      isLoading.value = false;
      return;
    }

    // ★★★ 2. fixed_costsテーブルからリストを取得（ここは変更なし） ★★★
    statusMessage.value = '固定費リストを取得中...';
    const { data: fixedCosts, error: fetchError } = await supabase
      .from('fixed_costs')
      .select('*');

    if (fetchError) throw fetchError;
    if (fixedCosts.length === 0) {
      statusMessage.value = '登録する固定費がありません。';
      isLoading.value = false;
      return;
    }

    // ★★★ 3. データを挿入形式に変換（ここは変更なし） ★★★
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
    
    // ★★★ 4. データを一括で挿入（ここは変更なし） ★★★
    statusMessage.value = 'データベースに登録中...';
    const { error: insertError } = await supabase
      .from('database')
      .insert(newEntries);

    if (insertError) throw insertError;

    statusMessage.value = `${targetMonth.value} 分の固定費 (${newEntries.length}件) を登録しました！ページをリロードして確認してください。`;
    isError.value = false;

  } catch (error) {
    console.error('固定費登録エラー:', error);
    // エラーオブジェクト全体を表示して、より詳細な情報を得る
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
/* <style>部分も変更ありません */
.form-section { border: 1px solid #eee; padding: 20px; border-radius: 8px; }
.form-group { margin-bottom: 15px; }
label { display: block; margin-bottom: 5px; font-weight: bold; }
input { width: 95%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; }
button { width: 100%; padding: 10px; background-color: #f0ad4e; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
button:hover { background-color: #ec971f; }
button:disabled { background-color: #aaa; cursor: not-allowed; }
.status-box { margin-top: 15px; padding: 10px; border-radius: 4px; font-weight: bold; }
.status-box.error { background-color: #f8d7da; color: #721c24; }
.status-box:not(.error) { background-color: #d4edda; color: #155724; }
</style>