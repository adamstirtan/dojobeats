<script setup>
import { computed, onMounted, ref, watch } from "vue";
import { playlist, soundEffects } from "./config/media";
import {
  List,
  SkipBack,
  Play,
  Pause,
  SkipForward,
  ArrowLeft,
} from "lucide-vue-next";

const PLAYLIST_STORAGE_KEY = "dojobeats-playlist-preferences";

function buildDefaultPlaylistState() {
  return playlist.map((track) => ({ ...track, enabled: true }));
}

function buildPlaylistStateFromPreferences(rawPreferences) {
  const baseTracksById = new Map(playlist.map((track) => [track.id, track]));

  if (!Array.isArray(rawPreferences) || !rawPreferences.length) {
    return buildDefaultPlaylistState();
  }

  const hydratedTracks = rawPreferences
    .map((item) => {
      const baseTrack = baseTracksById.get(item?.id);

      if (!baseTrack) {
        return null;
      }

      return {
        ...baseTrack,
        enabled: item.enabled !== false,
      };
    })
    .filter(Boolean);

  const hydratedIds = new Set(hydratedTracks.map((track) => track.id));
  const missingTracks = playlist
    .filter((track) => !hydratedIds.has(track.id))
    .map((track) => ({ ...track, enabled: true }));

  return [...hydratedTracks, ...missingTracks];
}

function loadPlaylistState() {
  if (typeof window === "undefined") {
    return buildDefaultPlaylistState();
  }

  try {
    const rawPreferences = window.localStorage.getItem(PLAYLIST_STORAGE_KEY);

    if (!rawPreferences) {
      return buildDefaultPlaylistState();
    }

    return buildPlaylistStateFromPreferences(JSON.parse(rawPreferences));
  } catch {
    return buildDefaultPlaylistState();
  }
}

const player = ref(null);
const playlistState = ref(buildDefaultPlaylistState());
const selectedTrackId = ref(playlist[0]?.id ?? null);
const isPlaying = ref(false);
const isMuted = ref(false);
const showPlaylistEditor = ref(false);
const progress = ref(0);

const enabledPlaylist = computed(() =>
  playlistState.value.filter((track) => track.enabled),
);

const currentTrackIndex = computed(() =>
  enabledPlaylist.value.findIndex(
    (track) => track.id === selectedTrackId.value,
  ),
);

const currentTrack = computed(() => {
  if (currentTrackIndex.value < 0) {
    return null;
  }
  return enabledPlaylist.value[currentTrackIndex.value];
});

function syncTrack({ autoplay = false } = {}) {
  if (!player.value || !currentTrack.value) {
    return;
  }

  player.value.src = currentTrack.value.src;
  player.value.load();

  if (autoplay) {
    playMusic();
  }
}

async function playMusic() {
  if (!player.value || !currentTrack.value) {
    return;
  }

  if (
    player.value.src !==
    new URL(currentTrack.value.src, window.location.origin).href
  ) {
    syncTrack();
  }

  try {
    await player.value.play();
    isPlaying.value = true;
  } catch {
    isPlaying.value = false;
  }
}

function pauseMusic() {
  if (!player.value) {
    return;
  }

  player.value.pause();
  isPlaying.value = false;
}

function togglePlay() {
  if (isPlaying.value) {
    pauseMusic();
    return;
  }
  playMusic();
}

function stepTrack(step) {
  if (!enabledPlaylist.value.length) {
    return;
  }

  const nextIndex =
    (currentTrackIndex.value + step + enabledPlaylist.value.length) %
    enabledPlaylist.value.length;
  selectedTrackId.value = enabledPlaylist.value[nextIndex].id;
}

function nextTrack() {
  stepTrack(1);
}

function previousTrack() {
  stepTrack(-1);
}

function toggleMute() {
  if (!player.value) {
    return;
  }

  isMuted.value = !isMuted.value;
  player.value.muted = isMuted.value;
}

function toggleEditorSide() {
  showPlaylistEditor.value = !showPlaylistEditor.value;
}

function toggleTrackEnabled(trackId) {
  playlistState.value = playlistState.value.map((track) =>
    track.id === trackId ? { ...track, enabled: !track.enabled } : track,
  );
}

function moveTrack(trackId, direction) {
  const currentIndex = playlistState.value.findIndex(
    (track) => track.id === trackId,
  );
  const targetIndex = currentIndex + direction;

  if (
    currentIndex < 0 ||
    targetIndex < 0 ||
    targetIndex >= playlistState.value.length
  ) {
    return;
  }

  const reorderedPlaylist = [...playlistState.value];
  const [movedTrack] = reorderedPlaylist.splice(currentIndex, 1);
  reorderedPlaylist.splice(targetIndex, 0, movedTrack);
  playlistState.value = reorderedPlaylist;
}

function playEffect(effect) {
  const effectPlayer = new Audio(effect.src);
  effectPlayer.play().catch(() => {});
}

function onTrackEnded() {
  nextTrack();
  playMusic();
}

function onTimeUpdate() {
  if (!player.value || !player.value.duration) return;
  progress.value = (player.value.currentTime / player.value.duration) * 100;
}

function onSeeked(event) {
  if (!player.value || !player.value.duration) return;
  const rect = event.currentTarget.getBoundingClientRect();
  const x = (event.clientX ?? event.touches?.[0]?.clientX ?? 0) - rect.left;
  player.value.currentTime = (x / rect.width) * player.value.duration;
}

watch(
  playlistState,
  (tracks) => {
    if (typeof window === "undefined") {
      return;
    }

    const storedPreferences = tracks.map((track) => ({
      id: track.id,
      enabled: track.enabled,
    }));

    window.localStorage.setItem(
      PLAYLIST_STORAGE_KEY,
      JSON.stringify(storedPreferences),
    );

    if (!enabledPlaylist.value.length) {
      pauseMusic();
      selectedTrackId.value = null;
      return;
    }

    const isSelectedTrackEnabled = enabledPlaylist.value.some(
      (track) => track.id === selectedTrackId.value,
    );

    if (!isSelectedTrackEnabled) {
      selectedTrackId.value = enabledPlaylist.value[0].id;
    }
  },
  { deep: true },
);

watch(selectedTrackId, () => {
  syncTrack({ autoplay: isPlaying.value });
});

function updateMediaSession() {
  if (!("mediaSession" in navigator)) return;

  if (currentTrack.value) {
    navigator.mediaSession.metadata = new MediaMetadata({
      title: currentTrack.value.title,
      artist: "DojoBeats",
    });
  }

  navigator.mediaSession.setActionHandler("play", () => playMusic());
  navigator.mediaSession.setActionHandler("pause", () => pauseMusic());
  navigator.mediaSession.setActionHandler("previoustrack", () => {
    previousTrack();
  });
  navigator.mediaSession.setActionHandler("nexttrack", () => {
    nextTrack();
  });
}

watch(currentTrack, () => {
  updateMediaSession();
});

watch(isPlaying, (playing) => {
  if (!("mediaSession" in navigator)) return;
  navigator.mediaSession.playbackState = playing ? "playing" : "paused";
});

onMounted(() => {
  playlistState.value = loadPlaylistState();

  if (playlistState.value.length) {
    const firstEnabledTrack = playlistState.value.find(
      (track) => track.enabled,
    );
    selectedTrackId.value = firstEnabledTrack?.id ?? null;
  }

  syncTrack();
  updateMediaSession();
});
</script>

<template>
  <div class="app">
    <Transition
      :name="showPlaylistEditor ? 'slide-left' : 'slide-right'"
      mode="out-in"
    >
      <!-- Player view -->
      <div
        v-if="!showPlaylistEditor"
        key="player"
        class="view view--player"
        :style="
          currentTrack?.image
            ? { '--track-image': `url('${currentTrack.image}')` }
            : {}
        "
      >
        <div class="view-top">
          <button
            class="icon-btn"
            type="button"
            aria-label="Show playlist"
            @click="toggleEditorSide"
          >
            <List :size="20" />
          </button>
        </div>

        <div class="player-body">
          <p class="track-title">
            {{ currentTrack?.title || "No songs enabled" }}
          </p>

          <div class="transport">
            <button
              class="transport-btn"
              type="button"
              aria-label="Previous track"
              @click="previousTrack"
            >
              <SkipBack :size="26" />
            </button>
            <button
              class="transport-btn transport-btn-primary"
              type="button"
              :aria-label="isPlaying ? 'Pause' : 'Play'"
              @click="togglePlay"
            >
              <Pause v-if="isPlaying" :size="28" />
              <Play v-else :size="28" />
            </button>
            <button
              class="transport-btn"
              type="button"
              aria-label="Next track"
              @click="nextTrack"
            >
              <SkipForward :size="26" />
            </button>
          </div>

          <div
            class="progress-bar"
            role="progressbar"
            :aria-valuenow="Math.round(progress)"
            aria-valuemin="0"
            aria-valuemax="100"
            @click="onSeeked"
          >
            <div class="progress-fill" :style="{ width: progress + '%' }" />
          </div>

          <button
            class="mute-btn"
            :class="{ muted: isMuted }"
            type="button"
            :aria-label="isMuted ? 'Unmute' : 'Mute'"
            @click="toggleMute"
          >
            {{ isMuted ? "Unmute" : "Mute" }}
          </button>
        </div>

        <div class="effects-row">
          <button
            v-for="effect in soundEffects"
            :key="effect.id"
            class="effect-btn"
            type="button"
            :aria-label="effect.label"
            :title="effect.label"
            @click="playEffect(effect)"
          >
            {{ effect.emoji }}
          </button>
        </div>
      </div>

      <!-- Playlist editor view -->
      <div v-else key="editor" class="view">
        <div class="view-top">
          <button
            class="icon-btn back-btn"
            type="button"
            aria-label="Back to player"
            @click="toggleEditorSide"
          >
            <ArrowLeft :size="22" />
          </button>
          <span class="editor-title">Playlist</span>
        </div>

        <div class="track-list">
          <div
            v-for="(track, index) in playlistState"
            :key="track.id"
            class="track-item"
          >
            <label class="track-label">
              <input
                type="checkbox"
                :checked="track.enabled"
                @change="toggleTrackEnabled(track.id)"
              />
              <span>{{ track.title }}</span>
            </label>
            <div class="track-actions">
              <button
                class="order-btn"
                type="button"
                :disabled="index === 0"
                :aria-label="`Move ${track.title} up`"
                @click="moveTrack(track.id, -1)"
              >
                ↑
              </button>
              <button
                class="order-btn"
                type="button"
                :disabled="index === playlistState.length - 1"
                :aria-label="`Move ${track.title} down`"
                @click="moveTrack(track.id, 1)"
              >
                ↓
              </button>
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <audio ref="player" @ended="onTrackEnded" @timeupdate="onTimeUpdate" />
  </div>
</template>

<style scoped>
/* ── Layout ── */
.app {
  min-height: 100dvh;
  overflow: hidden;
  position: relative;
}

/* ── Transition ── */
.slide-left-enter-active,
.slide-left-leave-active,
.slide-right-enter-active,
.slide-right-leave-active {
  transition:
    transform 0.32s ease,
    opacity 0.32s ease;
}

.slide-left-enter-from {
  transform: translateX(100%);
  opacity: 0;
}
.slide-left-leave-to {
  transform: translateX(-100%);
  opacity: 0;
}
.slide-right-enter-from {
  transform: translateX(-100%);
  opacity: 0;
}
.slide-right-leave-to {
  transform: translateX(100%);
  opacity: 0;
}

/* ── View shell ── */
.view {
  display: flex;
  flex-direction: column;
  min-height: 100dvh;
  max-width: 480px;
  margin: 0 auto;
  padding: 1.25rem 1.25rem 1.5rem;
}

.view--player {
  position: relative;
  isolation: isolate;
}

.view--player::before {
  content: "";
  position: fixed;
  inset: 0;
  background-image: var(--track-image);
  background-size: cover;
  background-position: center;
  transition: background-image 0.4s ease;
  z-index: -1;
}

.view--player::after {
  content: "";
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  backdrop-filter: blur(6px);
  -webkit-backdrop-filter: blur(6px);
  z-index: -1;
}

.view-top {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  min-height: 44px;
  margin-bottom: 0.5rem;
  flex-shrink: 0;
}

/* ── Icon button ── */
.icon-btn {
  width: 44px;
  height: 44px;
  border: none;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.18);
  color: #fff;
  font-size: 1.1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  backdrop-filter: blur(2px);
  -webkit-backdrop-filter: blur(2px);
}

/* ── Player body ── */
.player-body {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 2rem;
  padding: 1rem 0;
}

.track-title {
  font-size: 1.6rem;
  font-weight: 700;
  color: #fff;
  text-align: center;
  line-height: 1.3;
  text-shadow: 0 1px 4px rgba(0, 0, 0, 0.4);
}

/* ── Transport ── */
.transport {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

.transport-btn {
  border: none;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.18);
  color: #fff;
  font-size: 1.5rem;
  width: 72px;
  height: 72px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(2px);
  -webkit-backdrop-filter: blur(2px);
}

.transport-btn-primary {
  background: rgba(255, 255, 255, 0.18);
  color: #fff;
  width: 80px;
  height: 80px;
  border-radius: 50%;
  font-size: 1.7rem;
}

/* ── Progress bar ── */
.progress-bar {
  width: 100%;
  max-width: 280px;
  height: 4px;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 999px;
  cursor: pointer;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: #fff;
  border-radius: 999px;
  transition: width 0.25s linear;
  pointer-events: none;
}

/* ── Mute ── */
.mute-btn {
  width: 100%;
  max-width: 280px;
  border: none;
  border-radius: 999px;
  padding: 1.2rem 1rem;
  margin-top: 0.75rem;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  background: #ff3b30;
  color: #fff;
  box-shadow: 0 6px 18px rgba(255, 59, 48, 0.28);
}

.mute-btn.muted {
  background: #34c759;
  box-shadow: 0 6px 18px rgba(52, 199, 89, 0.28);
}

/* ── Effects row ── */
.effects-row {
  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;
  gap: 0.6rem;
  overflow-x: auto;
  overflow-y: hidden;
  -webkit-overflow-scrolling: touch;
  overscroll-behavior-x: contain;
  touch-action: pan-x;
  scrollbar-width: none;
  padding: 0.5rem 1.25rem;
  margin: 0 -1.25rem;
  flex-shrink: 0;
}

.effects-row::-webkit-scrollbar {
  display: none;
}

.effect-btn {
  flex: 0 0 auto;
  width: 64px;
  height: 64px;
  border: none;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.15);
  box-shadow: none;
  font-size: 1.75rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(2px);
  -webkit-backdrop-filter: blur(2px);
}

/* ── Playlist editor ── */
.back-btn {
  background: #1d1d1f;
  color: #fff;
  margin-right: 0.75rem;
}

.editor-title {
  font-size: 1.1rem;
  font-weight: 700;
  color: #111114;
  margin-right: auto;
}

.track-list {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  overflow-y: auto;
  flex: 1;
  padding-right: 0.25rem;
}

.track-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  padding: 0.9rem 1rem;
  border-radius: 18px;
  background: rgba(255, 255, 255, 0.72);
  border: 1px solid rgba(0, 0, 0, 0.07);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
}

.track-label {
  display: flex;
  align-items: center;
  gap: 0.65rem;
  font-size: 1rem;
  font-weight: 600;
  color: #111114;
  cursor: pointer;
  flex: 1;
  min-width: 0;
}

.track-label input[type="checkbox"] {
  width: 18px;
  height: 18px;
  flex-shrink: 0;
  cursor: pointer;
}

.track-label span {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.track-actions {
  display: flex;
  gap: 0.4rem;
  flex-shrink: 0;
}

.order-btn {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 12px;
  background: #f0f0f3;
  color: #1d1d1f;
  font-size: 1rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}

.order-btn:disabled {
  opacity: 0.3;
  cursor: default;
}
</style>
