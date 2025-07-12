<script setup>
import { ref } from 'vue';
import FileUploadForm from './FileUploadForm.vue';
import ManualEntryForm from './ManualEntryForm.vue';
import EntryList from './EntryList.vue';
import FixedCostForm from './FixedCostForm.vue';

// ★ 親(App.vue)から session と supabase を受け取る
const props = defineProps({
  session: Object,
  supabase: Object,
});

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
  <header>
    <h1>Flowee</h1>
    <!-- ★ ログアウトボタンを追加 -->
    <button @click="supabase.auth.signOut()" class="logout-btn">ログアウト</button>
  </header>
  
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
</template>

<style scoped>
header { display: flex; justify-content: space-between; align-items: center; }
.logout-btn { padding: 8px 15px; background-color: #777; color: white; border: none; border-radius: 4px; cursor: pointer; }
.form-container { display: flex; justify-content: center; align-items: flex-start; gap: 40px; flex-wrap: wrap; margin-top: 20px; }
.form-container > :deep(.form-section) { flex-basis: 420px; }
.fixed-cost-container { margin-top: 40px; padding-top: 40px; border-top: 1px solid #eee; }
</style>