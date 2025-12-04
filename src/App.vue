<template>
  <div class="app-container">
    <div class="tuner-casing">
      <div class="texture-overlay"></div>

      <!-- HUD Lens -->
      <div class="glass-hud">
        <div class="notch-top"></div>
        <div class="notch-bottom"></div>
      </div>

      <!-- SPLIT DRUM CONTAINER -->
      <div class="dual-drum-container">
        <!-- LEFT DRUM: CATEGORIES -->
        <div
          class="category-viewport"
          ref="catDrumRef"
          @scroll="handleCategoryScroll"
          @touchstart="onLeftDrumInteraction"
          @mousedown="onLeftDrumInteraction"
        >
          <ul class="category-list">
            <!-- Top Spacer for alignment -->
            <div class="list-spacer"></div>

            <li
              v-for="(cat, index) in categories"
              :key="cat"
              class="category-item"
              :class="{ 'cat-active': visualCategory === cat }"
              @click="scrollToCategory(cat)"
            >
              {{ cat }}
            </li>

            <!-- Bottom Spacer -->
            <div class="list-spacer"></div>
          </ul>
          <!-- Visual guide line -->
          <div class="cat-guide-line"></div>
        </div>

        <!-- RIGHT DRUM: STATIONS -->
        <div
          class="drum-viewport"
          ref="drumRef"
          @touchstart="onRightDrumInteraction"
          @mousedown="onRightDrumInteraction"
        >
          <ul class="station-list">
            <!-- Top Spacer for alignment -->
            <div class="list-spacer"></div>

            <li
              v-for="(station, index) in stations"
              :key="index"
              class="station-item"
              :style="getItemStyle(index)"
              @click="onStationClick(index)"
            >
              <div class="station-info">
                <span class="station-name">{{ station.name }}</span>
              </div>
            </li>

            <!-- Bottom Spacer -->
            <div class="list-spacer"></div>
          </ul>
        </div>
      </div>

      <!-- Floating Control Deck -->
      <div class="control-deck">
        <div class="deck-content">
          <button
            class="mech-btn"
            :class="{ 'is-playing': isPlaying, 'is-loading': isLoading }"
            @click="togglePlayback"
          >
            <div class="mech-btn-face">
              <div v-if="!isPlaying && !isLoading" class="icon-play"></div>
              <div v-else class="icon-pause">
                <div class="bar"></div>
                <div class="bar"></div>
              </div>
            </div>
          </button>

          <div class="system-status">
            <div class="status-content">
              SYSTEM:
              <span :class="{ blink: isLoading }">{{ statusText }}</span>
              <span class="divider">//</span>
              CH: <span>{{ currentChannelId }}</span>
            </div>
          </div>
        </div>
      </div>

      <audio
        ref="audioRef"
        :src="currentStation.url"
        @ended="onAudioEnded"
        @error="onAudioError"
        @waiting="isLoading = true"
        @playing="isLoading = false"
      ></audio>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from "vue";

// --- DATA ---
const stations = ref([
  {
    name: "清晨音乐台",
    url: "https://lhttp.qtfm.cn/live/4915/64k.mp3",
    category: "CHILL",
  },
  { name: "早安调频", url: "", category: "CHILL" },
  {
    name: "80 后音乐",
    url: "https://lhttp.qingting.fm/live/20207761/64k.mp3",
    category: "RETRO",
  },
  { name: "90s 金曲", url: "", category: "RETRO" },
  {
    name: "怀集音乐",
    url: "http://lhttp.qingting.fm/live/4804/64k.mp3",
    category: "LOCAL",
  },
  { name: "Cyber Tokyo", url: "", category: "FUT" },
  { name: "Neo-Seoul", url: "", category: "FUT" },
  { name: "Heavy Metal", url: "", category: "ROCK" },
  { name: "Hard Rock", url: "", category: "ROCK" },
  { name: "Deep Space", url: "", category: "AMBIENT" },
  { name: "Nebula", url: "", category: "AMBIENT" },
  { name: "Synthwave", url: "", category: "SYNTH" },
  { name: "Industrial", url: "", category: "MECH" },
  { name: "Factory Floor", url: "", category: "MECH" },
  { name: "Lo-Fi Grid", url: "", category: "LOFI" },
  { name: "Void Signal", url: "", category: "DARK" },
  { name: "Static FM", url: "", category: "NOISE" },
]);

const drumRef = ref(null);
const catDrumRef = ref(null);
const audioRef = ref(null);
const isPlaying = ref(false);
const isLoading = ref(false);
const activeIndex = ref(3);
const isProgrammaticScroll = ref(false);
const isResizing = ref(false);

// State for Interaction
const isBrowsingCategory = ref(false);
const manualVisualCategory = ref("");
const isSyncingLeft = ref(false); // LOCK: Prevents circular updates
let syncLockTimer = null;

// Physics State
const itemTransforms = ref(
  stations.value.map(() => ({
    rotateX: 0,
    scale: 1,
    opacity: 0.4,
    blur: 0,
    z: 0,
    active: false,
  }))
);

const itemHeight = 80;
const catItemHeight = 50;

// --- COMPUTED ---
const currentStation = computed(() => stations.value[activeIndex.value] || {});
const categories = computed(() => [
  ...new Set(stations.value.map((s) => s.category)),
]);
const activeCategory = computed(() => currentStation.value.category);

const visualCategory = computed(() => {
  return isBrowsingCategory.value
    ? manualVisualCategory.value
    : activeCategory.value;
});

const currentChannelId = computed(() =>
  (activeIndex.value + 1).toString().padStart(3, "0")
);
const statusText = computed(() => {
  if (isLoading.value) return "BUFFERING";
  if (isPlaying.value) return "ONLINE";
  return "STANDBY";
});

// --- METHODS ---

// 1. LEFT DRUM SCROLL (Auto-Select Logic)
let catScrollDebounce = null;

const handleCategoryScroll = () => {
  if (!catDrumRef.value) return;

  // LOCK: If this scroll is caused by "Right driving Left", IGNORE it.
  if (isSyncingLeft.value) return;

  isBrowsingCategory.value = true;
  clearTimeout(catScrollDebounce);

  // Update visual highlight
  const scrollTop = catDrumRef.value.scrollTop;
  const centerIndex = Math.round(scrollTop / catItemHeight);

  if (centerIndex >= 0 && centerIndex < categories.value.length) {
    manualVisualCategory.value = categories.value[centerIndex];
  }

  // Debounce: Only trigger if user STOPS scrolling left drum
  catScrollDebounce = setTimeout(() => {
    // Only jump if selected category is different
    if (
      manualVisualCategory.value &&
      manualVisualCategory.value !== activeCategory.value
    ) {
      performCategoryJump(manualVisualCategory.value);
    }
  }, 300);
};

// 2. CLICK CATEGORY (Direct Jump)
const scrollToCategory = (category) => {
  clearTimeout(catScrollDebounce);
  performCategoryJump(category);
};

// Internal helper for jumping
const performCategoryJump = (category) => {
  isBrowsingCategory.value = false;
  manualVisualCategory.value = category;

  const targetIndex = stations.value.findIndex((s) => s.category === category);
  if (targetIndex !== -1) {
    onStationClick(targetIndex);

    nextTick(() => {
      syncCategoryDrum(true);
    });
  }
};

const onLeftDrumInteraction = () => {
  isSyncingLeft.value = false;
  if (syncLockTimer) clearTimeout(syncLockTimer);
};

// 3. RIGHT DRUM INTERACTION
const onRightDrumInteraction = () => {
  clearTimeout(catScrollDebounce);
  isBrowsingCategory.value = false;
};

// 4. SYNC LEFT DRUM TO ACTIVE STATION (Right drives Left)
const syncCategoryDrum = (force = false) => {
  if (!catDrumRef.value) return;

  if (isBrowsingCategory.value && !force) return;

  const catIndex = categories.value.indexOf(activeCategory.value);
  if (catIndex !== -1) {
    const targetScroll = catIndex * catItemHeight;

    if (Math.abs(catDrumRef.value.scrollTop - targetScroll) < 5 && !force)
      return;

    isSyncingLeft.value = true;
    if (syncLockTimer) clearTimeout(syncLockTimer);

    catDrumRef.value.scrollTo({ top: targetScroll, behavior: "smooth" });

    syncLockTimer = setTimeout(() => {
      isSyncingLeft.value = false;
    }, 800);
  }
};

// --- EXISTING METHODS ---

const updateMediaSession = () => {
  if (!("mediaSession" in navigator)) return;
  navigator.mediaSession.metadata = new MediaMetadata({
    title: currentStation.value.name,
    artist: `CH ${currentChannelId.value} - ${currentStation.value.category}`,
    album: "Retro Future Radio",
  });
  navigator.mediaSession.setActionHandler("play", () => playStation());
  navigator.mediaSession.setActionHandler("pause", () => {
    audioRef.value?.pause();
    isPlaying.value = false;
  });
  navigator.mediaSession.setActionHandler("previoustrack", () =>
    switchChannel(-1)
  );
  navigator.mediaSession.setActionHandler("nexttrack", () => switchChannel(1));
};

const switchChannel = (direction) => {
  let newIndex = activeIndex.value + direction;
  if (newIndex < 0) newIndex = stations.value.length - 1;
  if (newIndex >= stations.value.length) newIndex = 0;

  isProgrammaticScroll.value = true;
  activeIndex.value = newIndex;
  scrollToItem(newIndex);

  setTimeout(() => {
    isProgrammaticScroll.value = false;
  }, 500);
  playStation();
};

const onStationClick = (index) => {
  isProgrammaticScroll.value = true;
  activeIndex.value = index;
  scrollToItem(index);
  playStation();
};

const updatePhysics = () => {
  if (!drumRef.value) return;
  const drum = drumRef.value;
  // Safety check to avoid NaN crash
  if (!drum.clientHeight || drum.clientHeight === 0) return;

  const viewportCenter = drum.scrollTop + drum.clientHeight / 2;
  let closestDist = Infinity;
  let closestIndex = -1;

  stations.value.forEach((_, i) => {
    // FIX: Using Spacers means we calculate item center directly
    // The spacer pushes the first item (i=0) to exactly (spacerHeight + itemHeight/2)
    // Spacer is 50% of container height.
    // So itemCenter = (containerHeight / 2) + (i * itemHeight) + (itemHeight / 2)
    // Wait, spacers are part of scroll flow.
    // Correct logic: Item offsetTop will include the top spacer.
    // Spacer height is approx 50vh - half item height.

    // Instead of querying DOM (slow), we use math assuming spacer is correct.
    // Padding logic was: padding + i*h + h/2.
    // Spacer logic is identical: spacerHeight + i*h + h/2.
    // Let's use the container half-height as the "Start Line"

    const startOffset = drum.clientHeight / 2 - itemHeight / 2;
    const itemCenter = startOffset + i * itemHeight + itemHeight / 2;
    const distance = viewportCenter - itemCenter;

    if (Math.abs(distance) < Math.abs(closestDist)) {
      closestDist = distance;
      closestIndex = i;
    }

    const normalizedDist = distance / 220;
    const rotateX = normalizedDist * -45;
    const scale = Math.max(0.75, 1.2 - Math.abs(normalizedDist) * 0.5);
    const opacity = Math.max(0.25, 1 - Math.abs(normalizedDist));
    const blur = Math.abs(normalizedDist) * 3;
    const z = Math.abs(normalizedDist) * -60;
    const isActive = Math.abs(normalizedDist) < 0.2;

    itemTransforms.value[i] = { rotateX, scale, opacity, blur, z, isActive };
  });

  if (
    closestIndex !== -1 &&
    closestIndex !== activeIndex.value &&
    !isProgrammaticScroll.value
  ) {
    activeIndex.value = closestIndex;
    triggerHaptic();
  }

  // Continuously sync left drum unless user is messing with it
  syncCategoryDrum();
};

const togglePlayback = () => {
  if (!audioRef.value) return;
  if (isPlaying.value || isLoading.value) {
    audioRef.value.pause();
    isPlaying.value = false;
    isLoading.value = false;
  } else {
    playStation();
  }
};

const playStation = async () => {
  if (!audioRef.value) return;
  const url = currentStation.value.url;
  updateMediaSession();
  isLoading.value = true;

  if (!url || url.startsWith("PREVIEW") || url === "") {
    console.log("Mock Station");
    setTimeout(() => {
      isLoading.value = false;
      isPlaying.value = true;
    }, 500);
    return;
  }

  try {
    audioRef.value.src = url;
    audioRef.value.load();
    await audioRef.value.play();
    isPlaying.value = true;
    isLoading.value = false;
  } catch (e) {
    if (e.name !== "AbortError") {
      console.error("Stream failed", e);
      isPlaying.value = false;
      isLoading.value = false;
    }
  }
};

const onAudioEnded = () => {
  isPlaying.value = false;
  isLoading.value = false;
};
const onAudioError = () => {
  console.warn("Audio error");
  isPlaying.value = false;
  isLoading.value = false;
};
const triggerHaptic = () => {
  if (navigator.vibrate) navigator.vibrate(10);
};

let scrollTimeout = null;
const handleScroll = () => {
  if (isResizing.value) {
    window.requestAnimationFrame(updatePhysics);
    return;
  }
  window.requestAnimationFrame(updatePhysics);
  clearTimeout(scrollTimeout);
  scrollTimeout = setTimeout(() => {
    if (isProgrammaticScroll.value) {
      isProgrammaticScroll.value = false;
      return;
    }
    if (isPlaying.value) {
      playStation();
      if (navigator.vibrate) navigator.vibrate([10, 30, 10]);
    }
  }, 150);
};

let resizeTimeout = null;
const handleResize = () => {
  isResizing.value = true;
  clearTimeout(resizeTimeout);
  resizeTimeout = setTimeout(() => {
    if (drumRef.value) {
      centerActiveItem();
      updatePhysics();
    }
    setTimeout(() => {
      isResizing.value = false;
    }, 300);
  }, 150);
};

const scrollToItem = (index) => {
  if (!drumRef.value) return;
  // Calculate scroll based on spacer height
  // Spacer = 50% height - half item height
  // ScrollTop 0 = Top of spacer.
  // We want item centered. Item center is at Spacer + i*h + h/2.
  // Viewport center is at 50% height.
  // So Target ScrollTop = (Spacer + i*h + h/2) - (ViewportHeight / 2)
  // Spacer = (ViewportHeight / 2) - (h / 2)
  // Target = ((VH/2 - h/2) + i*h + h/2) - VH/2
  // Target = VH/2 - h/2 + i*h + h/2 - VH/2 = i*h
  // Math simplifies beautifully to just index * itemHeight!
  const targetScroll = index * itemHeight;
  drumRef.value.scrollTo({ top: targetScroll, behavior: "smooth" });
};

const getItemStyle = (index) => {
  const t = itemTransforms.value[index];
  if (!t) return {};
  return {
    transform: `perspective(800px) rotateX(${t.rotateX}deg) scale(${t.scale}) translateZ(${t.z}px)`,
    opacity: t.opacity,
    filter: `blur(${t.blur}px)`,
    color: t.isActive ? "var(--accent-amber)" : "var(--text-dim)",
    textShadow: t.isActive ? "0 0 10px rgba(255,176,0,0.3)" : "none",
    pointerEvents: t.isActive ? "none" : "auto",
  };
};

const centerActiveItem = () => {
  if (drumRef.value) {
    const targetScroll = activeIndex.value * itemHeight;
    drumRef.value.scrollTo({ top: targetScroll, behavior: "auto" });
  }
};

onMounted(() => {
  if (drumRef.value) {
    drumRef.value.addEventListener("scroll", handleScroll);
    window.addEventListener("resize", handleResize);
    updatePhysics();
    nextTick(() => {
      setTimeout(() => {
        centerActiveItem();
        updatePhysics();
      }, 50);
    });
    if ("mediaSession" in navigator) updateMediaSession();
  }
});

onUnmounted(() => {
  if (drumRef.value) {
    drumRef.value.removeEventListener("scroll", handleScroll);
  }
  window.removeEventListener("resize", handleResize);
});
</script>

<style>
body {
  margin: 0;
  background-color: #0a0a0c;
  overflow: hidden;
}
</style>

<style scoped>
.app-container {
  --bg-color: #0a0a0c;
  --tuner-black: #111113;
  --text-dim: #4a4a50;
  --accent-amber: #ffb000;
  --accent-glow: rgba(255, 176, 0, 0.4);
  --glass-border: rgba(255, 255, 255, 0.1);
  --font-mono: "Courier New", Courier, monospace;
  --item-h: 80px;

  width: 100vw;
  height: 100vh;
  background-color: var(--bg-color);
  color: white;
  font-family: var(--font-mono);
  display: flex;
  justify-content: center;
  align-items: center;
}

.tuner-casing {
  position: relative;
  width: 100%;
  height: 100%;
  background: linear-gradient(145deg, #18181b, #0d0d0f);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.texture-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.7' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.04'/%3E%3C/svg%3E");
  pointer-events: none;
  z-index: 1;
}

/* --- DUAL DRUM LAYOUT --- */
.dual-drum-container {
  flex: 1;
  display: flex;
  position: relative;
  height: 100%;
  overflow: hidden;
}

/* --- SPACER SYSTEM FOR PERFECT CENTERING --- */
/* This replaces the unreliable padding-top/bottom */
.list-spacer {
  /* Height = 50% of container - Half Item Height */
  /* This ensures the first/last items sit exactly in the middle */
  height: calc(50% - 40px); /* 40px is half of 80px item height */
  width: 100%;
  pointer-events: none;
}

/* 1. LEFT DRUM: CATEGORIES (Coarse) */
.category-viewport {
  width: 25%; /* Takes up left side */
  height: 100%;
  overflow-y: scroll;
  overflow-x: hidden;
  background: rgba(0, 0, 0, 0.2); /* Slightly darker track */
  border-right: 1px solid rgba(255, 255, 255, 0.05);
  position: relative;
  z-index: 5;
  scrollbar-width: none;
  /* Shadow mask */
  mask-image: linear-gradient(
    to bottom,
    transparent 0%,
    black 20%,
    black 80%,
    transparent 100%
  );
  scroll-snap-type: y mandatory; /* Add snap for better feel */
}
.category-viewport::-webkit-scrollbar {
  display: none;
}

.category-list {
  margin: 0;
  padding: 0;
  list-style: none;
  text-align: right; /* Align text close to the main drum */
  height: 100%; /* Ensure spacers work */
}

/* Special spacer override for category list (smaller items) */
.category-list .list-spacer {
  height: calc(50% - 25px); /* 25px is half of 50px item height */
}

.category-item {
  height: 50px; /* Smaller height */
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 20px;
  font-size: 0.7rem;
  font-weight: bold;
  letter-spacing: 1px;
  color: #444;
  cursor: pointer;
  scroll-snap-align: center; /* Snap items */
  transition: all 0.2s ease;
  user-select: none;
  -webkit-user-select: none;
}

.category-item.cat-active {
  color: var(--accent-amber);
  text-shadow: 0 0 5px var(--accent-glow);
  font-size: 0.85rem;
}

/* 2. RIGHT DRUM: STATIONS (Fine) */
.drum-viewport {
  flex: 1;
  height: 100%;
  overflow-y: scroll;
  overflow-x: hidden;
  scroll-snap-type: y mandatory;
  scroll-behavior: smooth;
  perspective: 800px;
  perspective-origin: center center;
  scrollbar-width: none;
  mask-image: linear-gradient(
    to bottom,
    transparent 0%,
    black 15%,
    black 65%,
    transparent 85%
  );
  z-index: 2;
}
.drum-viewport::-webkit-scrollbar {
  display: none;
}

.station-list {
  margin: 0;
  padding: 0;
  list-style: none;
  transform-style: preserve-3d;
  height: 100%;
}

.station-item {
  height: var(--item-h);
  display: flex;
  align-items: center;
  justify-content: center;
  scroll-snap-align: center;
  scroll-snap-stop: normal;
  will-change: transform, opacity, filter;
  cursor: pointer;
  -webkit-font-smoothing: antialiased;
}

.station-name {
  font-size: 1.4rem;
  font-weight: 700;
  letter-spacing: 2px;
  text-transform: uppercase;
  text-align: center;
}

/* --- HUD LENS (Overlays both) --- */
.glass-hud {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: var(--item-h);
  transform: translateY(-50%);
  z-index: 10;
  pointer-events: none;
  /* Glass gradient */
  background: linear-gradient(
    to bottom,
    rgba(255, 255, 255, 0.01),
    rgba(255, 255, 255, 0.03) 50%,
    rgba(255, 255, 255, 0.01)
  );
  border-top: 1px solid var(--glass-border);
  border-bottom: 1px solid var(--glass-border);
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.5);
}

/* Notches align with Main Drum Center */
/* CALCULATION: 25% (Left Drum) + 75% (Right Drum) / 2 = 62.5% */
.notch-top,
.notch-bottom {
  position: absolute;
  left: 62.5%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 5px solid transparent;
  border-right: 5px solid transparent;
}
.notch-top {
  top: 0;
  border-top: 6px solid var(--accent-amber);
  filter: drop-shadow(0 0 4px var(--accent-amber));
}
.notch-bottom {
  bottom: 0;
  border-bottom: 6px solid var(--accent-amber);
  filter: drop-shadow(0 0 4px var(--accent-amber));
}

/* --- CONTROL DECK --- */
.control-deck {
  position: absolute;
  bottom: 40px;
  width: 100%;
  display: flex;
  flex-direction: column;
  /* ALIGNMENT FIX: Use padding to align items to the right drum's center */
  padding-left: 25%;
  box-sizing: border-box;
  align-items: center; /* Center items within the remaining 75% width */
  justify-content: flex-end;
  z-index: 20;
  pointer-events: none;
}

.deck-content {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.mech-btn {
  pointer-events: auto;
  width: 72px;
  height: 72px;
  border-radius: 50%;
  border: none;
  cursor: pointer;
  position: relative;
  outline: none;
  margin-bottom: 25px;
  -webkit-tap-highlight-color: transparent;
  background: linear-gradient(145deg, #2b2b30, #1a1a1c);
  box-shadow: 5px 5px 15px rgba(0, 0, 0, 0.5),
    -2px -2px 10px rgba(255, 255, 255, 0.05), inset 0 0 0 1px rgba(0, 0, 0, 0.8);
  transition: transform 0.1s cubic-bezier(0.4, 0, 0.2, 1);
}
.mech-btn-face {
  position: absolute;
  top: 6px;
  left: 6px;
  right: 6px;
  bottom: 6px;
  border-radius: 50%;
  background: linear-gradient(145deg, #131315, #1c1c1f);
  box-shadow: inset 3px 3px 8px #08080a,
    inset -2px -2px 5px rgba(255, 255, 255, 0.05);
  display: flex;
  align-items: center;
  justify-content: center;
}
.mech-btn:active {
  transform: scale(0.95);
}
.mech-btn.is-playing .mech-btn-face {
  background: #0f0f11;
  box-shadow: inset 5px 5px 12px #050506, inset -5px -5px 12px #25252b;
}
.mech-btn.is-playing::after {
  content: "";
  position: absolute;
  top: -2px;
  left: -2px;
  right: -2px;
  bottom: -2px;
  border-radius: 50%;
  box-shadow: 0 0 25px rgba(255, 176, 0, 0.15);
  pointer-events: none;
}

.icon-play {
  width: 0;
  height: 0;
  border-top: 9px solid transparent;
  border-bottom: 9px solid transparent;
  border-left: 16px solid #444;
  margin-left: 4px;
  transition: all 0.3s;
}
.icon-pause {
  display: flex;
  gap: 4px;
  height: 18px;
}
.bar {
  width: 5px;
  height: 100%;
  background-color: #333;
  box-shadow: inset 1px 1px 2px rgba(0, 0, 0, 0.8);
}
.mech-btn.is-playing .bar {
  background-color: var(--accent-amber);
  box-shadow: 0 0 8px var(--accent-glow);
}

.system-status {
  width: 100%;
  text-align: center;
  font-size: 0.7rem;
  color: #444;
  letter-spacing: 1.5px;
  text-shadow: 0 1px 1px rgba(0, 0, 0, 1);
}
.status-content span {
  color: var(--accent-amber);
  font-weight: bold;
}
.status-content span.divider {
  margin: 0 10px;
  color: #333;
  font-weight: normal;
}
.status-content span.blink {
  animation: blink 1s infinite;
}
@keyframes blink {
  50% {
    opacity: 0.5;
  }
}

/* --- LANDSCAPE MODE --- */
@media (orientation: landscape) and (max-height: 600px) {
  .control-deck {
    top: 0;
    bottom: auto;
    left: 0;
    right: 0;
    height: 100%;
    display: block;
    padding-left: 0; /* Reset padding for landscape custom positioning */
    pointer-events: none;
  }
  .mech-btn {
    position: absolute;
    top: 35px;
    right: max(40px, env(safe-area-inset-right));
    width: 60px;
    height: 60px;
    pointer-events: auto;
  }
  .system-status {
    position: absolute;
    top: 45px;
    left: max(40px, env(safe-area-inset-left));
    width: auto;
    text-align: left;
    font-size: 0.75rem;
    pointer-events: auto;
  }
}
</style>
