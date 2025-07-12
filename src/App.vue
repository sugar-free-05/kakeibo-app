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

<style>
/* グローバルスタイル */
:root {
  /* ★ カラーテーマをシンプルに再定義 */
  --c-brand: #2a9d8f; /* 落ち着いた緑 */
  --c-brand-light: #e9f5f3;
  --c-text-1: #2c3e50; /* メインのテキスト色 */
  --c-text-2: #475569; /* 少し薄いテキスト色 */
  --c-bg: #ffffff; /* 背景は白 */
  --c-border: #e2e8f0;
  --c-danger: #e53935;
  --c-edit: #1e88e5;
}

body {
  margin: 0;
  background-color: var(--c-bg);
  font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", Meiryo, sans-serif; /* ★ フォントを元に戻す */
  color: var(--c-text-1);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>

<style scoped>
main {
  max-width: 1200px; /* 全体の幅を広めに */
  margin: 0 auto;
  padding: 2rem 1.5rem; /* ★ 余白を多めに */
}
h1 {
  text-align: center;
  font-size: 3rem;
  font-weight: 600; /* 少し太くして存在感を出す */
  margin-bottom: 3rem;
  color: var(--c-brand);
}
.form-container {
  display: flex;
  justify-content: center;
  gap: 2rem;
  flex-wrap: wrap;
  margin-bottom: 3rem;
}
.fixed-cost-container {
  margin-top: 3rem;
  padding-top: 3rem;
  border-top: 1px solid var(--c-border);
}
</style>