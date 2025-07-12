<script setup>
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

<style scoped>
main {
  max-width: 1000px; /* 全体の幅をさらに広げました */
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: #333;
}
h1 {
  text-align: center;
  font-size: 2.5em;
  margin-bottom: 30px;
  color: #42b883;
}

/* ★★★ flexboxを使ったレイアウト指定 ★★★ */
.form-container {
  display: flex;
  justify-content: center; /* 中央寄せに変更 */
  align-items: flex-start;
  gap: 40px; /* 隙間 */
  flex-wrap: wrap; /* 画面が狭い時に折り返す */
}

/* ★★★ :deep()を使って子コンポーネントの幅を指定 ★★★ */
.form-container > :deep(.form-section) {
  flex-basis: 420px; /* 基本の幅を420pxに */
}

.fixed-cost-container {
  margin-top: 40px;
  padding-top: 40px;
  border-top: 1px solid #eee;
}
</style>