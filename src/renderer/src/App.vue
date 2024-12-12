<script setup lang="ts">
import { ref, onMounted, nextTick } from 'vue';
import APlayer from 'aplayer';
import DPlayer from 'dplayer';
import 'aplayer/dist/APlayer.min.css';


const mediaType = ref('audio');
const songInfo = ref({
  title: '',
  album: '',
  artist: '',
  lyricistComposer: '',
  creator: 'DoLyric',
  version: '1.2.0',
});

const currentTime = ref('00:00.00');
let player: APlayer | DPlayer | null = null;

const lyricsText = ref('');
const lyricsLines = ref<string[]>([]);
const currentLineIndex = ref(0);

const fileInputRef = ref<HTMLInputElement | null>(null);
const lyricFileInputRef = ref<HTMLInputElement | null>(null);

onMounted(() => {
  setInterval(() => {
    if (player) {
      const time = player.audio ? player.audio.currentTime : player.video.currentTime;
      currentTime.value = formatTime(time);
    }
  }, 100);
});

function formatTime(seconds: number): string {
  const minutes = Math.floor(seconds / 60);
  const remainingSeconds = seconds % 60;
  return `${String(minutes).padStart(2, '0')}:${String(Math.round(remainingSeconds * 100) / 100).padStart(5, '0')}`;
}

async function importLyrics(event: Event) {
  const target = event.target as HTMLInputElement;
  if (!target.files || target.files.length === 0) return;

  const file = target.files[0];
  const text = await file.text();
  lyricsText.value = text;
  lyricsLines.value = text.split('\n').filter(line => line.trim() !== '');
  currentLineIndex.value = 0;

  // Reset file input to allow re-selecting the same file
  target.value = '';
}

function addLrcTag() {
  if (currentLineIndex.value >= lyricsLines.value.length) return;

  const timeTag = `[${currentTime.value}] `;
  lyricsLines.value[currentLineIndex.value] = timeTag + lyricsLines.value[currentLineIndex.value].trim();
  lyricsText.value = lyricsLines.value.join('\n');

  currentLineIndex.value++;
}

function exportLrc() {
  const header = `\
[al:${songInfo.value.album}]\n\
[ar:${songInfo.value.artist}]\n\
[au:${songInfo.value.lyricistComposer}]\n\
[by:${songInfo.value.creator}]\n\
[re:${songInfo.value.creator}]\n\
[ti:${songInfo.value.title}]\n\
[ve:${songInfo.value.version}]\n`;

  const blob = new Blob([header + lyricsText.value], { type: 'text/plain;charset=utf-8' });
  const link = document.createElement('a');
  link.href = URL.createObjectURL(blob);
  link.download = `${songInfo.value.title}.lrc`;
  link.click();
}

async function handleFileChange(event: Event) {
  const target = event.target as HTMLInputElement;
  if (!target.files || target.files.length === 0) return;

  const file = target.files[0];
  const objectUrl = URL.createObjectURL(file);

  destroyPlayer();

  await nextTick(); // Ensure the DOM is updated

  if (mediaType.value === 'audio') {
    player = new APlayer({
      container: document.getElementById('player-audio'),
      audio: {
        url: objectUrl,
      },
    });
  } else {
    player = new DPlayer({
      container: document.getElementById('player-video'),
      video: {
        url: objectUrl,
      },
    });
  }

  // Reset file input to allow re-selecting the same file
  target.value = '';
}

function handleMediaTypeChange() {
  destroyPlayer();
}

function destroyPlayer() {
  if (player) {
    player.destroy();
    player = null;
  }
}
</script>

<template>
  <v-app>
    <v-container fluid class="pa-0 ma-0 h-100">
      <v-row no-gutters style="height: 100%;">
        <!-- Left Panel -->
        <v-col cols="6" class="pr-2">
          <v-card>
            <v-card-text>
              <!-- Media Type Selector -->
              <v-radio-group v-model="mediaType" row @change="handleMediaTypeChange">
                <v-radio label="Audio" value="audio"></v-radio>
                <v-radio label="Video" value="video"></v-radio>
              </v-radio-group>
              <!-- Player -->
              <div v-if="mediaType === 'audio'" id="player-audio" class="player"></div>
              <div v-if="mediaType === 'video'" id="player-video" class="player"></div>
              <!-- Upload Button -->
              <v-btn @click="fileInputRef?.click()" class="upload-btn">Upload Media</v-btn>
              <input type="file" @change="handleFileChange" ref="fileInputRef" hidden />
            </v-card-text>
          </v-card>
          <!-- Song Info and Buttons -->
          <v-card class="mt-2">
            <v-card-text>
              <v-text-field v-model="songInfo.title" label="Title" required></v-text-field>
              <v-text-field v-model="songInfo.album" label="Album" required></v-text-field>
              <v-text-field v-model="songInfo.artist" label="Artist" required></v-text-field>
              <v-text-field v-model="songInfo.lyricistComposer" label="Lyricist/Composer" required></v-text-field>
              <v-btn @click="exportLrc" class="export-btn">Export LRC</v-btn>
              <v-btn @click="addLrcTag" class="insert-btn">Insert LRC Tag</v-btn>
            </v-card-text>
          </v-card>
        </v-col>
        <!-- Right Panel -->
        <v-col cols="6" class="pl-2">
          <!-- Current Time and Buttons -->
          <v-card>
            <v-card-text>
              <v-row align="center">
                <v-col cols="4">
                  <span>{{ currentTime }}</span>
                </v-col>
                <v-col cols="8">
                  <v-btn @click="lyricFileInputRef?.click()" class="import-btn">Import Lyrics</v-btn>
                  <input type="file" @change="importLyrics" accept=".lrc" hidden ref="lyricFileInputRef" />
                </v-col>
              </v-row>
            </v-card-text>
          </v-card>
          <!-- Lyrics Textarea -->
          <v-card class="mt-2">
            <v-card-text>
              <v-textarea v-model="lyricsText" label="Lyrics" filled auto-grow rows="10"></v-textarea>
            </v-card-text>
          </v-card>
        </v-col>
      </v-row>
    </v-container>
  </v-app>
</template>

<style scoped>
.player {
  width: 100%;
  height: 300px;
  margin-bottom: 10px;
}

.upload-btn, .export-btn, .insert-btn, .import-btn {
  margin-right: 5px;
}

textarea {
  font-family: monospace;
}
</style>
