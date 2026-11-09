<script setup>
import { ref } from 'vue'
import jsQR from 'jsqr'

// 状態管理
const isLoading = ref(false)
const errorMsg = ref('')
const decodedUrl = ref('')
const previewUrl = ref('')
const isDragging = ref(false)
const showToast = ref(false)
const fileInput = ref(null)

// トースト表示タイマー
let toastTimer = null

const triggerToast = () => {
  showToast.value = true
  if (toastTimer) clearTimeout(toastTimer)
  toastTimer = setTimeout(() => {
    showToast.value = false
  }, 3000)
}

// クリップボードへの書き込み
const copyToClipboard = async (text) => {
  try {
    await navigator.clipboard.writeText(text)
    triggerToast()
  } catch (err) {
    console.error('クリップボードへの書き込みに失敗しました:', err)
    errorMsg.value = 'クリップボードへの保存に失敗しました。手動でコピーしてください。'
  }
}

// 画像の解析メインロジック
const processImageBlob = (blob) => {
  isLoading.value = true
  errorMsg.value = ''
  decodedUrl.value = ''
  
  // プレビュー用にオブジェクトURLを生成
  if (previewUrl.value) {
    URL.revokeObjectURL(previewUrl.value)
  }
  previewUrl.value = URL.createObjectURL(blob)

  const img = new Image()
  img.src = previewUrl.value

  img.onload = () => {
    try {
      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')
      
      // 画像サイズに合わせる
      canvas.width = img.naturalWidth
      canvas.height = img.naturalHeight
      
      ctx.drawImage(img, 0, 0)
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
      
      // jsQRで解析
      const code = jsQR(imageData.data, imageData.width, imageData.height)
      
      if (code) {
        decodedUrl.value = code.data
        // 成功したら結果をクリップボードに自動書き込み
        copyToClipboard(code.data)
      } else {
        errorMsg.value = '画像からQRコードが検出されませんでした。より鮮明なQRコード画像でお試しください。'
      }
    } catch (err) {
      console.error(err)
      errorMsg.value = '画像の処理中にエラーが発生しました。'
    } finally {
      isLoading.value = false
    }
  }

  img.onerror = () => {
    errorMsg.value = '画像を読み込めませんでした。'
    isLoading.value = false
  }
}

// クリップボードから画像を読み込む
const readClipboard = async () => {
  isLoading.value = true
  errorMsg.value = ''
  
  try {
    // navigator.clipboard.read がサポートされているか確認
    if (!navigator.clipboard || !navigator.clipboard.read) {
      throw new Error('お使いのブラウザはクリップボードの読み取りAPIに対応していません。ドラッグ＆ドロップ機能をご利用ください。')
    }

    const clipboardItems = await navigator.clipboard.read()
    let imageBlob = null

    for (const item of clipboardItems) {
      // image/ で始まるMIMEタイプを探す
      const imageType = item.types.find(type => type.startsWith('image/'))
      if (imageType) {
        imageBlob = await item.getType(imageType)
        break
      }
    }

    if (imageBlob) {
      processImageBlob(imageBlob)
    } else {
      errorMsg.value = 'クリップボードに画像が見つかりませんでした。QRコード画像をコピーした状態でボタンを押してください。'
      isLoading.value = false
    }
  } catch (err) {
    console.error(err)
    if (err.name === 'NotAllowedError') {
      errorMsg.value = 'クリップボードの読み取り許可が拒否されました。ブラウザの設定で許可を与えてください。'
    } else {
      errorMsg.value = err.message || 'クリップボードの読み取り中にエラーが発生しました。'
    }
    isLoading.value = false
  }
}

// ドラッグ＆ドロップのイベントハンドラ
const onDragOver = (e) => {
  e.preventDefault()
  isDragging.value = true
}

const onDragLeave = () => {
  isDragging.value = false
}

const onDrop = (e) => {
  e.preventDefault()
  isDragging.value = false
  
  const files = e.dataTransfer.files
  if (files && files.length > 0) {
    const file = files[0]
    if (file.type.startsWith('image/')) {
      processImageBlob(file)
    } else {
      errorMsg.value = '画像ファイル（PNG/JPEG等）をドロップしてください。'
    }
  }
}

// ファイル選択ダイアログのトリガー
const triggerFileInput = () => {
  fileInput.value.click()
}

const onFileChange = (e) => {
  const files = e.target.files
  if (files && files.length > 0) {
    processImageBlob(files[0])
  }
}
</script>

<template>
  <div class="container">
    <header class="app-header">
      <h1 class="app-title">QR <span class="gradient-text">ClipDecode</span></h1>
      <p class="app-subtitle">クリップボードのQR画像をワンクリックで解析し、URLを自動保存</p>
    </header>

    <div class="glass-panel">
      <!-- メインアクションエリア -->
      <div class="action-section">
        <button 
          @click="readClipboard" 
          class="btn-primary" 
          :disabled="isLoading"
          id="btn-read-clipboard"
        >
          <span v-if="isLoading" class="loader-spinner"></span>
          <svg v-else class="btn-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2" />
            <rect x="8" y="2" width="8" height="4" rx="1" ry="1" />
            <path d="M21 11.5a8.38 8.38 0 0 1-.9 3.8 8.5 8.5 0 0 1-7.6 4.7 8.38 8.38 0 0 1-3.8-.9L3 21l1.9-5.7a8.38 8.38 0 0 1-.9-3.8 8.5 8.5 0 0 1 4.7-7.6 8.38 8.38 0 0 1 3.8-.9h.5a8.48 8.48 0 0 1 8 8v.5z" />
          </svg>
          {{ isLoading ? '解析中...' : 'クリップボードから変換' }}
        </button>

        <p class="divider-text">または</p>

        <!-- ドラッグ＆ドロップエリア -->
        <div 
          @dragover="onDragOver" 
          @dragleave="onDragLeave" 
          @drop="onDrop" 
          @click="triggerFileInput"
          class="dropzone"
          :class="{ 'dropzone-active': isDragging }"
          id="dropzone"
        >
          <input 
            type="file" 
            ref="fileInput" 
            @change="onFileChange" 
            accept="image/*" 
            class="hidden-file-input" 
          />
          <svg class="dropzone-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
            <polyline points="17 8 12 3 7 8" />
            <line x1="12" y1="3" x2="12" y2="15" />
          </svg>
          <p class="dropzone-title">QRコード画像をここにドラッグ＆ドロップ</p>
          <p class="dropzone-subtitle">またはクリックしてファイルを選択</p>
        </div>
      </div>

      <!-- 結果表示セクション -->
      <transition name="fade">
        <div v-if="decodedUrl || errorMsg || previewUrl" class="result-section">
          <!-- プレビュー画像 -->
          <div v-if="previewUrl" class="preview-container">
            <p class="section-label">読み込んだ画像</p>
            <div class="image-wrapper">
              <img :src="previewUrl" alt="QR Code Preview" class="qr-preview" />
            </div>
          </div>

          <!-- 解析成功結果 -->
          <div v-if="decodedUrl" class="success-box">
            <div class="success-header">
              <svg class="success-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="20 6 9 17 4 12" />
              </svg>
              <span>QRコードの解析に成功しました！</span>
            </div>
            <p class="toast-indicator-inline">URLはクリップボードに自動保存されました。</p>
            
            <div class="url-display-container">
              <span class="url-text">{{ decodedUrl }}</span>
              <button @click="copyToClipboard(decodedUrl)" class="btn-copy" title="URLをコピー" id="btn-copy-manual">
                <svg class="copy-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <rect x="9" y="9" width="13" height="13" rx="2" ry="2" />
                  <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1" />
                </svg>
              </button>
            </div>
          </div>

          <!-- エラー表示 -->
          <div v-if="errorMsg" class="error-box">
            <svg class="error-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10" />
              <line x1="12" y1="8" x2="12" y2="12" />
              <line x1="12" y1="16" x2="12.01" y2="16" />
            </svg>
            <p>{{ errorMsg }}</p>
          </div>
        </div>
      </transition>
    </div>

    <!-- トースト通知 -->
    <transition name="fade">
      <div v-if="showToast" class="toast" id="toast-message">
        <svg class="toast-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <polyline points="20 6 9 17 4 12" />
        </svg>
        <span>クリップボードにコピーしました！</span>
      </div>
    </transition>

    <footer class="app-footer-note">
      <p>※ クリップボードからの変換機能は、ブラウザへの読み取り許可が必要です。</p>
      <p>※ セキュアな環境 (localhost または HTTPS) でのみ動作します。</p>
    </footer>
  </div>
</template>

<style scoped>
.container {
  width: 100%;
  max-width: 640px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2rem;
}

/* ヘッダー */
.app-header {
  text-align: center;
}

.app-title {
  font-family: 'Outfit', sans-serif;
  font-size: 3rem;
  font-weight: 700;
  letter-spacing: -0.05em;
  margin-bottom: 0.5rem;
}

.gradient-text {
  background: linear-gradient(135deg, var(--accent-purple) 0%, var(--accent-cyan) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.app-subtitle {
  color: var(--text-secondary);
  font-size: 1rem;
  font-weight: 400;
}

/* メインパネル */
.glass-panel {
  width: 100%;
  background: var(--panel-bg);
  border: 1px solid var(--panel-border);
  border-radius: 24px;
  padding: 2.5rem;
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  box-shadow: var(--shadow-lg), var(--shadow-glow);
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

/* アクションセクション */
.action-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
}

.btn-primary {
  width: 100%;
  background: linear-gradient(135deg, var(--accent-purple) 0%, #8b5cf6 100%);
  color: var(--text-primary);
  border: none;
  border-radius: 16px;
  padding: 1.1rem 2rem;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  box-shadow: 0 4px 20px rgba(168, 85, 247, 0.3);
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.btn-primary:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 8px 30px rgba(168, 85, 247, 0.5);
  background: linear-gradient(135deg, #b86dfc 0%, #9061f9 100%);
}

.btn-primary:active:not(:disabled) {
  transform: translateY(0);
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-icon {
  width: 20px;
  height: 20px;
}

.loader-spinner {
  width: 20px;
  height: 20px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.divider-text {
  color: var(--text-muted);
  font-size: 0.85rem;
  margin: 1.25rem 0;
  position: relative;
  width: 100%;
  text-align: center;
}

.divider-text::before,
.divider-text::after {
  content: '';
  position: absolute;
  top: 50%;
  width: 40%;
  height: 1px;
  background: rgba(255, 255, 255, 0.08);
}

.divider-text::before { left: 0; }
.divider-text::after { right: 0; }

/* ドロップゾーン */
.dropzone {
  width: 100%;
  border: 2px dashed rgba(255, 255, 255, 0.15);
  border-radius: 16px;
  padding: 2.2rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  background: rgba(255, 255, 255, 0.02);
  transition: all 0.3s ease;
}

.dropzone:hover, .dropzone-active {
  border-color: var(--accent-purple);
  background: rgba(168, 85, 247, 0.04);
}

.dropzone-active {
  transform: scale(1.02);
}

.dropzone-icon {
  width: 32px;
  height: 32px;
  color: var(--text-secondary);
  margin-bottom: 0.75rem;
  transition: color 0.3s ease;
}

.dropzone:hover .dropzone-icon {
  color: var(--accent-purple);
}

.dropzone-title {
  font-size: 0.95rem;
  font-weight: 500;
  margin-bottom: 0.25rem;
}

.dropzone-subtitle {
  color: var(--text-secondary);
  font-size: 0.8rem;
}

.hidden-file-input {
  display: none;
}

/* 結果表示セクション */
.result-section {
  border-top: 1px solid rgba(255, 255, 255, 0.08);
  padding-top: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  width: 100%;
}

.section-label {
  color: var(--text-secondary);
  font-size: 0.85rem;
  font-weight: 500;
  margin-bottom: 0.5rem;
  text-align: left;
}

.preview-container {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
}

.image-wrapper {
  background: rgba(0, 0, 0, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 12px;
  padding: 8px;
  max-width: 160px;
  max-height: 160px;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.qr-preview {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 6px;
}

/* 成功ボックス */
.success-box {
  background: var(--success-bg);
  border: 1px solid rgba(16, 185, 129, 0.3);
  border-radius: 16px;
  padding: 1.25rem;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.success-header {
  color: var(--success);
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 1rem;
}

.success-icon {
  width: 18px;
  height: 18px;
}

.toast-indicator-inline {
  font-size: 0.85rem;
  color: var(--text-secondary);
}

.url-display-container {
  display: flex;
  background: rgba(0, 0, 0, 0.25);
  border: 1px solid rgba(255, 255, 255, 0.06);
  border-radius: 12px;
  padding: 0.75rem 1rem;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  margin-top: 0.5rem;
}

.url-text {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.9rem;
  color: var(--accent-cyan);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  flex-grow: 1;
}

.btn-copy {
  background: transparent;
  border: none;
  color: var(--text-secondary);
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 6px;
  border-radius: 8px;
  transition: all 0.2s ease;
}

.btn-copy:hover {
  background: rgba(255, 255, 255, 0.08);
  color: var(--text-primary);
}

.copy-icon {
  width: 18px;
  height: 18px;
}

/* エラーボックス */
.error-box {
  background: var(--error-bg);
  border: 1px solid rgba(244, 63, 94, 0.3);
  border-radius: 16px;
  padding: 1.1rem 1.25rem;
  color: #fda4af;
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  font-size: 0.9rem;
  text-align: left;
  line-height: 1.4;
}

.error-icon {
  width: 20px;
  height: 20px;
  color: var(--error);
  flex-shrink: 0;
  margin-top: 1px;
}

/* フローティングトースト */
.toast {
  position: fixed;
  bottom: 2rem;
  background: rgba(16, 185, 129, 0.95);
  color: white;
  padding: 0.9rem 1.5rem;
  border-radius: 50px;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-weight: 500;
  box-shadow: 0 10px 25px rgba(16, 185, 129, 0.4);
  z-index: 100;
  font-size: 0.95rem;
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
}

.toast-icon {
  width: 18px;
  height: 18px;
}

/* フッター注釈 */
.app-footer-note {
  text-align: center;
  font-size: 0.75rem;
  color: var(--text-muted);
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

@media (max-width: 640px) {
  .glass-panel {
    padding: 1.5rem;
  }
  .app-title {
    font-size: 2.2rem;
  }
}
</style>
