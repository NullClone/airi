<script setup lang="ts">
import {
  SpeechPlayground,
  SpeechProviderSettings,
} from '@proj-airi/stage-ui/components'
import { useSpeechStore } from '@proj-airi/stage-ui/stores/modules/speech'
import { useProvidersStore } from '@proj-airi/stage-ui/stores/providers'
import { FieldRange } from '@proj-airi/ui'
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'

const providerId = 'aivis-speech'
const defaultModel = 'aivis-speech'

const speechStore = useSpeechStore()
const providersStore = useProvidersStore()
const { t } = useI18n()

const apiKeyConfigured = computed(() => true) // AivisSpeech doesn't require API key

const availableVoices = computed(() => {
  return speechStore.availableVoices[providerId] || []
})

const speed = ref<number>(1.0)
const pitch = ref<number>(0.0)
const intonation = ref<number>(1.0)

onMounted(async () => {
  // Initialize provider first
  providersStore.initializeProvider(providerId)

  // Load saved parameters
  const providerConfig = providersStore.getProviderConfig(providerId)
  if (providerConfig.speed !== undefined) {
    speed.value = providerConfig.speed as number
  }
  if (providerConfig.pitch !== undefined) {
    pitch.value = providerConfig.pitch as number
  }
  if (providerConfig.intonation !== undefined) {
    intonation.value = providerConfig.intonation as number
  }

  // Try to load voices (will fail gracefully if engine is not running)
  try {
    await speechStore.loadVoicesForProvider(providerId)
  }
  catch (error) {
    // Error is already handled by loadVoicesForProvider
    console.warn('Failed to load AivisSpeech voices:', error)
  }
})

watch([apiKeyConfigured], async () => {
  try {
    await speechStore.loadVoicesForProvider(providerId)
  }
  catch (error) {
    console.warn('Failed to load AivisSpeech voices:', error)
  }
})

// Generate speech using the provider's defined speech method
// 設定画面でのテスト発話用関数
// 上の import 部分などはそのままでOK
// handleGenerateSpeech 関数の中身だけをこれに差し替えます

async function handleGenerateSpeech(input: string, voiceId: string, _useSSML: boolean) {
  // 1. ストアから最新の設定（URL）を取得
  const config = providersStore.getProviderConfig(providerId)
  // URLの末尾のスラッシュを削除して正規化
  const baseUrl = ((config.baseUrl as string) || 'http://127.0.0.1:10101').trim().replace(/\/$/, '')

  if (!input)
    throw new Error('Text is empty')

  // 2. パラメータの準備
  const params = {
    speedScale: speed.value,
    pitchScale: pitch.value,
    intonationScale: intonation.value,
  }

  // --- Step 1: audio_query (クエリ作成) ---
  const queryUrl = `${baseUrl}/audio_query?text=${encodeURIComponent(input)}&speaker=${voiceId}`
  const queryRes = await fetch(queryUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
  })

  if (!queryRes.ok) {
    throw new Error(`audio_query failed: ${queryRes.status} ${queryRes.statusText}`)
  }

  const queryJson = await queryRes.json()

  // パラメータを適用 (AivisSpeech/VOICEVOXの仕様)
  queryJson.speedScale = params.speedScale
  queryJson.pitchScale = params.pitchScale
  queryJson.intonationScale = params.intonationScale

  // --- Step 2: synthesis (音声合成) ---
  const synthUrl = `${baseUrl}/synthesis?speaker=${voiceId}`
  const synthRes = await fetch(synthUrl, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(queryJson), // Step 1の結果を送信
  })

  if (!synthRes.ok) {
    throw new Error(`synthesis failed: ${synthRes.status} ${synthRes.statusText}`)
  }

  // 音声データ(ArrayBuffer)を返して再生させる
  return await synthRes.arrayBuffer()
}

watch(speed, () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.speed = speed.value
})

watch(pitch, () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.pitch = pitch.value
})

watch(intonation, () => {
  const providerConfig = providersStore.getProviderConfig(providerId)
  providerConfig.intonation = intonation.value
})
</script>

<template>
  <SpeechProviderSettings :provider-id="providerId" :default-model="defaultModel">
    <!-- Voice settings specific to AivisSpeech -->
    <template #voice-settings>
      <!-- Speed control -->
      <FieldRange
        v-model="speed"
        :label="t('settings.pages.providers.provider.common.fields.field.speed.label')"
        :description="t('settings.pages.providers.provider.common.fields.field.speed.description')"
        :min="0.5"
        :max="2.0"
        :step="0.01"
      />

      <!-- Pitch control -->
      <FieldRange
        v-model="pitch"
        :label="t('settings.pages.providers.provider.aivis-speech.fields.pitch.label')"
        :description="t('settings.pages.providers.provider.aivis-speech.fields.pitch.description')"
        :min="-0.15"
        :max="0.15"
        :step="0.01"
      />

      <!-- Intonation control -->
      <FieldRange
        v-model="intonation"
        :label="t('settings.pages.providers.provider.aivis-speech.fields.intonation.label')"
        :description="t('settings.pages.providers.provider.aivis-speech.fields.intonation.description')"
        :min="0.0"
        :max="2.0"
        :step="0.01"
      />
    </template>

    <template #playground>
      <SpeechPlayground
        :available-voices="availableVoices"
        :generate-speech="handleGenerateSpeech"
        :api-key-configured="apiKeyConfigured"
        :use-ssml="false"
        default-text="こんにちは、AivisSpeechの音声合成テストです。"
      />
    </template>
  </SpeechProviderSettings>
</template>

<route lang="yaml">
  meta:
    layout: settings
    stageTransition:
      name: slide
</route>
