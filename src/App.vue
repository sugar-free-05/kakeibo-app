<script setup>
// スクリプト部分は変更なし
import { ref } from 'vue';
import FileUploadForm from './components/FileUploadForm.vue';
import ManualEntryForm from './components/ManualEntryForm.vue';
import EntryList from './components/EntryList.vue';
import FixedCostForm from './components/FixedCostForm.vue';

const entryToEdit = ref(null);

function handleEditRequest(entry) {
  entryToEdit.value = entry;
  window.scrollTo({ top: 0, behavior: 'smooth' });
}
function handleCancelEdit() {
  entryToEdit.value = null;
}
</script>

<template>
  <!-- template部分は変更なし -->
  <main>
    <h1>Flowee</h1>
    <div class="form-container">
      <FileUploadForm />
      <ManualEntryForm 
        :entry-to-edit="entryToEdit"
        @cancel-edit="handleCancelEdit" 
      />
    </div>
    <div class="fixed-cost-container">
      <FixedCostForm />
    </div>
    <EntryList @edit-entry="handleEditRequest" />
  </main>
</template>

<style> /* ★★★ scoped を外して、グローバルなスタイルを定義 ★★★ */
:root {
  /* ★ カラー変数を定義 */
  --primary-color: #4CAF50; /* メインの緑 */
  --primary-hover-color: #45a049;
  --secondary-color: #FFC107; /* アクセントの黄色 */
  --secondary-hover-color: #f0b400;
  --danger-color: #f44336; /* 削除ボタンの赤 */
  --danger-hover-color: #e53935;
  --edit-color: #2196F3; /* 編集ボタンの青 */
  --edit-hover-color: #1e88e5;
  --text-color: #333;
  --background-color: #f4f4f4;
  --component-bg-color: #ffffff;
  --border-color: #ddd;
}

body {
  margin: 0;
  background-color: var(--background-color); /* 背景色を適用 */
  font-family: 'Helvetica Neue', Arial, 'Hiragino Kaku Gothic ProN', 'Hiragino Sans', Meiryo, sans-serif;
  color: var(--text-color);
}
</style>

<style scoped>
main {
  max-width: 1000px;
  margin: 20px auto;
  padding: 20px;
}
h1 {
  text-align: center;
  font-size: 3em;
  margin-bottom: 40px;
  color: var(--primary-color); /* 変数を使用 */
  font-weight: 300;
}
.form-container {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  gap: 40px;
  flex-wrap: wrap;
}
.form-container > :deep(.form-section) {
  flex-basis: 420px;
  background-color: var(--component-bg-color);
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
.fixed-cost-container {
  margin-top: 40px;
  padding-top: 40px;
  border-top: 1px solid var(--border-color);
}
.fixed-cost-container > :deep(.form-section) {
  background-color: var(--component-bg-color);
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  max-width: 500px;
  margin: 0 auto;
}
</style>