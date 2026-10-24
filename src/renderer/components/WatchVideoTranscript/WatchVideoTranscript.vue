<template>
  <FtCard class="videoTranscript">
    <details class="transcriptDetails">
      <summary class="transcriptSummary">
        <h3 class="transcriptTitle">
          {{ $t('Transcript.Transcript') }}

          <FontAwesomeIcon
            class="transcriptChevron"
            :icon="['fas', 'chevron-right']"
          />
        </h3>
      </summary>

      <div
        v-if="loading"
        class="transcriptLoading"
      >
        <FtLoader />
      </div>
      <template v-else>
        <div class="transcriptToolbar">
          <FtSelect
            v-if="languages.length > 1"
            class="transcriptLanguageSelect"
            :placeholder="$t('Transcript.Language')"
            :value="selectedLanguage"
            :select-names="languages.map(l => l.name)"
            :select-values="languages.map(l => l.code)"
            :icon="['fas', 'language']"
            @change="(languageCode) => emit('language-change', languageCode)"
          />
          <FtIconButton
            class="transcriptDownload"
            :title="$t('Transcript.Download Transcript')"
            :icon="['fas', 'download']"
            theme="base"
            @click="downloadTranscript"
          />
        </div>

        <div
          ref="transcriptWrapper"
          class="transcriptWrapper"
          @wheel.passive="handleUserScroll"
          @touchmove.passive="handleUserScroll"
          @keydown.arrow-up.stop.prevent="navigateSegments('up')"
          @keydown.arrow-down.stop.prevent="navigateSegments('down')"
        >
          <div
            v-for="(segment, index) in segments"
            :key="index"
            class="transcriptSegment"
            :class="{ active: index === activeSegmentIndex }"
            role="button"
            tabindex="0"
            @click="seekToSegment(index)"
            @keydown.enter.space.stop.prevent="seekToSegment(index)"
          >
            <div class="segmentTimestamp">
              {{ segment.timestamp }}
            </div>
            <p
              class="segmentText"
              dir="auto"
            >
              {{ segment.text }}
            </p>
          </div>
        </div>
      </template>
    </details>
  </FtCard>
</template>

<script setup>
import { computed, ref, useTemplateRef, watch } from 'vue'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'

import FtCard from '../ft-card/ft-card.vue'
import FtIconButton from '../FtIconButton/FtIconButton.vue'
import FtLoader from '../FtLoader/FtLoader.vue'
import FtSelect from '../FtSelect/FtSelect.vue'

import { writeFileWithPicker } from '../../helpers/utils'

const props = defineProps({
  videoTitle: {
    type: String,
    required: true
  },
  segments: {
    type: Array,
    required: true
  },
  languages: {
    type: Array,
    required: true
  },
  selectedLanguage: {
    type: String,
    required: true
  },
  loading: {
    type: Boolean,
    required: true
  },
  currentTime: {
    type: Number,
    default: 0
  }
})

const emit = defineEmits(['timestamp-event', 'language-change'])

const transcriptWrapper = useTemplateRef('transcriptWrapper')

/** How long to pause auto-scrolling after the user scrolls the transcript manually (ms) */
const USER_SCROLL_PAUSE_MS = 3000

const lastUserScrollTime = ref(0)

function handleUserScroll() {
  lastUserScrollTime.value = Date.now()
}

/**
 * The index of the transcript segment currently being spoken,
 * i.e. the last segment that starts at or before the current playback time.
 */
const activeSegmentIndex = computed(() => {
  const segments = props.segments
  const currentMs = props.currentTime * 1000

  let low = 0
  let high = segments.length - 1
  let result = -1

  while (low <= high) {
    const mid = Math.floor((low + high) / 2)

    if (segments[mid].startMs <= currentMs) {
      result = mid
      low = mid + 1
    } else {
      high = mid - 1
    }
  }

  return result
})

watch(activeSegmentIndex, (index) => {
  if (index === -1) { return }

  // Don't yank the transcript around while the user is scrolling through it themselves
  if (Date.now() - lastUserScrollTime.value < USER_SCROLL_PAUSE_MS) { return }

  const wrapper = transcriptWrapper.value
  const segment = wrapper?.children[index]
  if (!wrapper || !segment) { return }

  const wrapperRect = wrapper.getBoundingClientRect()
  const segmentRect = segment.getBoundingClientRect()

  wrapper.scrollTo({
    top: wrapper.scrollTop + segmentRect.top - wrapperRect.top - (wrapper.clientHeight / 2) + (segmentRect.height / 2),
    behavior: 'smooth'
  })
})

/**
 * @param {number} index
 */
function seekToSegment(index) {
  emit('timestamp-event', props.segments[index].startMs / 1000)
  window.scrollTo(0, 0)
}

/**
 * @param {'up' | 'down'} direction
 */
function navigateSegments(direction) {
  const segmentElements = Array.from(transcriptWrapper.value.children)
  const focusedIndex = segmentElements.indexOf(document.activeElement)

  let newIndex = focusedIndex
  if (direction === 'up') {
    newIndex = focusedIndex === 0 ? segmentElements.length - 1 : newIndex - 1
  } else {
    newIndex = focusedIndex === segmentElements.length - 1 ? 0 : newIndex + 1
  }

  segmentElements[newIndex].focus()
}

async function downloadTranscript() {
  const text = props.segments
    .map(segment => `${segment.timestamp} ${segment.text}`)
    .join('\n')

  await writeFileWithPicker(
    `${props.videoTitle}-transcript.txt`,
    text,
    'Text file',
    'text/plain',
    '.txt',
    'transcripts',
    'documents'
  )
}
</script>

<style scoped src="./WatchVideoTranscript.css" />
