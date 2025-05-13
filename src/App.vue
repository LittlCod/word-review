<template>
    <div class="container">
        <h1>单词背诵表</h1>

        <!-- 添加新单词 -->
        <el-form :model="newWord" label-width="100px" class="form">
            <el-form-item label="单词">
                <el-input v-model="newWord.word" />
            </el-form-item>
            <el-form-item label="中文意思">
                <el-input v-model="newWord.meaning" />
            </el-form-item>
            <el-form-item label="背诵日期">
                <el-date-picker v-model="newWord.date" type="date" placeholder="选择背诵日期" />
            </el-form-item>   
            <el-form-item label="首次是否正确">
                <el-switch v-model="newWord.firstCorrect" />
            </el-form-item>
            <el-button type="primary" @click="addWord">添加</el-button>
            <el-button @click="exportJson" style="margin-left: 12px;">导出JSON</el-button>
            <el-button @click="uploadJson" style="margin-left: 12px;">导入JSON</el-button>
            <input type="file" ref="fileInput" accept=".json" @change="handleFile" style="display: none;" />
        </el-form>


        <el-dialog v-model="showMemorize" fullscreen :show-close="false" custom-class="memorize-dialog">
            <template #header>
                <span style="font-size: 18px;">默写模式</span>
            </template>

            <div class="memorize-list">
                <div v-for="(item, index) in todayWords" :key="index" class="memorize-item">
                    {{ index + 1 }}. {{ item.meaning }}
                </div>
            </div>

            <template #footer>
                <el-button type="danger" @click="showMemorize = false">退出默写</el-button>
            </template>
        </el-dialog>


        <!-- 今日复习部分 -->
        <h2>今天要复习的单词 ({{ todayWords.length }}个)</h2>
        <el-button type="primary" @click="showMemorize = true" style="margin-bottom: 16px;">开始默写</el-button>
        <div class="grid">
            <div class="card" v-for="(word, index) in todayWords" :key="index">
                <strong>{{ word.word }}</strong>
                <div>{{ word.meaning }}</div>

                <!-- 复习操作按钮 -->
                <div class="review-buttons">
                    <el-button type="success" icon="Check" circle size="small" @click="markReviewResult(word, true)" />
                    <el-button type="danger" icon="Close" circle size="small" @click="markReviewResult(word, false)" />
                </div>
            </div>
        </div>

        <!-- 未来五天复习单词 -->
        <h2>接下来5天预计要复习的单词</h2>
        <ul class="upcoming-list">
            <li class="upcoming-card" v-for="(item, index) in upcomingWords" :key="index">
                {{ item.date }} ({{ item.words.length }}个)：{{item.words.map(w => w.word).join(', ')}}
            </li>
        </ul>
    </div>
</template>

<script setup>
import { reactive, ref, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import dayjs from 'dayjs'

const newWord = reactive({
    word: '',
    date: dayjs().format('YYYY-MM-DD'),
    meaning: '',
    firstCorrect: true
})

const WORD_KEY = 'vocab_words'
const words = ref([])
const showMemorize = ref(false)

onMounted(() => {
    const saved = localStorage.getItem(WORD_KEY)
    words.value = saved ? JSON.parse(saved) : []
})

// 保存单词到本地存储
function saveWords() {
    localStorage.setItem(WORD_KEY, JSON.stringify(words.value))
}

// 计算复习计划
function generatePlan(baseDate) {
    return [1, 2, 4, 7, 15, 30].map(n => dayjs(baseDate).add(n, 'day').format('YYYY-MM-DD'))
}

// 添加新单词
function addWord() {
    const today = newWord.date
    const plan = newWord.firstCorrect ? generatePlan(today) : []
    const nextReview = newWord.firstCorrect
        ? plan[0]
        : dayjs().add(1, 'day').format('YYYY-MM-DD')
    words.value.push({
        word: newWord.word,
        meaning: newWord.meaning,
        plan,
        nextReview
    })
    saveWords()
    // 重置表单
    Object.assign(newWord, { word: '', meaning: '', firstCorrect: true, date: dayjs().format('YYYY-MM-DD') })
}

// 今天要复习的单词
const todayWords = computed(() => {
    const today = dayjs().format('YYYY-MM-DD')
    return words.value.filter(w => w.nextReview === today)
})

// 计算未来5天的单词计划
const upcomingWords = computed(() => {
    const map = {}
    const today = dayjs()
    for (let i = 1; i <= 5; i++) {
        const date = today.add(i, 'day').format('YYYY-MM-DD')
        map[date] = words.value.filter(w => w.nextReview === date)
    }
    return Object.entries(map).map(([date, words]) => ({ date, words }))
})

// 标记复习结果
function markReviewResult(word, isCorrect) {
    const storedWords = JSON.parse(localStorage.getItem('vocab_words') || '[]')
    const w = storedWords.find(w => w.word === word.word)

    if (!w) return

    if (isCorrect) {
        if (w.plan.length === 0) {
            // 如果空，说明是复习失败重来的，重新生成计划
            const reviewDates = [1, 2, 4, 7, 15, 30].map(d =>
                dayjs().add(d, 'day').format('YYYY-MM-DD')
            )
            w.plan = [...reviewDates]
        } else {
            // 删除已完成的这次复习
            w.plan.shift()
        }

        // 更新下次复习日期
        w.nextReview = w.plan[0] || null
        if (!w.nextReview) {
            // 全部复习完成，移除单词
            const index = storedWords.findIndex(wd => wd.word === word.word)
            storedWords.splice(index, 1)
        }
    } else {
        // 复习失败，清空计划并推迟到明天
        w.plan = []
        w.nextReview = dayjs().add(1, 'day').format('YYYY-MM-DD')
    }

    localStorage.setItem('vocab_words', JSON.stringify(storedWords))
    const saved = localStorage.getItem(WORD_KEY)
    words.value = saved ? JSON.parse(saved) : []
}

// 导出JSON文件
function exportJson() {
    const data = localStorage.getItem('vocab_words') || '[]'
    const blob = new Blob([data], { type: 'application/json' })
    const url = URL.createObjectURL(blob)

    const a = document.createElement('a')
    a.href = url
    a.download = 'vocab_words.json'
    a.click()
    URL.revokeObjectURL(url)
}

const fileInput = ref(null)

// 导入JSON文件
function uploadJson() {
    fileInput.value.click()
}

// 处理文件上传
function handleFile(event) {
    const file = event.target.files[0]
    if (!file) return

    const reader = new FileReader()
    reader.onload = () => {
        try {
            const json = JSON.parse(reader.result)
            if (!Array.isArray(json)) throw new Error('格式错误：应为数组')
            if (!json.every(item => item.word && item.meaning && 'nextReview' in item)) {
                throw new Error('缺少必要字段')
            }
            localStorage.setItem('vocab_words', JSON.stringify(json))
            ElMessage.success('导入成功！')
            const saved = localStorage.getItem(WORD_KEY)
            words.value = saved ? JSON.parse(saved) : []
        } catch (e) {
            ElMessage.error('导入失败，JSON格式错误')
        }
    }
    reader.readAsText(file)
}
</script>

<style scoped>
* {
    padding: 0;
    margin: 0;
}

li {
    list-style: none;
}

h2 {
    margin-bottom: 16px;
}

.container {
    max-width: 900px;
    margin: auto;
    padding: 20px;
    font-family: 'Helvetica Neue', sans-serif;
}

.form {
    margin-bottom: 30px;
}

.grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
    margin-bottom: 30px;
}

.card {
    position: relative;
    text-align: center;
    padding: 20px;
    border-radius: 16px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
    transition: transform 0.2s, background-color 0.2s;
    height: 120px;
}

.card:hover {
    background-color: #f5f7fa;
    transform: scale(1.02);
}

.card strong {
    font-size: 1.2em;
}

.card div:nth-child(2) {
    color: #666;
    font-size: 0.9em;
    margin-top: 6px;
}

.upcoming-list {
    display: grid;
    gap: 16px;
}

.upcoming-card {
    background: #ffffff;
    padding: 16px;
    border-radius: 12px;
    box-shadow: 0 1px 6px rgba(0, 0, 0, 0.06);
}

.upcoming-card h3 {
    margin: 0 0 10px;
    font-size: 1.05em;
    color: #333;
}

.word-line {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    color: #555;
    font-size: 0.95em;
}

.memorize-dialog {
    background: #fdfdfd;
    padding: 40px;
}

.memorize-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 16px;
    margin-bottom: 30px;
    font-size: 1.3em;
    color: #333;
}

.memorize-item {
    padding: 12px 16px;
    background: #f3f4f6;
    border-radius: 10px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.05);
}

.review-buttons {
    position: absolute;
    width: 170px;
    bottom: 16px;
    display: flex;
    justify-content: center;
    gap: 10px;
}
</style>