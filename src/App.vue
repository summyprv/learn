<template>
  <!-- コンテナ自体の幅を画面幅（%）ベースで固定 -->
  <div class="quiz-container">
    <h1>CSVクイズアプリ</h1>

    <!-- 1. CSVファイル読み込みエリア -->
    <div v-if="quizzes.length === 0" class="file-input-section">
      <p>CSVファイルを選択してください。</p>
      <input type="file" accept=".csv" @change="handleFileUpload" />
    </div>

    <!-- 2. クイズ実行エリア -->
    <div v-else-if="currentIndex < quizzes.length" class="quiz-section">
      <div class="progress">問題: {{ currentIndex + 1 }} / {{ quizzes.length }}</div>
      
      <!-- 原型・意味・問題の表示 -->
      <div class="quiz-header">
        <p class="word-info"><strong>原型:</strong> {{ currentQuiz.baseForm }}</p>
        <p class="word-info"><strong>意味:</strong> {{ currentQuiz.meaning }}</p>
        <!-- 問題文が長すぎても枠をはみ出さないように自動改行 -->
        <h2 class="question-text">{{ currentQuiz.question }}</h2>
      </div>

      <!-- 選択肢A〜Dボタン -->
      <div class="choices-grid">
        <button 
          v-for="(choice, index) in currentChoices" 
          :key="index" 
          @click="selectAnswer(choice.label)"
          class="choice-btn"
        >
          <span class="choice-label">{{ choice.label }}:</span> {{ choice.text }}
        </button>
      </div>

      <!-- ヒントエリア -->
      <div class="hint-section">
        <button @click="showHint = true" class="hint-btn" :disabled="showHint">
          ヒントを表示
        </button>
        <p v-if="showHint" class="hint-text">💡 {{ currentQuiz.hint }}</p>
      </div>
    </div>

    <!-- 3. 結果表示エリア（最終行まで完了後） -->
    <div v-else class="result-section">
      <h2>結果発表</h2>
      <p class="score">正解数: {{ correctCount }} / {{ quizzes.length }}</p>

      <div class="table-wrapper">
        <table class="result-table">
          <thead>
            <tr>
              <th>No.</th>
              <th>問題</th>
              <th>あなたの回答</th>
              <th>正解</th>
              <th>判定</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(quiz, index) in quizzes" :key="index" :class="{ 'row-correct': userAnswers[index] === quiz.answer, 'row-wrong': userAnswers[index] !== quiz.answer }">
              <td>{{ index + 1 }}</td>
              <td class="table-question-cell">{{ quiz.question }}</td>
              <td><strong>{{ userAnswers[index] }}</strong></td>
              <td><strong>{{ quiz.answer }}</strong></td>
              <td>
                <span v-if="userAnswers[index] === quiz.answer" class="badge correct">正解</span>
                <span v-else class="badge wrong">不正解</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <button @click="resetQuiz" class="reset-btn">もう一度挑戦する</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// 状態管理
const quizzes = ref([])
const currentIndex = ref(0)
const userAnswers = ref([])
const showHint = ref(false)

// 現在の問題オブジェクトを算出
const currentQuiz = computed(() => quizzes.value[currentIndex.value] || {})

// 現在の選択肢リストを構造化
const currentChoices = computed(() => {
  if (!currentQuiz.value.question) return []
  return [
    { label: 'A', text: currentQuiz.value.choiceA },
    { label: 'B', text: currentQuiz.value.choiceB },
    { label: 'C', text: currentQuiz.value.choiceC },
    { label: 'D', text: currentQuiz.value.choiceD }
  ]
})

// 正解数をカウント
const correctCount = computed(() => {
  return quizzes.value.filter((quiz, index) => userAnswers.value[index] === quiz.answer).length
})

// CSVファイルを読み込んでパースする処理
const handleFileUpload = (event) => {
  const file = event.target.files?.[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    const text = e.target.result
    // 改行で分割して1行ずつ処理 (空行は除外)
    const lines = text.split(/\r?\n/).filter(line => line.trim() !== '')
    
    const parsedQuizzes = []

    lines.forEach(line => {
      // 1. ".," で「問題」と「それ以降のデータ」に分割する
      const separator = '.,'
      const firstIndex = line.indexOf(separator)
      
      let question = ''
      let restOfLine = ''
      
      if (firstIndex !== -1) {
        question = line.substring(0, firstIndex).trim()
        restOfLine = line.substring(firstIndex + separator.length)
      } else {
        const firstComma = line.indexOf(',')
        question = line.substring(0, firstComma).trim()
        restOfLine = line.substring(firstComma + 1)
      }

      // 問題文の末尾にピリオドを保証
      if (question && !question.endsWith('.') && !question.endsWith('?') && !question.endsWith('!')) {
        question += '.'
      }

      // 2. 残りのデータを通常のカンマで分割する
      const [choiceA, choiceB, choiceC, choiceD, answer, hint, baseForm, meaning] = restOfLine.split(',')
      
      // 各データの空白を除去
      const txtA = choiceA?.trim()
      const txtB = choiceB?.trim()
      const txtC = choiceC?.trim()
      const txtD = choiceD?.trim()
      const txtAnswer = answer?.trim() // 解答文字列

      // 選択肢の配列を作成
      const choices = [txtA, txtB, txtC, txtD]

      // ★ スキップ判定ロジック
      // 解答文字列が空、または選択肢のいずれとも一致しない場合はスキップ
      if (!txtAnswer || !choices.includes(txtAnswer)) {
        return // forEachの次の行の処理へ進む
      }

      // 一致した選択肢のインデックスから、画面内部用の解答ラベル（A, B, C, D）に変換
      const labelMap = ['A', 'B', 'C', 'D']
      const correctLabel = labelMap[choices.indexOf(txtAnswer)]

      // 正常なデータのみ配列に追加
      parsedQuizzes.push({ 
        question: question, 
        choiceA: txtA, 
        choiceB: txtB, 
        choiceC: txtC, 
        choiceD: txtD, 
        answer: correctLabel, // 内部の判定や結果表示は 'A'~'D' のラベルで行う
        hint: hint?.trim(), 
        baseForm: baseForm?.trim(), 
        meaning: meaning?.trim() 
      })
    })

    // パース結果を反映
    quizzes.value = parsedQuizzes
    currentIndex.value = 0
    userAnswers.value = []
    showHint.value = false
  }
  reader.readAsText(file, 'UTF-8')
}

// 選択肢ボタンを押したときの処理
const selectAnswer = (label) => {
  userAnswers.value.push(label)
  showHint.value = false
  currentIndex.value++
}

// クイズのリセット
const resetQuiz = () => {
  quizzes.value = []
  currentIndex.value = 0
  userAnswers.value = []
  showHint.value = false
}
</script>

<style scoped>
.quiz-container {
  /* ★ブラウザの幅（画面幅）に合わせて固定する設定 */
  width: 85vw;             /* 画面の横幅（Viewport Width）の85%に固定 */
  max-width: 1000px;       /* PCなど大きな画面でも広がりすぎない上限（1000px） */
  min-width: 320px;        /* スマホなどの最小幅 */
  margin: 40px auto;       /* 上下に余白、左右は中央寄せ */
  padding: 20px;
  font-family: sans-serif;
  color: #333;
  box-sizing: border-box;
}
.file-input-section, .quiz-section, .result-section {
  background: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 30px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
}
.progress {
  text-align: right;
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 10px;
}
.quiz-header {
  margin-bottom: 25px;
}
.word-info {
  margin: 6px 0;
  color: #555;
  font-size: 1.05rem;
}
.question-text {
  margin-top: 15px;
  font-size: 1.4rem;
  line-height: 1.5;
  word-break: break-word; /* 長い英単語があっても枠内で自動改行させる */
}
.choices-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 12px;
  margin-bottom: 25px;
}
.choice-btn {
  padding: 14px 20px;
  font-size: 1.05rem;
  text-align: left;
  background: #fff;
  border: 1px solid #ccc;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}
.choice-btn:hover {
  background: #eef5ff;
  border-color: #3b82f6;
}
.choice-label {
  font-weight: bold;
  color: #3b82f6;
  margin-right: 8px;
}
.hint-section {
  border-top: 1px dashed #ddd;
  padding-top: 20px;
}
.hint-btn {
  padding: 10px 20px;
  font-size: 0.95rem;
  background: #6b7280;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.hint-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}
.hint-text {
  margin-top: 12px;
  background: #fff3cd;
  padding: 12px;
  border-left: 4px solid #ffc107;
  border-radius: 6px;
  font-size: 1rem;
}
.score {
  font-size: 1.4rem;
  font-weight: bold;
  margin-bottom: 20px;
}
/* 結果表示テーブルがはみ出さないようにスクロール可能にする */
.table-wrapper {
  width: 100%;
  overflow-x: auto;
  margin-bottom: 20px;
}
.result-table {
  width: 100%;
  border-collapse: collapse;
}
.result-table th, .result-table td {
  border: 1px solid #ddd;
  padding: 12px;
  text-align: left;
}
.result-table th {
  background: #eee;
}
.table-question-cell {
  word-break: break-word;
}
.row-correct { background-color: #e6f4ea; }
.row-wrong { background-color: #fce8e6; }
.badge {
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.85rem;
  font-weight: bold;
  color: #fff;
}
.badge.correct { background: #137333; }
.badge.wrong { background: #c5221f; }
.reset-btn {
  width: 100%;
  padding: 14px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 1.1rem;
  cursor: pointer;
}
</style>