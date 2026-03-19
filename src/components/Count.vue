<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

import sndClickB from '@/assets/Sounds/click-b.ogg'
import sndClickA from '@/assets/Sounds/click-a.ogg'
import sndTapB from '@/assets/Sounds/tap-b.ogg'
import sndSwitchB from '@/assets/Sounds/switch-b.ogg'

// Template refs for audio elements
const sfxToggleOn = ref(null);
const sfxToggleOff = ref(null);
const sfxMilestone = ref(null);
const sfxComplete = ref(null);

// Set the program start and target dates
const startDate = new Date('2026-01-18T00:00:00');
const targetDate = new Date('2026-12-18T23:59:59');
const breakDays = 20;

// Reactive state
const soundEnabled = ref(false);
const lastMilestone = ref(-1);
let completionSoundPlayed = false;

const progressPercentage = ref(0);
const roundedProgress = ref(0);
const statusMessage = ref('Loading status...');
const remainingDays = ref(0);
const currentDateFormatted = ref('-');
const targetDateFormatted = ref('-');

const dateFormatter = new Intl.DateTimeFormat('en-US', {
    month: 'short',
    day: 'numeric',
    year: 'numeric'
});

function playSound(soundElement) {
    if (!soundEnabled.value || !soundElement) {
        return;
    }
    soundElement.currentTime = 0;
    soundElement.play().catch(() => {});
}

function normalizeDate(dateValue) {
    const date = new Date(dateValue);
    date.setHours(0, 0, 0, 0);
    return date;
}

function countWeekdays(start, end) {
    const startDateOnly = normalizeDate(start);
    const endDateOnly = normalizeDate(end);

    if (endDateOnly < startDateOnly) {
        return 0;
    }

    let weekdays = 0;
    const cursor = new Date(startDateOnly);

    while (cursor <= endDateOnly) {
        const day = cursor.getDay();
        if (day !== 0 && day !== 6) {
            weekdays += 1;
        }
        cursor.setDate(cursor.getDate() + 1);
    }

    return weekdays;
}

function updateProgress() {
    const now = new Date();
    const totalWeekdays = countWeekdays(startDate, targetDate);
    const effectiveTotalDays = Math.max(1, totalWeekdays - breakDays);
    const elapsedWeekdays = countWeekdays(startDate, now);
    const effectiveElapsedDays = Math.max(0, elapsedWeekdays);

    remainingDays.value = Math.max(0, effectiveTotalDays - effectiveElapsedDays);

    let pct = (effectiveElapsedDays / effectiveTotalDays) * 100;
    let status = 'Program in progress';

    if (now <= startDate) {
        pct = 0;
        status = 'Program has not started yet';
    } else if (now >= targetDate) {
        pct = 100;
        status = 'Program completed';
    }

    pct = Math.min(100, Math.max(0, pct));

    progressPercentage.value = pct;
    roundedProgress.value = Math.floor(pct);
    statusMessage.value = status;
    currentDateFormatted.value = dateFormatter.format(now);
    targetDateFormatted.value = dateFormatter.format(targetDate);

    const currentMilestone = Math.floor(roundedProgress.value / 10);
    if (currentMilestone > lastMilestone.value && roundedProgress.value < 100) {
        lastMilestone.value = currentMilestone;
        playSound(sfxMilestone.value);
    }

    if (roundedProgress.value === 100 && !completionSoundPlayed) {
        completionSoundPlayed = true;
        playSound(sfxComplete.value);
    }
}

function toggleSound() {
    soundEnabled.value = !soundEnabled.value;

    if (soundEnabled.value) {
        sfxToggleOn.value?.play().catch(() => {});
    } else {
        sfxToggleOff.value?.play().catch(() => {});
    }
}

let intervalId = null;

onMounted(() => {
    updateProgress();
    intervalId = setInterval(updateProgress, 1000);
});

onUnmounted(() => {
    if (intervalId) clearInterval(intervalId);
});
</script>


<template>
    <main class="app-shell">
        <section id="progress-container" aria-live="polite">
            <div class="top-row">
                <span class="badge">AgTech Dashboard</span>
                <button class="sound-btn" type="button" :aria-pressed="String(soundEnabled)" @click="toggleSound">
                    {{ soundEnabled ? '🔊 Sound On' : '🔇 Sound Off' }}
                </button>
            </div>

            <h1>AgTech Program</h1>
            <p class="subtitle">Tracking program progress to completion date</p>

            <div id="progress-bar" role="progressbar" aria-valuemin="0" aria-valuemax="100" :aria-valuenow="roundedProgress">
                <div id="progress-fill" :style="{ width: progressPercentage + '%' }"></div>
            </div>

            <div class="progress-meta">
                <p id="progress-text">{{ roundedProgress }}%</p>
                <p id="status-text">{{ statusMessage }}</p>
            </div>

            <div class="details-grid">
                <article class="detail-card">
                    <div class="detail-head">
                        <img src="@/assets/Vector/Green/icon_outline_circle.svg" alt="" aria-hidden="true" class="detail-icon">
                        <h2>Days Left</h2>
                    </div>
                    <p>{{ remainingDays }}</p>
                </article>
                <article class="detail-card">
                    <div class="detail-head">
                        <img src="@/assets/Vector/Green/icon_outline_square.svg" alt="" aria-hidden="true" class="detail-icon">
                        <h2>Today</h2>
                    </div>
                    <p>{{ currentDateFormatted }}</p>
                </article>
                <article class="detail-card">
                    <div class="detail-head">
                        <img src="@/assets/Vector/Green/icon_outline_checkmark.svg" alt="" aria-hidden="true" class="detail-icon">
                        <h2>Target Date</h2>
                    </div>
                    <p>{{ targetDateFormatted }}</p>
                </article>
            </div>

            <p class="helper-text">Progress uses working days only (weekends and 20 break days excluded).</p>
        </section>
    </main>

    <audio ref="sfxToggleOn" preload="auto" :src="sndClickB"></audio>
    <audio ref="sfxToggleOff" preload="auto" :src="sndClickA"></audio>
    <audio ref="sfxMilestone" preload="auto" :src="sndTapB"></audio>
    <audio ref="sfxComplete" preload="auto" :src="sndSwitchB"></audio>
</template>


<style scoped>
.app-shell {
    min-height: 100vh;
    display: grid;
    place-items: center;
    padding: 24px;
}

#progress-container {
    position: relative;
    z-index: 1;
    width: min(760px, 95vw);
    padding: 28px;
    border-radius: 22px;
    background: rgba(247, 252, 243, 0.85);
    border: 1px solid rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(9px);
    box-shadow: 0 16px 38px rgba(20, 64, 43, 0.2);
}

.top-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    margin-bottom: 10px;
}

.badge {
    display: inline-flex;
    align-items: center;
    padding: 6px 12px;
    border-radius: 999px;
    font-size: 0.8rem;
    letter-spacing: 0.04em;
    text-transform: uppercase;
    color: #154e39;
    background: rgba(168, 234, 106, 0.35);
    border: 1px solid rgba(73, 136, 91, 0.25);
}

.sound-btn {
    border: 1px solid rgba(47, 105, 75, 0.28);
    background: #ffffff;
    color: var(--text-soft);
    border-radius: 999px;
    padding: 8px 14px;
    font-family: var(--font-body);
    font-size: 0.86rem;
    cursor: pointer;
    transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.sound-btn:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(20, 64, 43, 0.15);
}

.sound-btn[aria-pressed="true"] {
    background: linear-gradient(90deg, var(--agtech-green-700), var(--agtech-green-500));
    color: #fff;
    border-color: transparent;
}

h1 {
    margin: 0;
    font-size: clamp(1.7rem, 4vw, 2.6rem);
    line-height: 1.15;
    text-align: center;
    font-family: var(--font-display);
    letter-spacing: 0.02em;
}

.subtitle {
    margin: 8px 0 18px;
    color: var(--text-soft);
    font-size: 1rem;
    text-align: center;
}

#progress-bar {
    width: 100%;
    height: 34px;
    background: rgba(21, 92, 60, 0.14);
    border-radius: 999px;
    overflow: hidden;
    border: 1px solid rgba(255, 255, 255, 0.95);
}

#progress-fill {
    position: relative;
    height: 100%;
    width: 0%;
    border-radius: inherit;
    overflow: hidden;
    background: linear-gradient(100deg, var(--agtech-green-700), var(--agtech-green-500), var(--agtech-lime-300));
    background-size: 200% 100%;
    transition: width 0.6s ease;
    animation: growthPulse 4.2s ease-in-out infinite;
}

#progress-fill::after {
    content: "";
    position: absolute;
    inset: 0;
    background: linear-gradient(90deg, transparent 0%, rgba(255, 255, 255, 0.38) 45%, transparent 100%);
    transform: translateX(-130%);
    animation: signalSweep 2.9s linear infinite;
}

.progress-meta {
    margin-top: 14px;
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    gap: 16px;
    flex-wrap: wrap;
}

#progress-text {
    margin: 0;
    font-size: 1.65rem;
    font-weight: 700;
}

#status-text {
    margin: 0;
    color: var(--text-soft);
    font-weight: 600;
}

.details-grid {
    margin-top: 20px;
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 12px;
}

.detail-card {
    background: rgba(240, 250, 234, 0.85);
    border: 1px solid rgba(214, 236, 201, 0.95);
    border-radius: 14px;
    padding: 14px;
}

.detail-card h2 {
    margin: 0;
    font-size: 0.86rem;
    font-weight: 700;
    letter-spacing: 0.02em;
    text-transform: uppercase;
    color: var(--text-soft);
}

.detail-head {
    display: flex;
    align-items: center;
    gap: 8px;
}

.detail-icon {
    width: 18px;
    height: 18px;
    opacity: 0.9;
}

.detail-card p {
    margin: 8px 0 0;
    font-size: 1rem;
    font-weight: 700;
}

.helper-text {
    margin: 14px 0 0;
    color: var(--text-soft);
    font-size: 0.85rem;
    text-align: center;
}

@keyframes growthPulse {
    0%,
    100% {
        background-position: 0% 50%;
    }
    50% {
        background-position: 100% 50%;
    }
}

@keyframes signalSweep {
    0% {
        transform: translateX(-130%);
    }
    100% {
        transform: translateX(130%);
    }
}

@media (prefers-reduced-motion: reduce) {
    #progress-fill,
    #progress-fill::after {
        animation: none;
    }
}

@media (max-width: 680px) {
    #progress-container {
        padding: 22px;
    }

    .details-grid {
        grid-template-columns: 1fr;
    }

    #progress-text {
        font-size: 1.4rem;
    }

    .top-row {
        justify-content: center;
    }
}
</style>