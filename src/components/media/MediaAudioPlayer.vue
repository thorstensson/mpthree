<script setup lang="ts">
import { useEventListener, type MaybeRef } from "@vueuse/core"
import {
  ref,
  useTemplateRef,
  reactive,
  onMounted,
  computed,
  provide,
} from "vue"
import {
  PlayIcon,
  PauseIcon,
  ChevronRightIcon,
  ChevronLeftIcon,
} from "@heroicons/vue/24/solid"

import AudioVisualizerThree from "./MediaAudioVisualizerThree.vue"
import { useStoreRef } from "@/composable/useStoreRef"
const audioEl = useTemplateRef("audio-element")

// Provide audioEl to child components
provide("audioEl", audioEl)

const trackTime = ref<string>("00:00")
const trackDuration = ref<string>("00.00")
const trackIndex = ref<number>(0)
const currentTrack = ref<string>("")
const isPlaying = ref<boolean>(false)

const PATH = import.meta.env.VITE_MP3_URL

//Add tracks here; no plans to make a DOM playlist
const playlist = reactive([
  { artist: "Lorn", track: "Lorn - Folding (Original Mix).mp3" },

  { artist: "ashess", track: "ashess - only you (Original Mix).mp3" },

  { artist: "Sky_s Memoirs", track: "Sky_s Memoirs - Nova (Original Mix).mp3" },
])

// Check for remaining tracks
const ifTrackNext = computed(() => {
  return trackIndex.value < playlist.length - 1
})

const ifTrackPrev = computed(() => {
  return trackIndex.value > 0
})

// Check for current track
const currTrack = computed(() => {
  return playlist[trackIndex.value].track
})

const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  if (isPlaying.value && audioEl.value) {
    playTrack()
  } else if (audioEl.value) {
    audioEl.value.pause()
  }
}

// For previous and next we need to know if track is playing when we press them
const nextTrack = () => {
  if (ifTrackNext.value && !isPlaying.value) {
    trackIndex.value++
    currentTrack.value = currTrack.value
  } else if (ifTrackNext.value) {
    isPlaying.value = true
    trackIndex.value++
    playTrack()
  }
}

const prevTrack = () => {
  if (ifTrackPrev.value && !isPlaying.value) {
    trackIndex.value--
    currentTrack.value = currTrack.value
  } else if (ifTrackPrev.value) {
    isPlaying.value = true
    trackIndex.value--
    playTrack()
  }
}

const playTrack = () => {
  // Vueuse, easy cancel. oncanplaythrough does not work on mobile, loadedmetadata does???
  const cancelcan = useEventListener(
    audioEl.value as unknown as MaybeRef,
    "loadedmetadata",
    () => {
      audioEl.value?.play()
      cancelcan()
    }
  )
  // Synchronous, so we do this after adding event
  isPlaying.value = true
  audioEl.value!.currentTime = 0
  currentTrack.value = currTrack.value
  audioEl.value?.load()
}

// E from v-on listener
const timeUpdate = () => {
  setTimes()
}

// Times, leaving this in, in case you want to use
const setTimes = () => {
  const m = ("0" + Math.floor((audioEl.value!.currentTime / 60) % 60)).slice(-2)
  const s = ("0" + Math.floor(audioEl.value!.currentTime % 60)).slice(-2)
  trackTime.value = `${m}:${s}`
}

// E from v-on listener, leaving this in, in case you want to use
const durationUpdate = () => {
  const m = ("0" + Math.floor((audioEl.value!.duration / 60) % 60)).slice(-2)
  const s = ("0" + Math.floor(audioEl.value!.duration % 60)).slice(-2)
  trackDuration.value = `${m}:${s}`
}

const onTrackEnded = () => {
  if (ifTrackNext.value) {
    trackIndex.value++
    playTrack()
  } else if (audioEl.value) {
    isPlaying.value = false
    trackIndex.value = 0
    audioEl.value.pause()
    audioEl.value.currentTime = 0
    currentTrack.value = currTrack.value
  }
}
onMounted(() => {
  const { addElem } = useStoreRef()
  addElem("audioEl", audioEl)
  currentTrack.value = currTrack.value
})
</script>

<template>
  <div class="media-wrapper">
    <!-- Three.js fullscreen background visualizer -->
    <AudioVisualizerThree />

    <div class="player">
      <audio
        :src="`${PATH}/mp3/${currentTrack}`"
        type="audio/mp3"
        preload="auto"
        v-on:timeupdate="timeUpdate"
        v-on:durationchange="durationUpdate"
        v-on:ended="onTrackEnded"
        ref="audio-element"
        crossorigin="anonymous"
      ></audio>
      <div class="player__controls">
        <ChevronLeftIcon
          @click="prevTrack"
          class="player__controls__prev"
          :class="{ 'player__controls__prev--end': !ifTrackPrev }"
        >
        </ChevronLeftIcon>
        <PlayIcon
          @click="togglePlay"
          class="player__controls__play"
          :class="{ 'player__controls__play--show': !isPlaying }"
        />
        <PauseIcon
          @click="togglePlay"
          class="player__controls__pause"
          :class="{ 'player__controls__pause--show': isPlaying }"
        />
        <ChevronRightIcon
          @click="nextTrack"
          class="player__controls__next"
          :class="{ 'player__controls__next--end': !ifTrackNext }"
        >
        </ChevronRightIcon>
      </div>
      <div class="player__info">
        <span class="player__info__text"
          >{{ playlist[trackIndex].artist }} -
          {{
            playlist[trackIndex].track
              .replace(".mp3", "")
              .replace(playlist[trackIndex].artist + " - ", "")
          }}</span
        >
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
body {
  -webkit-overflow-scrolling: none;
  overflow: hidden;
  overscroll-behavior: none;
}

.media-wrapper {
  display: flex;
  flex-flow: column;
  row-gap: 16px;
  -webkit-overflow-scrolling: none;
  overflow: hidden;
  overscroll-behavior: none;
  text-transform: uppercase;
}

.player__controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 16px;
  &__play,
  &__pause {
    display: none;
    width: 32px;
    height: auto;
    color: #e6e6e6;
    cursor: pointer;
    transition: color 0.3s ease-in-out;

    /* Prevent focus outline on touch devices */
    -webkit-tap-highlight-color: transparent;
    outline: none;

    &--show {
      display: block;
    }
  }

  &__play:hover,
  &__pause:hover {
    color: $accent2;
  }

  &__prev,
  &__next {
    width: 32px;
    height: auto;
    color: #e6e6e6;
    cursor: pointer;
    transition: color 0.3s ease-in-out;

    /* Prevent focus outline on touch devices */
    -webkit-tap-highlight-color: transparent;
    outline: none;

    &--end {
      opacity: 0.5;
    }
  }

  &__next:hover,
  &__prev:focus {
    color: $accent2;
  }
}

.player__info {
  margin-top: 16px;
  text-align: center;

  &__text {
    font-family: $sans-text;
    font-size: 12px;
    font-weight: 400;
    color: #e6e6e6;
    letter-spacing: 0.5px;
    user-select: none;
  }
}
</style>
