<!-- eslint-disable eslint-comments/no-unlimited-disable -->
<script setup generic="T extends any, O extends any">
import vocabulary from './vocabulary'

const CHAPTER_KEY = 'vocabulary_chapter'

const isTrainingModel = ref(false)
const isShowMeaning = ref(false)
const isAutoPlayWordAudio = ref(true)
const isOnlyShowErrors = ref(false)
const isFinishTraining = ref(false)

const trainingStats = ref('')
const keyword = ref('')
const chapters = Object.keys(vocabulary)
const category = ref(localStorage.getItem(CHAPTER_KEY) || chapters[0])

const loaded = ref(false)
const refVocabulary = reactive(vocabulary)

const finishedList = ref([])
// function finishTraining() {
//   finishedList.value = [...flatDisplayWords.value] // ✅ 锁定当前范围
//   isFinishTraining.value = true
// }

function finishTraining() {
  const list = flatDisplayWords.value

  let correct = 0
  let error = 0

  for (const item of list) {
    const value = item.spellValue
    if (!value) continue

    const isCorrect = item.word
      .map(v => v.toLowerCase().trim())
      .includes(value)

    item.spellError = !isCorrect

    if (isCorrect) {
      correct++
    } else {
      error++
    }

    updateProgress(item, isCorrect)
  }

  finishedList.value = [...list]
  isFinishTraining.value = true
  isShowMeaning.value = true

  // ✅ 记录逻辑（核心）
  const cat = category.value

  if (!trainingRecordsMap.value[cat]) {
    trainingRecordsMap.value[cat] = []
  }

  trainingRecordsMap.value[cat].unshift({
    time: Date.now(),
    correct,
    error
  })

  // ✅ 只保留最近10条
  trainingRecordsMap.value[cat] =
    trainingRecordsMap.value[cat].slice(0, 10)

  localStorage.setItem(
    RECORD_KEY,
    JSON.stringify(trainingRecordsMap.value)
  )

  trainingStats.value = calcStats()


}

function formatTime(ts) {
  const d = new Date(ts)
  const pad = n => n.toString().padStart(2, '0')

  return `${d.getFullYear()}年${d.getMonth() + 1}月${d.getDate()}日 `
    + `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`
}

watch(category, (newVal, oldVal) => {
  // console.log(newVal, oldVal)
  localStorage.setItem(CHAPTER_KEY, newVal)

  // ✅ 每次切换章节都初始化
  const words = refVocabulary[newVal].words
  resetTrainingState(words)

  nextTick(() => {
    applyProgressToItems(words)
  })
})

const flatDisplayWords = computed(() => {
  return displayWords.value.flat()
})

function calcStats() {
  let error = 0
  let missing = 0
  let correct = 0


  if (isTrainingModel.value) {

    const list = isFinishTraining.value
      ? finishedList.value
      : flatDisplayWords.value

    for (const item of list) {
      if (item.spellValue) {
        if (item.spellError) {
          error++
        } else {
          correct++
        }
      } else {
        missing++
      }
    }
  }

  const rate = ((correct / (correct + error)) * 100).toFixed(0)

  return `${missing} 个未完成，${correct} 个正确，${error} 个错误  （正确率 ${ rate }%）`
}

onMounted(() => {

  document.addEventListener('keydown', (e) => {
    console.log('🌍 全局keydown:', e.key)
  })

  loaded.value = true

  // 只能同时播放一个音频
  const audioTags = document.getElementsByTagName('audio')
  for (const audio of audioTags) {
    audio.onplay = () => {
      for (const _audio of audioTags) {
        _audio.blur()
        if (audio !== _audio) {
          _audio.pause()
        }
      }
    }
  }
})

onUpdated(() => {
  console.log('🔄 DOM更新了')

  // 音频再切换 SRC 之后需要调用一下 load() 不然看不到效果
  for (const el of document.getElementsByTagName('audio')) {
    el.load()
  }
})

document.addEventListener('keydown', (ev) => {
  // 激活的那个音频可以通过方向键进行快进/退
  // if (['ArrowLeft', 'ArrowRight', ' '].includes(ev.key)) {
  //   ev.preventDefault()
  //   const audioTags = document.getElementsByTagName('audio')
  //   const keyMap = {
  //     ArrowLeft: -5,
  //     ArrowRight: 5,
  //   }
  //   for (const audioTag of audioTags) {
  //     audioTag.blur()
  //     if (keyMap[ev.key]) {
  //       const step = keyMap[ev.key]
  //       audioTag.currentTime = audioTag.currentTime + step
  //       // console.log(step, audioT ag.currentTime)
  //     }
  //     if (ev.key === ' ') {
  //       if (audioTag.paused)
  //         audioTag.play()
  //       else
  //         audioTag.pause()
  //     }
  //   }
  // }
})

let audio = null

function play(audioPath) {
  if (audio) {
    audio.pause()
    audio.currentTime = 0
  }
  audio = document.createElement('audio')
  audio.src = audioPath
  audio.play()
}

function copyText(item) {
  const text = `${item.word} ${item.pos} ${item.meaning}`
  navigator.clipboard.writeText(text)
}

function onInputKeydown(e, item) {
  const {key} = e
  e.stopPropagation()

  const list = flatDisplayWords.value
  const currentIndex = list.findIndex(v => v.id === item.id)

  console.log('👉 keydown:', {
    key,
    id: item.id,
    currentIndex
  })

  // 🔼 上一个
  if (key === 'Enter' && e.shiftKey) {
    e.preventDefault()

    let prevIndex = currentIndex - 1

    while (prevIndex >= 0) {
      if (!list[prevIndex].hiddenByCheck) break
      prevIndex--
    }

    const prevItem = list[prevIndex]

    console.log('👆 上一个:', prevItem)

    if (prevItem) {
      setTimeout(() => {
        document.getElementById(prevItem.id)?.focus()
      }, 50)
    }
  }

  // 🔽 下一个
  else if (key === 'Enter') {
    e.preventDefault()

    let nextIndex = currentIndex + 1

    while (nextIndex < list.length) {
      if (!list[nextIndex].hiddenByCheck) break
      nextIndex++
    }

    const nextItem = list[nextIndex]

    console.log('👇 下一个:', nextItem)

    if (nextItem) {
      setTimeout(() => {
        document.getElementById(nextItem.id)?.focus()
      }, 50)
    } else {
      console.log('🎉 已到最后一个（全局）')
    }
  }

  // 🔊 播音
  if (key === 'ArrowUp') {
    e.preventDefault()
    play(`vocabulary/audio/${category.value}/${item.word[0]}.mp3`)
  }

  // 👇 展示提示
  if (key === 'ArrowDown') {
    e.preventDefault()
    toggleHint(item)
  }

  // // ⬅️ 隐藏并跳下一个（也要改成全局！）
  // if (key === 'ArrowLeft') {
  //   e.preventDefault()
  //
  //   item.hiddenByCheck = true
  //
  //   let nextIndex = currentIndex + 1
  //
  //   while (nextIndex < list.length) {
  //     if (!list[nextIndex].hiddenByCheck) break
  //     nextIndex++
  //   }
  //
  //   const nextItem = list[nextIndex]
  //
  //   if (nextItem) {
  //     setTimeout(() => {
  //       document.getElementById(nextItem.id)?.focus()
  //     }, 50)
  //   }
  // }

  // ➡️ 取消隐藏
  if (key === 'ArrowRight') {
    e.preventDefault()
    item.hiddenByCheck = false
  }
}

function onInputFoucsIn(e, audioPath) {
  if (isAutoPlayWordAudio.value) {
    play(audioPath)
  }
}

function onInputFoucsOut(e, item) {
  const {target} = e
  const spellValue = target.value.toLowerCase().trim()

  console.log('💥 focusout:', {
    id: item.id,
    value: spellValue
  })

  if (spellValue.length < 1) {
    item.spellValue = ''
    item.spellError = false
  } else {
    item.spellValue = spellValue

    // const isCorrect = item.word.map(v => v.toLowerCase().trim()).includes(spellValue)

    // item.spellError = !isCorrect
    //
    // updateProgress(item, isCorrect)   // ✅ 加这一行
  }
  trainingStats.value = calcStats()
}

function getInputStyleClass(item) {
  const cls = {
    error: 'ml-4 bg-red-50 border border-red-500 text-red-900 placeholder-red-700 text-sm rounded-lg focus:ring-red-500 dark:bg-gray-700 focus:border-red-500 inline-block p-2.5 dark:text-red-500 dark:placeholder-red-500 dark:border-red-500',
    normal: 'ml-4 inline-block border w-[100px] border-gray-300 rounded-lg bg-gray-50 p-2.5 text-sm text-gray-900 dark:border-gray-600 focus:border-blue-500 dark:bg-gray-700 dark:text-white focus:ring-blue-500 dark:focus:border-blue-500 dark:focus:ring-blue-500 dark:placeholder-gray-400',
    success: 'ml-4 bg-green-50 border border-green-500 text-green-900 dark:text-green-400 placeholder-green-700 dark:placeholder-green-500 text-sm rounded-lg focus:ring-green-500 focus:border-green-500 inline-block p-2.5 dark:bg-gray-700 dark:border-green-500',
  }
  if (isFinishTraining.value) {
    if (item.spellError) {
      return cls.error
    }
    if (item.spellValue.length > 0 && !item.spellError) {
      return cls.success
    }
  }
  return cls.normal
}

function copyAllError() {
  const list = flatDisplayWords.value   // ✅ 改这里

  const errorWords = list
    .filter(item => item.spellError)
    .map(item => `${item.word} ${item.pos} ${item.meaning}`)

  navigator.clipboard.writeText(errorWords.join('\n\n'))
}

const STORAGE_VERSION = 'v1'
const STORAGE_KEY = `vocabulary_progress_${STORAGE_VERSION}`

const progressMap = ref(
  JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}')
)

function updateProgress(item, isCorrect) {
  const cat = category.value
  const id = item.id

  if (!progressMap.value[cat]) {
    progressMap.value[cat] = {}
  }

  if (!progressMap.value[cat][id]) {
    progressMap.value[cat][id] = {
      wrongCount: 0,
      correctCount: 0,
      lastResult: '',
      lastTime: 0
    }
  }

  const record = progressMap.value[cat][id]

  if (isCorrect) {
    record.correctCount++
  } else {
    record.wrongCount++
  }

  record.lastResult = isCorrect ? 'correct' : 'wrong'
  record.lastTime = Date.now()

  localStorage.setItem(STORAGE_KEY, JSON.stringify(progressMap.value))
}

function applyProgressToItems(words) {

}

function getRecord(item) {
  return progressMap.value[category.value]?.[item.id]
}

function getWrongLevel(item) {
  const record = getRecord(item)
  return record ? record.wrongCount : 0
}

function isHistoricalWrong(item) {
  const record = getRecord(item)
  return record && record.wrongCount > 0
}

const isErrorTrainingMode = ref(false)

const displayWords = computed(() => {
  const words = refVocabulary[category.value].words

  // 👉 扁平化
  const flat = []
  for (const group of words) {
    for (const item of group) {
      flat.push(item)
    }
  }

  // 🟢 非练习模式
  if (!isTrainingModel.value) {
    return words
  }

  // 🔴 错词训练模式（排序）
  if (isErrorTrainingMode.value) {
    const filtered = flat.filter(item => isHistoricalWrong(item))
    return [[...filtered].sort((a, b) => getWrongLevel(b) - getWrongLevel(a))]
  }

  // 🟡 仅错词（不排序）
  if (isOnlyShowErrors.value) {
    const filtered = flat.filter(item => isHistoricalWrong(item))
    return [filtered]
  }

  // 🟢 默认
  return words
})

function resetTrainingState(words) {
  for (const group of words) {
    for (const item of group) {
      item.hiddenByCheck = false
      item.markedRed = false
      item.showExample = false
      item.showExtra = false
      item.showMeaning = false
      item.spellValue = ''
      item.spellError = false
    }
  }

  trainingStats.value = calcStats()
}

watch(isErrorTrainingMode, () => {
  const words = refVocabulary[category.value].words

  resetTrainingState(words)
})

watch(isTrainingModel, (val) => {
  if (val) {
    isShowMeaning.value = false
  }
})

function toggleHint(item) {
  const isOpen = item.showMeaning

  // 👉 统一开关
  item.showMeaning = !isOpen
  item.showExample = !isOpen
  item.showExtra = !isOpen
}

watch(isErrorTrainingMode, () => {
  const words = refVocabulary[category.value].words

  for (const group of words) {
    for (const item of group) {
      item.hiddenByCheck = false
      item.showExample = false
      item.showExtra = false
    }
  }
})

const RECORD_KEY = 'vocabulary_training_records_v2'

const trainingRecordsMap = ref(
  JSON.parse(localStorage.getItem(RECORD_KEY) || '{}')
)

const currentRecords = computed(() => {
  return trainingRecordsMap.value[category.value] || []
})
</script>

<template>
  <div class="px-4 pt-6 2xl:px-0">
    <div class="border border-gray-200 rounded-lg bg-white p-4 shadow-sm dark:border-gray-700 dark:bg-gray-800 sm:p-6">
      <!-- Card header -->
      <div class="items-center justify-between lg:flex">
        <div class="mb-4 lg:mb-0">
          <h3 class="mb-2 text-xl font-bold text-gray-900 dark:text-white">
            雅思词汇真经
          </h3>
          <span class="text-base font-normal text-gray-500 dark:text-gray-400">涵盖雅思必备核心词，逻辑词群记忆法</span>
        </div>
        <div class="items-center sm:flex">
          <div class="flex items-center">
            <select
              v-model="category"
              class="block w-full flex-1 border border-gray-300 rounded-lg bg-gray-50 p-2.5 text-sm text-gray-900 dark:border-gray-600 focus:border-blue-500 dark:bg-gray-700 dark:text-white focus:ring-blue-500 dark:focus:border-blue-500 dark:focus:ring-blue-500 dark:placeholder-gray-400"
            >
              <!-- <option value="">
                全部章节
              </option> -->
              <option v-for="(_, k) in refVocabulary" :key="k" :value="k">
                {{ k }}
              </option>
            </select>

            <label class="ml-2 inline-flex cursor-pointer items-center">
              <input v-model="isTrainingModel" type="checkbox" class="peer sr-only">
              <div
                class="peer relative h-6 w-11 rounded-full bg-gray-200 after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5 after:border after:border-gray-300 dark:border-gray-600 after:rounded-full after:bg-white dark:bg-gray-700 peer-checked:bg-blue-600 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 after:transition-all after:content-[''] peer-checked:after:translate-x-full peer-checked:after:border-white dark:peer-focus:ring-blue-800 rtl:peer-checked:after:-translate-x-full"
              />
              <span class="ms-3 text-sm font-medium text-gray-900 dark:text-gray-300">练习模式</span>
            </label>

            <label v-if="isTrainingModel" class="ml-2 inline-flex cursor-pointer items-center">
              <input v-model="isErrorTrainingMode" type="checkbox" class="peer sr-only">
              <div
                class="peer relative h-6 w-11 rounded-full bg-gray-200
                after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5
                after:border after:border-gray-300 dark:border-gray-600 after:rounded-full
                after:bg-white dark:bg-gray-700 peer-checked:bg-red-600
                peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-red-300
                after:transition-all after:content-['']
                peer-checked:after:translate-x-full peer-checked:after:border-white"
              />
              <span class="ms-3 text-sm font-medium text-gray-900 dark:text-gray-300">
                错词练习
              </span>
            </label>

            <label v-if="isTrainingModel" class="ml-2 inline-flex cursor-pointer items-center">
              <input v-model="isShowMeaning" type="checkbox" class="peer sr-only">
              <div
                class="peer relative h-6 w-11 rounded-full bg-gray-200 after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5 after:border after:border-gray-300 dark:border-gray-600 after:rounded-full after:bg-white dark:bg-gray-700 peer-checked:bg-blue-600 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 after:transition-all after:content-[''] peer-checked:after:translate-x-full peer-checked:after:border-white dark:peer-focus:ring-blue-800 rtl:peer-checked:after:-translate-x-full"
              />
              <span class="ms-3 text-sm font-medium text-gray-900 dark:text-gray-300">释义</span>
            </label>

            <label v-if="isTrainingModel" class="ml-2 inline-flex cursor-pointer items-center">
              <input v-model="isAutoPlayWordAudio" type="checkbox" class="peer sr-only">
              <div
                class="peer relative h-6 w-11 rounded-full bg-gray-200 after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5 after:border after:border-gray-300 dark:border-gray-600 after:rounded-full after:bg-white dark:bg-gray-700 peer-checked:bg-blue-600 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 after:transition-all after:content-[''] peer-checked:after:translate-x-full peer-checked:after:border-white dark:peer-focus:ring-blue-800 rtl:peer-checked:after:-translate-x-full"
              />
              <span class="ms-3 text-sm font-medium text-gray-900 dark:text-gray-300">自动播放</span>
            </label>
          </div>
        </div>
      </div>
      <!-- Table -->
      <div class="mt-6 flex flex-col">
        <div class="overflow-x-auto rounded-lg">
          <div class="inline-block min-w-full align-middle">
            <div class="overflow-hidden shadow sm:rounded-lg">
              <table class="min-w-full table-fixed divide-y divide-gray-200 dark:divide-gray-600">
                <thead class="bg-gray-50 dark:bg-gray-700">
                <tr>
                  <th class="w-[2%] p-4 text-left text-xs ">隐藏</th>
                  <th class="w-[2%] p-4 text-left text-xs  tracking-wider text-gray-500 dark:text-white">
                    #
                  </th>
                  <th class="w-[2%] p-4 text-xs  tracking-wider text-gray-500 dark:text-white">
                    <br>
                  </th>
                  <th class="w-[5%] p-4 text-left text-xs  tracking-wider text-gray-500 dark:text-white">
                    词
                  </th>
                  <th class="w-[6%] w-0 text-left text-xs  text-gray-500 dark:text-white">
                    词性
                  </th>
                  <th class="w-[18%] p-4 text-left text-xs tracking-wider text-gray-500 dark:text-white">
                    词义
                  </th>
                  <th class="w-[32%] p-4 text-left text-xs  tracking-wider text-gray-500 dark:text-white">
                    例句
                  </th>
                  <th class="w-[33%] p-4 text-left text-xs tracking-wider text-gray-500 dark:text-white">
                    拓展
                  </th>
                </tr>
                </thead>
                <tbody class="bg-white dark:bg-gray-800">
                <tr class="bg-hex-f3f3f3">
                  <td
                    colspan="8"
                    class="px-4 py-6 text-sm font-normal text-gray-900 dark:bg-gray-500 dark:text-white"
                  >
                    <div class="flex flex-row">
                      <div class="flex flex-1 items-center">
                        <span class="text-lg">{{ category }}</span>
                        （ {{ refVocabulary[category].groupCount }} 组 {{ refVocabulary[category].wordCount }} 个词 ）
                      </div>
                      <div class="justify-items-end">
                        <audio controls class="chapter">
                          <source :src="`vocabulary/audio/${refVocabulary[category].audio}`" type="audio/mpeg">
                        </audio>
                      </div>
                    </div>
                  </td>
                </tr>
                <template v-for="(wordGroup, i) of displayWords" :key="i">
                  <tr
                    v-for="(item, index) of wordGroup"
                    v-show="!item.hiddenByCheck"
                    :key="item.id"
                    :class="[
                            item.id % 2 === 0 ? 'bg-gray-50 dark:bg-gray-700' : '',

                            (!isTrainingModel && isHistoricalWrong(item))
                              ? 'bg-red-50 dark:bg-red-800'
                              : '',

                            item.markedRed ? 'bg-red-100 dark:bg-red-900' : '',

                            `group-color-${i % 15}`
                          ]"
                    class="text-sm text-gray-900 dark:text-white"
                  >
                    <td class="p-4 flex items-center gap-2">
                      <!-- 隐藏 -->
                      <input type="checkbox" v-model="item.hiddenByCheck">

                      <!-- 标红按钮 -->
                      <i
                        class="i-ph-flag-fill cursor-pointer"
                        :class="item.markedRed ? 'text-red-500' : 'text-gray-400'"
                        @click.stop="item.markedRed = !item.markedRed"
                      />
                    </td>
                    <td class="p-4">
                      {{ item.id }}
                    </td>
                    <td>
                      <i
                        class="i-ph-speaker-simple-high-bold inline-block cursor-pointer"
                        @click.stop="play(`vocabulary/audio/${category}/${item.word[0]}.mp3`)"
                      />

                      <template v-if="isTrainingModel">
                        <i
                          :class="`${item.showMeaning  ? 'i-ph-eye-slash-bold' : 'i-ph-eye-bold'} inline-block cursor-pointer ml-4`"
                          title="显示原词" @click.stop="toggleHint(item)"
                        />
                        <input
                          :id="item.id" autocomplete="off" :class="getInputStyleClass(item) + ' w-[100px]'"
                          type="text"
                          @focusout="onInputFoucsOut($event, item)"
                          @focusin="onInputFoucsIn($event, `vocabulary/audio/${category}/${item.word[0]}.mp3`)"
                          @keydown="onInputKeydown($event, item)"
                        >
                      </template>
                    </td>
                    <td class="group relative whitespace-nowrap overflow-hidden text-ellipsis px-2 py-2 text-sm">
                      <div v-if="
                                !isTrainingModel
                                || isShowMeaning
                                || item.showMeaning
                                || (isOnlyShowErrors && isHistoricalWrong(item))
                              "
                      >

                        <div class="flex items-center">
                          <!-- 单词 -->
                          <div>
                            <p v-for="w in item.word" :key="w">
                              <a
                                class="hover:underline"
                                :title="`在剑桥词典中查询 ${w}`"
                                target="_blank"
                                :href="`https://dictionary.cambridge.org/dictionary/english-chinese-simplified/${w}`"
                              >
                                {{ w }}
                              </a>
                            </p>
                          </div>

                          <!-- 🔥 错词强度 -->
                          <span
                            v-if="getWrongLevel(item) > 0"
                            class="ml-2 text-xs text-red-500"
                          >
                            🔥{{ getWrongLevel(item) }}
                          </span>
                        </div>

                        <!-- 复制按钮 -->
                        <div
                          class="absolute right-0 top-0 hidden h-100% items-center group-hover:flex"
                          @click.stop="copyText(item)"
                        >
                          <i class="i-ph-copy block cursor-pointer px-4"/>
                        </div>
                      </div>
                    </td>


                    <td style="font-style: italic; font-family: times;">
                      {{ item.pos }}
                    </td>
                    <td class="p-4">
                      {{
                        isTrainingModel
                          ? ((isShowMeaning || item.showMeaning) ? item.meaning : '')
                          : item.meaning
                      }}
                    </td>
                    <td class="p-4">
                      {{
                        isTrainingModel
                          ? (isShowMeaning || item.showMeaning || item.showExample ? item.example : '')
                          : item.example
                      }}
                    </td>
                    <td class="p-4 whitespace-pre-line">
                      {{
                        isTrainingModel
                          ? (isShowMeaning || item.showMeaning || item.showExtra ? item.extra : '')
                          : item.extra
                      }}
                    </td>
                  </tr>
                </template>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
      <!-- Card Footer -->
      <div class="flex items-center justify-between pt-3 sm:pt-6">
        <div>
          <p v-if="isTrainingModel">
            {{ trainingStats }}
          </p>
        </div>
        <div v-if="isTrainingModel" class="flex-shrink-0">
          <button
            type="button"
            class="rounded-lg bg-blue-700 px-5 py-2.5 text-sm font-medium text-white dark:bg-blue-600 hover:bg-blue-800 focus:outline-none focus:ring-4 focus:ring-blue-300 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
            @click.stop="finishTraining"
          >
            完成练习
          </button>
          <button
            type="button"
            class="ml-2 rounded-lg bg-blue-700 px-5 py-2.5 text-sm font-medium text-white dark:bg-blue-600 hover:bg-blue-800 focus:outline-none focus:ring-4 focus:ring-blue-300 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
            @click.stop="isOnlyShowErrors = !isOnlyShowErrors"
          >
            {{ isOnlyShowErrors ? '展示所有' : '仅展示错词' }}
          </button>
          <button
            type="button"
            class="ml-2 rounded-lg bg-blue-700 px-5 py-2.5 text-sm font-medium text-white dark:bg-blue-600 hover:bg-blue-800 focus:outline-none focus:ring-4 focus:ring-blue-300 dark:hover:bg-blue-700 dark:focus:ring-blue-800"
            @click.stop="copyAllError"
          >
            拷贝错词
          </button>
        </div>
      </div>

      <div v-if="currentRecords.length" class="mt-4 rounded-lg border p-3">
        <div
          v-for="(r, i) in currentRecords"
          :key="i"
          :class="[
      'px-3 py-2 text-sm',
      i % 2 === 0 ? 'bg-gray-50' : 'bg-white'
    ]"
        >
          {{ formatTime(r.time) }}
          {{ r.correct }}个正确，
          {{ r.error }}个错误
        </div>
      </div>
    </div>
  </div>
</template>
