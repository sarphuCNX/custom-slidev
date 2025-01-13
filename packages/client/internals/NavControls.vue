<script setup lang="ts">
import CustomNavControls from '#slidev/custom-nav-controls'
import { computed, onMounted, onUnmounted, ref, shallowRef, watch } from 'vue'
import { useDrawings } from '../composables/useDrawings'
import { useNav } from '../composables/useNav'
import { configs } from '../env'
import { cnxSetIsDark, isColorSchemaConfigured, isDark, toggleDark } from '../logic/dark'
import { activeElement, breakpoints, fullscreen, hasViewerCssFilter, presenterLayout, setOverview, showEditor, showInfoDialog, showPresenterCursor, toggleOverview, togglePresenterLayout } from '../state'
import { downloadPDF } from '../utils'
import IconButton from './IconButton.vue'
import MenuButton from './MenuButton.vue'
import Settings from './Settings.vue'
import SyncControls from './SyncControls.vue'

import VerticalDivider from './VerticalDivider.vue'

interface ISendMessageToParent {
  event: string
  type: string
  data: any
}

const props = defineProps({
  persist: {
    default: false,
  },
})

const {
  currentSlideNo,
  hasNext,
  hasPrev,
  isEmbedded,
  isPresenter,
  isPresenterAvailable,
  next,
  prev,
  total,
  enterPresenter,
  exitPresenter,
  go,
} = useNav()
const {
  brush,
  drawingEnabled,
} = useDrawings()

const isNavControlsVisible = ref(false)

function setNavControlsVisibility(visible: boolean) {
  if (typeof visible === 'boolean') {
    isNavControlsVisible.value = visible
  }
}

// Data to track previously sent values
let previousNavData: Partial<Record<string, any>> = {}

function sendMessageToWebParent(payload: ISendMessageToParent) {
  window.parent.postMessage({ type: payload.type, data: payload.data }, '*')
}

function sendMessageToDeviceParent(payload: ISendMessageToParent) {
  if (window.jsToNativeHandler && window.jsToNativeHandler.sendMessage) {
    window.jsToNativeHandler.sendMessage(JSON.stringify(payload))
  }
}

function sendMessageToParent(payload: ISendMessageToParent) {
  sendMessageToWebParent(payload)
  sendMessageToDeviceParent(payload)
}

// Function to check for changes and send updated data
function sendNavData() {
  const navData = {
    currentSlideNo: currentSlideNo.value,
    hasNext: hasNext.value,
    hasPrev: hasPrev.value,
    total: total.value,
    isPresenter: isPresenter.value,
  }

  // Check if the new navData is different from the previous one
  if (JSON.stringify(navData) !== JSON.stringify(previousNavData)) {
    sendMessageToParent({ event: 'message', type: 'slidev-nav-update', data: navData })

    previousNavData = { ...navData }
  }
}

// Watch for changes in useNav properties and call sendNavData
const stopWatchers = [
  watch(currentSlideNo, sendNavData),
  watch(hasNext, sendNavData),
  watch(hasPrev, sendNavData),
  watch(total, sendNavData),
  watch(isPresenter, sendNavData),
]

/**
 * Note:
 * Custom Code
 * @param event
 */
function handleMessage(event: MessageEvent) {
  if (event.data.type === 'nav-command') {
    if (event.data.command === 'next') {
      next()
    }
    else if (event.data.command === 'prev') {
      prev()
    }
    else if (event.data.command === 'goto') {
      const value = Number.isInteger(event.data.value) ? event.data.value : 1
      go(value)
    }
    else if (event.data.command === 'show-nav-controls') {
      const value = typeof event.data.value === 'boolean' ? event.data.value : false
      setNavControlsVisibility(value)
    }
  }
  else if (event.data.type === 'theme-command') {
    if (event.data.command === 'set-dark-theme') {
      const value = typeof event.data.value === 'boolean' ? event.data.value : true
      cnxSetIsDark(value)
    }
  }
  else if (event.data.type === 'drawing-command') {
    if (event.data.command === 'set-drawing-enabled') {
      const value = typeof event.data.value === 'boolean' ? event.data.value : false
      drawingEnabled.value = value
    }
  }
  else if (event.data.type === 'grid-command') {
    if (event.data.command === 'show-grid') {
      const value = typeof event.data.value === 'boolean' ? event.data.value : false
      setOverview(value)
    }
  }
}

onMounted(() => {
  window.addEventListener('message', handleMessage)
})

// Clean up watchers on unmount
onUnmounted(() => {
  stopWatchers.forEach(stop => stop())
  window.removeEventListener('message', handleMessage)
})

const md = breakpoints.smaller('md')
const { isFullscreen, toggle: toggleFullscreen } = fullscreen

const root = ref<HTMLDivElement>()
function onMouseLeave() {
  if (root.value && activeElement.value && root.value.contains(activeElement.value))
    activeElement.value.blur()
}

const barStyle = computed(() => props.persist
  ? 'text-$slidev-controls-foreground bg-transparent'
  : 'rounded-md bg-main shadow-xl border border-main')

const RecordingControls = shallowRef<any>()
if (__SLIDEV_FEATURE_RECORD__)
  import('./RecordingControls.vue').then(v => RecordingControls.value = v.default)
</script>

<template>
  <nav v-if="isNavControlsVisible" ref="root" class="flex flex-col">
    <div
      class="flex flex-wrap-reverse text-xl gap-0.5 p-1 lg:p-2"
      :class="barStyle"
      @mouseleave="onMouseLeave"
    >
      <IconButton v-if="false && !isEmbedded" :title="isFullscreen ? 'Close fullscreen' : 'Enter fullscreen'" @click="toggleFullscreen">
        <div v-if="isFullscreen" class="i-carbon:minimize" />
        <div v-else class="i-carbon:maximize" />
      </IconButton>
      <IconButton :class="{ disabled: !hasPrev }" title="Go to previous slide" @click="prev">
        <div class="i-carbon:arrow-left" />
      </IconButton>
      <IconButton :class="{ disabled: !hasNext }" title="Go to next slide" @click="next">
        <div class="i-carbon:arrow-right" />
      </IconButton>
      <IconButton v-if="!isEmbedded" title="Show slide overview" @click="toggleOverview()">
        <div class="i-carbon:apps" />
      </IconButton>
      <IconButton
        v-if="!isColorSchemaConfigured"
        :title="isDark ? 'Switch to light mode theme' : 'Switch to dark mode theme'"
        @click="toggleDark()"
      >
        <carbon-moon v-if="isDark" />
        <carbon-sun v-else />
      </IconButton>

      <VerticalDivider />

      <template v-if="false && !isEmbedded">
        <template v-if="!isPresenter && !md && RecordingControls">
          <RecordingControls />
          <VerticalDivider />
        </template>

        <IconButton
          v-if="isPresenter"
          :title="showPresenterCursor ? 'Hide presenter cursor' : 'Show presenter cursor'"
          @click="showPresenterCursor = !showPresenterCursor"
        >
          <ph-cursor-fill v-if="showPresenterCursor" />
          <ph-cursor-duotone v-else />
        </IconButton>
      </template>

      <template v-if="__SLIDEV_FEATURE_DRAWINGS__ && (!configs.drawings.presenterOnly || isPresenter) && !isEmbedded">
        <IconButton class="relative" :title="drawingEnabled ? 'Hide drawing toolbar' : 'Show drawing toolbar'" @click="drawingEnabled = !drawingEnabled">
          <div class="i-carbon:pen" />
          <div
            v-if="drawingEnabled"
            class="absolute left-1 right-1 bottom-0 h-0.7 rounded-full"
            :style="{ background: brush.color }"
          />
        </IconButton>
        <VerticalDivider />
      </template>

      <template v-if="false && !isEmbedded">
        <IconButton v-if="isPresenter" title="Play Mode" @click="exitPresenter">
          <div class="i-carbon:presentation-file" />
        </IconButton>
        <IconButton v-if="__SLIDEV_FEATURE_PRESENTER__ && isPresenterAvailable" title="Presenter Mode" @click="enterPresenter">
          <div class="i-carbon:user-speaker" />
        </IconButton>

        <IconButton
          v-if="false && __DEV__ && __SLIDEV_FEATURE_EDITOR__"
          :title="showEditor ? 'Hide editor' : 'Show editor'"
          class="lt-md:hidden"
          @click="showEditor = !showEditor"
        >
          <div class="i-carbon:text-annotation-toggle" />
        </IconButton>
      </template>

      <template v-if="false && !__DEV__">
        <IconButton v-if="configs.download" title="Download as PDF" @click="downloadPDF">
          <div class="i-carbon:download" />
        </IconButton>
      </template>

      <template v-if="false && __SLIDEV_FEATURE_BROWSER_EXPORTER__ && !isEmbedded && !isPresenter">
        <IconButton title="Browser Exporter" to="/export">
          <div class="i-carbon:document-pdf" />
        </IconButton>
      </template>

      <IconButton
        v-if="false && !isPresenter && configs.info && !isEmbedded"
        title="Show info"
        @click="showInfoDialog = !showInfoDialog"
      >
        <div class="i-carbon:information" />
      </IconButton>

      <template v-if="false && !isEmbedded">
        <VerticalDivider />

        <IconButton v-if="isPresenter" title="Toggle Presenter Layout" class="aspect-ratio-initial flex items-center" @click="togglePresenterLayout">
          <div class="i-carbon:template" />
          {{ presenterLayout }}
        </IconButton>

        <SyncControls v-if="__SLIDEV_FEATURE_PRESENTER__" />

        <MenuButton>
          <template #button>
            <IconButton title="More Options">
              <div class="i-carbon:settings-adjust" />
              <div v-if="hasViewerCssFilter" w-2 h-2 bg-primary rounded-full absolute top-0.5 right-0.5 />
            </IconButton>
          </template>
          <template #menu>
            <Settings />
          </template>
        </MenuButton>
      </template>

      <VerticalDivider v-if="false && !isEmbedded" />

      <div class="h-40px flex" p="l-1 t-0.5 r-2" text="sm leading-2">
        <div class="my-auto">
          {{ currentSlideNo }}
          <span class="opacity-50">/ {{ total }}</span>
        </div>
      </div>

      <CustomNavControls />
    </div>
  </nav>
</template>
