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
/* スタイル部分は変更ありません */
.form-section { border: 1px solid #eee; padding: 20px; border-radius: 8px; }
.form-group { margin-bottom: 15px; }
button { width: 100%; padding: 10px; background-color: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
button:hover { background-color: #0056b3; }
</style>