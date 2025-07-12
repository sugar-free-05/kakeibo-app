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
    <!-- ★★★ :key="entryToEdit" のような形で、キーを渡す（このままでOK） ★★★ -->
    <!-- このファイルは編集機能のために既にキーがありますが、そのまま使います -->
    <EntryList @edit-entry="handleEditRequest" />
  </main>
</template>

<style>
/* スタイルは変更なし */
</style>
<style scoped>
/* スタイルは変更なし */
</style>