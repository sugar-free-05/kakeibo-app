<script setup>
// スクリプト部分は変更ありません
</script>

<template>
  <!-- style="..." を削除した、元のシンプルなdivです -->
  <div class="form-section">
    <h2>ファイルで追加</h2>
    <p>CSVファイル、またはレシートの画像ファイルを選択してください。</p>
    <div class="form-group">
      <input type="file" id="file-upload" />
    </div>
    <button>アップロード</button>
  </div>
</template>

<style scoped>
.form-section {
  background: white;
  padding: 2rem;
  border-radius: 12px;
  border: 1px solid var(--c-border);
}
h2 { margin-top: 0; }
/* ... */
</style>