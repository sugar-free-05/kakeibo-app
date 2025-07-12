<script setup>
import { ref } from 'vue';

const props = defineProps({
  supabase: Object, // 親(App.vue)からsupabaseクライアントを受け取る
});

const loading = ref(false);
const email = ref('');
const password = ref('');

const handleLogin = async () => {
  try {
    loading.value = true;
    const { error } = await props.supabase.auth.signInWithPassword({
      email: email.value,
      password: password.value,
    });
    if (error) throw error;
    alert('ログインしました！');
  } catch (error) {
    alert(error.error_description || error.message);
  } finally {
    loading.value = false;
  }
};

const handleSignup = async () => {
  try {
    loading.value = true;
    const { error } = await props.supabase.auth.signUp({
      email: email.value,
      password: password.value,
    });
    if (error) throw error;
    alert('確認メールを送信しました！メール内のリンクをクリックして登録を完了してください。');
  } catch (error) {
    alert(error.error_description || error.message);
  } finally {
    loading.value = false;
  }
};
</script>

<template>
  <div class="auth-form-container">
    <h1>Floweeへようこそ</h1>
    <p>メールアドレスとパスワードでログインまたはサインアップしてください。</p>
    <div class="form-group">
      <label for="email">メールアドレス</label>
      <input id="email" type="email" placeholder="Your email" v-model="email" />
    </div>
    <div class="form-group">
      <label for="password">パスワード</label>
      <input id="password" type="password" placeholder="Your password" v-model="password" />
    </div>
    <div class="button-group">
      <button @click="handleLogin" :disabled="loading">
        {{ loading ? '読み込み中...' : 'ログイン' }}
      </button>
      <button @click="handleSignup" :disabled="loading">
        {{ loading ? '読み込み中...' : 'サインアップ' }}
      </button>
    </div>
  </div>
</template>

<style scoped>
.auth-form-container { max-width: 400px; margin: 40px auto; padding: 20px; border: 1px solid #eee; border-radius: 8px; }
.form-group { margin-bottom: 15px; }
label { display: block; margin-bottom: 5px; }
input { width: 95%; padding: 8px; }
.button-group { display: flex; gap: 10px; }
button { flex-grow: 1; padding: 10px; }
</style>