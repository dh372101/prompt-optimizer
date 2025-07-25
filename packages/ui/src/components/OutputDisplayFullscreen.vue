<template>
  <FullscreenDialog v-model="internalVisible" :title="title || t('common.content')">
    <OutputDisplayCore
        ref="coreDisplayRef"
        :content="content"
        :originalContent="originalContent"
        :reasoning="reasoning"
        :mode="mode"
        :reasoningMode="reasoningMode"
        :enabledActions="coreEnabledActions"
        height="100%"
        :placeholder="placeholder"
        :loading="loading"
        :streaming="streaming"
        :compareService="compareService"
        @update:content="handleContentUpdate"
        @copy="handleCopy"
    />
  </FullscreenDialog>
</template>

<script setup lang="ts">
import { computed, ref, watch, nextTick, inject } from 'vue'
import { useI18n } from 'vue-i18n'
import FullscreenDialog from './FullscreenDialog.vue'
import OutputDisplayCore from './OutputDisplayCore.vue'
import type { AppServices } from '../types/services';
import type { Ref } from 'vue';

const { t } = useI18n()

// Props
interface Props {
  modelValue: boolean
  content?: string
  originalContent?: string
  reasoning?: string
  title?: string
  mode: 'readonly' | 'editable'
  reasoningMode?: 'show' | 'hide' | 'auto'
  enabledActions?: ('fullscreen' | 'diff' | 'copy' | 'edit' | 'reasoning')[]
  streaming?: boolean
  loading?: boolean
  placeholder?: string
}
const props = withDefaults(defineProps<Props>(), {
  content: '',
  originalContent: '',
  reasoning: '',
  title: '',
  mode: 'readonly',
  reasoningMode: 'auto',
  enabledActions: () => ['diff', 'copy', 'edit', 'reasoning'],
  placeholder: ''
})

// Emits
const emit = defineEmits<{
  'update:modelValue': [value: boolean]
  'update:content': [content: string]
  'copy': [content: string, type: 'content' | 'reasoning' | 'all']
}>()


// 注入服务并获取 CompareService
const services = inject<Ref<AppServices | null>>('services');
if (!services) {
  throw new Error('[OutputDisplayFullscreen] services未正确注入，请确保在App组件中正确provide了services');
}

const compareService = computed(() => {
  const servicesValue = services.value;
  if (!servicesValue) {
    throw new Error('[OutputDisplayFullscreen] services未初始化，请确保应用已正确启动');
  }

  const service = servicesValue.compareService;
  if (!service) {
    throw new Error('[OutputDisplayFullscreen] compareService未初始化，请确保服务已正确配置');
  }

  return service;
});

// Internal state
const coreDisplayRef = ref<InstanceType<typeof OutputDisplayCore> | null>(null)
const internalVisible = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})

const coreEnabledActions = computed(() => {
  return props.enabledActions?.filter(action => action !== 'fullscreen')
})

const internalContent = ref(props.content)
const isFullscreenReasoningExpanded = ref(true)

watch(() => props.content, (newVal) => {
  internalContent.value = newVal
})

// Sync back changes when dialog closes
watch(internalVisible, (newVal) => {
  if (newVal) {
    // Dialog is opening, decide the initial state of the reasoning panel.
    const hasMainContent = !!props.content;
    // Main content is loading if streaming has started AND there's some content already.
    const isMainContentLoading = props.loading || (props.streaming && hasMainContent);

    const shouldReasoningBeExpanded = !(hasMainContent || isMainContentLoading);

    nextTick(() => {
      coreDisplayRef.value?.resetReasoningState(shouldReasoningBeExpanded)
    })
  } else {
    // Dialog is closing
    if (props.content !== internalContent.value) {
      emit('update:content', internalContent.value || '')
    }
  }
})

// Methods
const handleContentUpdate = (newContent: string) => {
    internalContent.value = newContent
}

const handleCopy = (content: string, type: 'content' | 'reasoning' | 'all') => {
  emit('copy', content, type)
}

</script> 