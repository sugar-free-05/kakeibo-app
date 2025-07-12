<script setup>
import { ref } from 'vue';
import Auth from './components/Auth.vue';
import KakeiboApp from './components/KakeiboApp.vue';
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

const session = ref(null);
onMounted(() => {
  supabase.auth.getSession().then(({ data }) => {
    session.value = data.session;
  });
  supabase.auth.onAuthStateChange((_event, _session) => {
    session.value = _session;
  });
});
</script>

<template>
  <div>
    <KakeiboApp v-if="session" :session="session" :supabase="supabase" />
    <Auth v-else :supabase="supabase" />
  </div>
</template>

<style>
/* アプリ全体の基本設定 */
:root {
  --c-brand: #2a9d8f;
  --c-text-1: #2c3e50;
  --c-text-2: #475569;
  --c-bg: #f9fafb; /* 背景を少しグレーに */
  --c-border: #e5e7eb;
  --c-danger: #ef4444;
  --c-edit: #3b82f6;
}
body {
  margin: 0;
  background-color: var(--c-bg);
  font-family: "Hiragino Kaku Gothic ProN", "Hiragino Sans", Meiryo, sans-serif;
  color: var(--c-text-1);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>