<script setup lang="ts">
import { ref, nextTick, onMounted } from "vue";
import { NButton, NUpload, NButtonGroup, useMessage } from "naive-ui";
import { Play, Pause, Download } from "@vicons/ionicons5";
import WaveSurfer from "wavesurfer.js";
import Timeline from "wavesurfer.js/dist/plugin/wavesurfer.timeline";
import sampleFile from "../assets/silent_10s.wav";

const MS_IN_PERIOD = 250;
const BYTE_IN_PERIOD = 256 * 1024;

const supportFileType = ["audio/wav", "audio/x-wav"];

const message = useMessage();

const fileRef = ref(null);
const fileList = ref([]);
const audioRef = ref(null);
const audioSource = ref(null);
const waveformRef = ref(null);
const waveformTimeRef = ref(null);
const transcript = ref("");
const downloadable = ref(false);

let websocket;
function initWebsocket(file) {
  websocket = new WebSocket(
    "ws://asr-8k-01.iot.iptnet.net:8080/client/ws/speech"
  );
  websocket.binaryType = "arraybuffer";

  websocket.onopen = () => {
    console.log("websocket is opened");
    handleFile(file);
  };

  websocket.onmessage = (event) => {
    const data = JSON.parse(event.data);

    if (data.adaptation_state) {
      downloadable.value = true;
    }

    if (data.result) {
      transcript.value = data.result.hypotheses[0].transcript;
    }
  };

  websocket.onclose = () => {
    console.log("websocket is successfully closed");
  };

  websocket.onerror = (error) => {
    message.error(`websocket has an error: ${error.message}`);
  };
}

function handleFile(file) {
  const fileReader = new FileReader();

  fileReader.onload = (event) => {
    const rawData = event.target.result;

    const blockCount = Math.ceil(rawData.byteLength / BYTE_IN_PERIOD);

    let currentCount = 0;
    const interval = setInterval(() => {
      const start = currentCount * BYTE_IN_PERIOD;
      const end = (currentCount + 1) * BYTE_IN_PERIOD;

      if (currentCount >= blockCount) {
        clearInterval(interval);
      }

      websocket.send(rawData.slice(start, end));

      currentCount++;
    }, MS_IN_PERIOD);
  };

  fileReader.readAsArrayBuffer(file);
}

function beforeUploadFile(data) {
  if (!supportFileType.includes(data.file.file.type)) {
    message.error("不支持的檔案類型,請上傳wav檔");
    return false;
  }
}

function initWaveSurfer() {
  wavesurfer = WaveSurfer.create({
    container: waveformRef.value,
    waveColor: "#409eff",
    mediaControls: false,
    backend: "MediaElement",
    audioRate: "1",
    plugins: [
      Timeline.create({
        container: waveformTimeRef.value,
      }),
    ],
  });

  wavesurfer.load(sampleFile);
}

onMounted(() => {
  initWaveSurfer();
});

let wavesurfer = null;
async function uploadFile(data) {
  audioSource.value = URL.createObjectURL(data.file.file);

  await nextTick();

  wavesurfer.load(audioSource.value);

  initWebsocket(data.file.file);
}

function handleFileListChange(data) {
  fileList.value = [
    {
      id: data[0].id,
      name: data[0].name,
    },
  ];
}

function handleRemove() {
  fileList.value = [];
}

function playAudio() {
  wavesurfer.play();
}

function stopAudio() {
  wavesurfer.pause();
}

function reset() {
  wavesurfer.pause();
  wavesurfer.seekTo(0);
  wavesurfer.load(sampleFile);

  transcript.value = "";
  fileList.value = [];
  downloadable.value = false;

  if (websocket) {
    websocket.close();
  }
}

function makeTextFile() {
  const data = new Blob([transcript.value], {
    type: "text/plain;charset=utf-8",
  });
  return window.URL.createObjectURL(data);
}

function handleDownload() {
  const link = document.createElement("a");
  link.setAttribute("download", "result.txt");
  link.href = makeTextFile();
  document.body.appendChild(link);

  window.requestAnimationFrame(() => {
    const evt = new MouseEvent("click");
    link.dispatchEvent(evt);
    document.body.removeChild(link);
  });
}
</script>

<template>
  <div class="wrapper">
    <div flex justify-center items-center min-h-100vh>
      <div w-800px class="content" p-12px min-h-600px>
        <div>
          <audio hidden ref="audioRef">
            <source :src="audioSource" />
          </audio>

          <div ref="waveformRef"></div>
          <div ref="waveformTimeRef" mb-24px></div>

          <NButtonGroup w="1/3" mb-8px>
            <NButton @click="playAudio" type="primary" :block="true">
              <template #icon>
                <Play />
              </template>
              播放
            </NButton>
            <NButton @click="stopAudio" type="primary" :block="true">
              <template #icon>
                <Pause />
              </template>
              停止
            </NButton>
            <NButton
              @click="handleDownload"
              type="primary"
              :disabled="!downloadable"
              :block="true"
            >
              <template #icon>
                <Download />
              </template>
              下載</NButton
            >
          </NButtonGroup>

          <NButton @click="reset" type="primary" :block="true" mb-8px
            >重置</NButton
          >

          <NUpload
            ref="fileRef"
            v-model:file-list="fileList"
            :default-upload="false"
            @before-upload="beforeUploadFile"
            @change="uploadFile"
            @update:file-list="handleFileListChange"
            :block="true"
            :max="1"
          >
            <NButton :block="true" type="primary">請上傳檔案</NButton>
          </NUpload>

          <h1 mt-24px>字詞辨識結果:</h1>
          <div my-12px>{{ transcript }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
:deep(.n-upload-trigger) {
  width: 100%;
}

.wrapper {
  position: relative;
}

.wrapper::before {
  content: "";
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  background: url("../assets/bg.jpg") no-repeat center center fixed;
  background-size: cover;
  opacity: 0.4;
  z-index: -1;
}

.content {
  position: relative;
}

.content::before {
  position: absolute;
  top: 0;
  left: 0;
  content: "";
  width: 100%;
  height: 100%;
  background-color: #ffffff;
  z-index: -1;
  opacity: 0.6;
  border: 1px solid #131313;
}
</style>
