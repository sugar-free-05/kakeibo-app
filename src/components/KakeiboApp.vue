<script setup>
import { ref } from 'vue';
import FileUploadForm from './FileUploadForm.vue';
import ManualEntryForm from './ManualEntryForm.vue';
import EntryList from './EntryList.vue';
import FixedCostForm from './FixedCostForm.vue';

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
  <div class="kakeibo-app-container">
    <header>
      <h1>Flowee</h1>
      <button @click="supabase.auth.signOut()" class="logout-btn">ログアウト</button>
    </header>
  
    <main>
      <div class="forms-grid">
        <ManualEntryForm 
          :entry-to-edit="entryToEdit"
          @cancel-edit="handleCancelEdit" 
        />
        <div class="side-forms">
          <FileUploadForm />
          <FixedCostForm />
        </div>
      </div>
      <EntryList @edit-entry="handleEditRequest" />
    </main>
  </div>
</template>

<style scoped>
.kakeibo-app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}
header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
}
h1 {
  font-size: 3rem;
  font-weight: 600;
  color: var(--c-brand);
  margin: 0;
}
.logout-btn {
  padding: 0.5rem 1rem;
  background-color: var(--c-text-2);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.logout-btn:hover { background-color: #333; }
.forms-grid {
  display: grid;
  grid-template-columns: 2fr 1fr; /* 2:1の比率で分割 */
  gap: 2rem;
  margin-bottom: 2rem;
}
.side-forms {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}
@media (max-width: 960px) {
  .forms-grid {
    grid-template-columns: 1fr; /* 画面が狭い場合は縦並び */
  }
}
</style>