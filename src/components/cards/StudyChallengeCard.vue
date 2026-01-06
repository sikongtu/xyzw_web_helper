<template>
  <MyCard class="study" :statusClass="{ weekly: true, completed: study.isCompleted }">
    <template #icon>
      <img src="/icons/1736425783912140.png" alt="学习图标">
    </template>
    <template #title>
      <h3>咸鱼大冲关</h3>
      <p>每日知识挑战</p>
    </template>
    <template #badge>
      <span>每周任务</span>
    </template>
    <template #default>
      <p class="description">没有什么可以阻挡我求知的欲望！</p>
    </template>
    <template #action>
      <a-button v-if="!study.thisWeek" status="primary" @click="startStudy">
        🎯 一键答题.
      </a-button>
      <a-button v-if="!study.thisWeek && study.status == 'starting'" status="warning" :disabled="true">
        正在获取题库...
      </a-button>
      <a-button v-if="!study.thisWeek && study.status == 'answering'" status="warning" :disabled="true">
        答题中...
      </a-button>
      <a-button v-if="!study.thisWeek && study.status == 'claiming_rewards'" status="warning" :disabled="true">
        正在领取奖励...
      </a-button>
      <a-button v-if="!study.thisWeek && study.status == 'completed'" status="warning" :disabled="true">
        答题完成
      </a-button>
      <a-button v-if="study.thisWeek" status="success" @click="startStudy">
        ✅ 已完成，点击可继续答题
      </a-button>
    </template>
  </MyCard>
</template>

<script setup>
import { computed } from 'vue'
import { useMessage } from 'naive-ui'
import { preloadQuestions, getQuestionCount } from '@/utils/studyQuestionsFromJSON.js'
import { useTokenStore } from '@/stores/tokenStore'
import MyCard from '../Common/MyCard.vue'

const tokenStore = useTokenStore()
const message = useMessage()
const study = computed(() => tokenStore.gameData.studyStatus)

const startStudy = async () => {

  if (!tokenStore.selectedToken) return
  if (study.value.status != "" && study.value.status != "idel") return
  console.log("开始答题",study.value)

  study.value.status = "starting";
  await preloadQuestions()
  study.value.status = "answering";
  const questionCount = await getQuestionCount()
  message.info(`🚀 开始一键答题... (题库包含 ${questionCount} 道题目)`)

  // 移除对已完成状态的检查，允许用户重复答题
  if (study.value.isCompleted) message.info('🔄 已完成任务，继续答题中...')
  try {
    tokenStore.gameData.studyStatus = {
      ...tokenStore.gameData.studyStatus,
      isAnswering: true,
      questionCount: 0,
      answeredCount: 0,
      status: 'starting',
      timestamp: Date.now()
    }
    const tokenId = tokenStore.selectedToken.id
    tokenStore.sendMessage(tokenId, 'study_startgame')
    setTimeout(() => {
      if (tokenStore.gameData.studyStatus.isAnswering) {
        tokenStore.gameData.studyStatus = {
          ...tokenStore.gameData.studyStatus,
          isAnswering: false,
          questionCount: 0,
          answeredCount: 0,
          status: '',
          timestamp: null
        }
        message.warning('答题超时，已自动重置状态')
      }
    }, 40000)
    message.info(`🚀 开始一键答题... (题库包含 ${questionCount} 道题目)`)
  } catch (error) {
    console.error('启动答题失败:', error)
    message.error('启动答题失败: ' + error.message)
  }
}
</script>
