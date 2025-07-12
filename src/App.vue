<script setup>
import { ref, onMounted } from 'vue';
import { createClient } from '@supabase/supabase-js';
import Auth from './components/Auth.vue'; // ★ 認証コンポーネント
import KakeiboApp from './components/KakeiboApp.vue'; // ★ 家計簿アプリ本体

// ★★★ SupabaseクライアントはApp.vueで一元管理する ★★★
const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseKey = import.meta.env.VITE_SUPABASE_KEY;
const supabase = createClient(supabaseUrl, supabaseKey);

// ログイン中のセッション（ユーザー情報）を保持する変数
const session = ref(null);

onMounted(() => {
  // 1. 現在のセッションを取得
  supabase.auth.getSession().then(({ data }) => {
    session.value = data.session;
  });

  // 2. ログイン状態の変化を監視
  supabase.auth.onAuthStateChange((_event, _session) => {
    session.value = _session;
  });
});
</script>

<template>
  <div class="container">
    <!-- 
      session（ユーザー情報）があれば、KakeiboApp（家計簿本体）を表示
      なければ、Auth（ログインフォーム）を表示する
    -->
    <KakeiboApp v-if="session" :session="session" :supabase="supabase" />
    <Auth v-else :supabase="supabase" />
  </div>
</template>

<style scoped>
.container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: #333;
}
</style>