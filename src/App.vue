<script setup>
import { computed, onMounted, ref, watch } from 'vue'
import { playlist, soundEffects } from './config/media'

const player = ref(null)
const selectedTrackId = ref(playlist[0]?.id ?? null)
const isPlaying = ref(false)
const isMuted = ref(false)

const currentTrackIndex = computed(() =>
  playlist.findIndex((track) => track.id === selectedTrackId.value),
)

const currentTrack = computed(() => {
  if (currentTrackIndex.value < 0) {
    return null
  }
  return playlist[currentTrackIndex.value]
})

function syncTrack({ autoplay = false } = {}) {
  if (!player.value || !currentTrack.value) {
    return
  }

  player.value.src = currentTrack.value.src
  player.value.load()

  if (autoplay) {
    playMusic()
  }
}

async function playMusic() {
  if (!player.value || !currentTrack.value) {
    return
  }

  if (player.value.src !== new URL(currentTrack.value.src, window.location.origin).href) {
    syncTrack()
  }

  try {
    await player.value.play()
    isPlaying.value = true
  } catch {
    isPlaying.value = false
  }
}

function pauseMusic() {
  if (!player.value) {
    return
  }

  player.value.pause()
  isPlaying.value = false
}

function togglePlay() {
  if (isPlaying.value) {
    pauseMusic()
    return
  }
  playMusic()
}

function stepTrack(step) {
  if (!playlist.length) {
    return
  }

  const nextIndex =
    (currentTrackIndex.value + step + playlist.length) % playlist.length
  selectedTrackId.value = playlist[nextIndex].id
}

function nextTrack() {
  stepTrack(1)
}

function previousTrack() {
  stepTrack(-1)
}

function toggleMute() {
  if (!player.value) {
    return
  }

  isMuted.value = !isMuted.value
  player.value.muted = isMuted.value
}

function playEffect(effect) {
  const effectPlayer = new Audio(effect.src)
  effectPlayer.play().catch(() => {})
}

function onTrackEnded() {
  nextTrack()
  playMusic()
}

watch(selectedTrackId, () => {
  syncTrack({ autoplay: isPlaying.value })
})

onMounted(() => {
  syncTrack()
})
</script>

<template>
  <v-app>
    <v-main>
      <v-container class="py-4">
        <v-btn
          class="mute-button"
          color="error"
          size="x-large"
          block
          @click="toggleMute"
        >
          {{ isMuted ? 'Unmute Music' : 'Mute Music' }}
        </v-btn>

        <v-card class="mt-4" rounded="lg">
          <v-card-title class="text-h5">Now Playing</v-card-title>
          <v-card-subtitle>
            {{ currentTrack?.title || 'No track selected' }}
          </v-card-subtitle>
          <v-card-text class="pt-2">
            <div class="text-body-1 font-weight-medium">
              {{ currentTrack?.artist || 'Select a track to start' }}
            </div>

            <v-btn-toggle class="mt-4 d-flex" divided>
              <v-btn size="large" class="flex-fill" @click="previousTrack">Previous</v-btn>
              <v-btn size="large" class="flex-fill" color="primary" @click="togglePlay">
                {{ isPlaying ? 'Pause' : 'Play' }}
              </v-btn>
              <v-btn size="large" class="flex-fill" @click="nextTrack">Next</v-btn>
            </v-btn-toggle>

            <audio ref="player" @ended="onTrackEnded" />
          </v-card-text>
        </v-card>

        <v-card class="mt-4" rounded="lg">
          <v-card-title class="text-h6">Playlist</v-card-title>
          <v-card-text>
            <v-select
              v-model="selectedTrackId"
              :items="playlist"
              item-title="title"
              item-value="id"
              label="Select Song"
              density="comfortable"
              hide-details
            />
          </v-card-text>
        </v-card>

        <v-card class="mt-4" rounded="lg">
          <v-card-title class="text-h6">Sound Effects</v-card-title>
          <v-card-text>
            <v-row dense>
              <v-col
                v-for="effect in soundEffects"
                :key="effect.id"
                cols="6"
                sm="4"
              >
                <v-btn
                  block
                  color="secondary"
                  size="large"
                  @click="playEffect(effect)"
                >
                  {{ effect.label }}
                </v-btn>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-container>
    </v-main>
  </v-app>
</template>

<style scoped>
.mute-button {
  position: sticky;
  top: 0;
  z-index: 5;
}
</style>
