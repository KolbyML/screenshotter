<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, watch } from "vue";
import { Image } from "@tauri-apps/api/image";
import { writeImage } from "@tauri-apps/plugin-clipboard-manager";
import { Store } from '@tauri-apps/plugin-store';
import { getScreenshotableWindows, getWindowScreenshot } from "tauri-plugin-screenshots-api";

let store: Store | null = null;

// Screenshot windows
const windows_list = ref<Array<{ id: number; name: string }>>([]);
const selectedWindowId = ref<number | null>(null);

const selectedWindow = computed(() => {
    if (selectedWindowId.value == null) return null;
    return windows_list.value.find(w => w.id === selectedWindowId.value) ?? null;
});

type CaptureEntry = { ok: boolean; when: number; msg: string };
const lastCapture = ref<CaptureEntry | null>(null);
const captureHistory = ref<CaptureEntry[]>([]); // keep last 10

function recordCapture(ok: boolean, msg: string) {
    const entry = { ok, when: Date.now(), msg };
    lastCapture.value = entry;
    captureHistory.value.unshift(entry);
    if (captureHistory.value.length > 10) captureHistory.value.pop();
}

function fmtTime(ts: number) {
    // e.g., 22:07:13 • 2025-08-26
    const d = new Date(ts);
    const hh = `${d.getHours()}`.padStart(2, "0");
    const mm = `${d.getMinutes()}`.padStart(2, "0");
    const ss = `${d.getSeconds()}`.padStart(2, "0");
    const yyyy = d.getFullYear();
    const mo = `${d.getMonth() + 1}`.padStart(2, "0");
    const da = `${d.getDate()}`.padStart(2, "0");
    return `${hh}:${mm}:${ss} • ${yyyy}-${mo}-${da}`;
}

async function get_windows() {
    try {
        const windows = await getScreenshotableWindows();
        windows_list.value = windows;

        if (!selectedWindow.value && windows_list.value.length > 0) {
            selectedWindowId.value = windows_list.value[0].id;
        }
    } catch (err) {
        recordCapture(false, "Failed to fetch windows");
        console.warn("Failed to fetch windows:", err);
    }
}

// Interval loop
const seconds = ref(3);
const isRunning = ref(false);
let intervalId: number | null = null;
let isCapturing = false;


async function takeScreenshotAndFillClipboard() {
    if (isCapturing) return;
    isCapturing = true;

    let image: Image | null = null;
    try {
        if (!selectedWindow.value) {
            console.warn("No window selected");
            return;
        }

        const path = await getWindowScreenshot(selectedWindow.value.id);
        console.log("Saved screenshot at:", path);

        image = await Image.fromPath(path);
        await writeImage(image);

        console.log("✅ Screenshot copied to clipboard (image)");
        recordCapture(true, "Screenshot copied to clipboard");
    } catch (err) {
        recordCapture(false, `Failed to copy screenshot to clipboard: ${err}`);
        console.error("Failed to copy screenshot to clipboard:", err);
    } finally {
        try {
            if (image) await image.close();
        } catch (err) {
            recordCapture(false, `Failed to close image: ${err}`);
            console.warn("Failed to close image:", err);
        }
        isCapturing = false;
    }
}

function toggleInterval() {
    if (isRunning.value) {
        if (intervalId) clearInterval(intervalId);
        intervalId = null;
        isRunning.value = false;
    } else {
        const delay = Math.min(3600, Math.max(1, seconds.value)) * 1000;
        intervalId = window.setInterval(takeScreenshotAndFillClipboard, delay);
        isRunning.value = true;
    }
}

onMounted(async () => {
    try {
        store = await Store.load("store.json");
        const savedSeconds = await store.get<number>("seconds");
        if (typeof savedSeconds === "number" && Number.isFinite(savedSeconds)) {
            seconds.value = Math.min(3600, Math.max(1, Math.round(savedSeconds)));
        }
    } catch (err) {
        console.warn("Failed to load settings:", err);
    }

    await get_windows();
});

watch(seconds, async (val) => {
    if (store) {
        await store.set("seconds", Math.min(3600, Math.max(1, Math.round(val))));
        await store.save();        
    }
});

onBeforeUnmount(() => {
    if (intervalId) clearInterval(intervalId);
    intervalId = null;
});
</script>


<template>
    <div class="picker-row">
        <label class="picker-label">Window</label>

        <div class="picker-wrap">
            <select class="picker-select" v-model="selectedWindowId" :disabled="windows_list.length === 0"
                aria-label="Select window to capture">
                <option v-if="windows_list.length === 0" :value="null" disabled>
                    No windows available
                </option>
                <option v-for="window in windows_list" :key="window.id" :value="window.id">
                    {{ window.name }}
                </option>
            </select>

            <button class="picker-refresh" @click="get_windows" title="Refresh windows list">
                Refresh
            </button>
        </div>
    </div>

    <div v-if="lastCapture" class="status">
        <span class="dot" :class="lastCapture.ok ? 'ok' : 'err'"></span>
        <span class="status-text">
            {{ lastCapture.ok ? 'Success' : 'Failed' }} — {{ lastCapture.msg }}
        </span>
        <span class="ts">{{ fmtTime(lastCapture.when) }}</span>
    </div>

    <div class="controls">
        <label>
            Interval (seconds):
            <input type="number" v-model.number="seconds" min="1" max="3600" />
        </label>
        <button @click="toggleInterval">
            {{ isRunning ? "Stop" : "Start" }}
        </button>
        <button @click="takeScreenshotAndFillClipboard">Capture Now</button>
    </div>

    <details class="history">
        <summary>Capture history (last 10)</summary>
        <ul>
            <li v-for="(h, i) in captureHistory" :key="i">
                <span class="dot tiny" :class="h.ok ? 'ok' : 'err'"></span>
                <span>{{ h.ok ? 'Success' : 'Failed' }} — {{ h.msg }}</span>
                <span class="ts">{{ fmtTime(h.when) }}</span>
            </li>
        </ul>
    </details>
</template>

<style scoped>
.picker-row {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    max-width: 640px;
}

.picker-label {
    min-width: 72px;
    font-weight: 600;
}

.picker-wrap {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    flex: 1;
    /* take remaining space */
    min-width: 0;
}

.picker-select {
    flex: 1;
    /* grow to fill */
    min-width: 220px;
    padding: 0.5rem 0.75rem;
}

.picker-refresh {
    margin-left: auto;
    /* push refresh to the right edge */
    padding: 0.5rem 0.75rem;
}

.selected-indicator {
    margin: 0.5rem 0 1rem;
    color: #555;
}

.controls {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    margin-top: 0.5rem;
}


.logo.vite:hover {
    filter: drop-shadow(0 0 2em #747bff);
}

.logo.vue:hover {
    filter: drop-shadow(0 0 2em #249b73);
}

.status {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: .5rem .75rem;
    border-radius: .5rem;
    background: #f7f7f7;
    border: 1px solid #e7e7e7;
    max-width: 720px;
}

.dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    display: inline-block;
}

.dot.ok {
    background: #16a34a;
}

/* green */
.dot.err {
    background: #dc2626;
}

/* red */
.status-text {
    font-weight: 600;
    color: black;
}

.ts {
    margin-left: auto;
    font-variant-numeric: tabular-nums;
    color: #666;
    font-size: 0.9rem;
}

/* history */
.history {
    margin-top: .75rem;
}

.history ul {
    margin: .5rem 0 0;
    padding-left: 1rem;
}

.history li {
    display: flex;
    align-items: center;
    gap: .5rem;
    padding: .25rem 0;
}

.dot.tiny {
    width: 8px;
    height: 8px;
}
</style>
<style>
:root {
    font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
    font-size: 16px;
    line-height: 24px;
    font-weight: 400;

    color: #0f0f0f;
    background-color: #f6f6f6;

    font-synthesis: none;
    text-rendering: optimizeLegibility;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    -webkit-text-size-adjust: 100%;
}

.container {
    margin: 0;
    padding-top: 10vh;
    display: flex;
    flex-direction: column;
    justify-content: center;
    text-align: center;
}

.logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: 0.75s;
}

.logo.tauri:hover {
    filter: drop-shadow(0 0 2em #24c8db);
}

.row {
    display: flex;
    justify-content: center;
}

a {
    font-weight: 500;
    color: #646cff;
    text-decoration: inherit;
}

a:hover {
    color: #535bf2;
}

h1 {
    text-align: center;
}

input,
button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #0f0f0f;
    background-color: #ffffff;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
}

button {
    cursor: pointer;
}

button:hover {
    border-color: #396cd8;
}

button:active {
    border-color: #396cd8;
    background-color: #e8e8e8;
}

input,
button {
    outline: none;
}

#greet-input {
    margin-right: 5px;
}

@media (prefers-color-scheme: dark) {
    :root {
        color: #f6f6f6;
        background-color: #2f2f2f;
    }

    a:hover {
        color: #24c8db;
    }

    input,
    button {
        color: #ffffff;
        background-color: #0f0f0f98;
    }

    button:active {
        background-color: #0f0f0f69;
    }
}
</style>