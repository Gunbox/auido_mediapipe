<script setup lang="ts">
import { AudioClassifier, FilesetResolver } from "@mediapipe/tasks-audio";
import type { TableColumn, TabsItem } from "@nuxt/ui";

type Category = {
  time: number;
  category: string[];
  confidence: number[];
};

const tabs = ref<TabsItem[]>([
  {
    label: "Upload file",
    icon: "i-lucide-upload",
    slot: "file-upload",
  },
  {
    label: "Microphone",
    icon: "i-lucide-mic",
    slot: "microphone",
  },
]);

const pending = ref(true);
const resultsLoading = ref(false);
const resultsData = ref<Category[]>([]);
const resultsError = ref("");
const selectedFile = ref<File | null>(null);

const columns: TableColumn<Category>[] = [
  {
    accessorKey: "time",
    header: "Time",
    cell: ({ row }) => `${row.getValue("time")} ms`,
  },
  {
    accessorKey: "category",
    header: "Category",
    cell: ({ row }) => (row.getValue("category") as string[]).join("\n"),
  },
  {
    accessorKey: "confidence",
    header: "Confidence",
    cell: ({ row }) =>
      (row.getValue("confidence") as number[])
        .map((c) => `${(c * 100).toFixed(2)}%`)
        .join("\n"),
  },
];

let audioClassifier: AudioClassifier;
let audioCtx: AudioContext;

const createAudioClassifier = async () => {
  pending.value = true;

  const audio = await FilesetResolver.forAudioTasks(
    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-audio@0.10.0/wasm"
  );

  audioClassifier = await AudioClassifier.createFromOptions(audio, {
    baseOptions: {
      modelAssetPath:
        "https://storage.googleapis.com/mediapipe-models/audio_classifier/yamnet/float32/1/yamnet.tflite",
    },
  });

  pending.value = false;
};

async function runAudioClassification() {
  if (!selectedFile.value) {
    alert("Choose an audio file first");
    return;
  }
  if (!audioClassifier) {
    alert("Classifier is still loading. Please try again later.");
    return;
  }
  resultsLoading.value = true;
  if (!audioCtx) {
    audioCtx = new AudioContext();
  }
  const reader = new FileReader();
  reader.onload = async (e) => {
    const arrayBuffer = e.target?.result;
    if (!arrayBuffer || typeof arrayBuffer !== "object") return;
    const audioBuffer = await audioCtx.decodeAudioData(
      arrayBuffer as ArrayBuffer
    );
    const results = audioClassifier.classify(
      audioBuffer.getChannelData(0),
      audioBuffer.sampleRate
    );
    displayClassificationResults(results);
    resultsLoading.value = false;
  };
  reader.readAsArrayBuffer(selectedFile.value);
}

function displayClassificationResults(results: any) {
  console.log(results);
  if (!results.length) {
    resultsError.value = "Нет результатов";
    return;
  }

  resultsData.value = results.map((item: any) => {
    return {
      time: item.timestampMs,
      category: item.classifications[0].categories
        .slice(0, 5)
        .map((c: any) => c.categoryName),
      confidence: item.classifications[0].categories
        .slice(0, 5)
        .map((c: any) => c.score.toFixed(2)),
    };
  });
}

enum ProcessState {
  IDLE,
  RUNNING,
}

const ProcessTitles = {
  [ProcessState.IDLE]: "START",
  [ProcessState.RUNNING]: "STOP",
};

const microResult = ref<{ category: string; score: number }[]>([]);
const processState = ref<ProcessState>(ProcessState.IDLE);
const processTitle = computed(() => ProcessTitles[processState.value]);

async function runStreamingAudioClassification() {
  const constraints = { audio: true };
  let stream;

  try {
    stream = await navigator.mediaDevices.getUserMedia(constraints);
  } catch (err) {
    console.log("The following error occured: " + err);
    alert("getUserMedia not supported on your browser");
  }

  if (!audioCtx) {
    audioCtx = new AudioContext({ sampleRate: 16000 });
  } else if (audioCtx.state === "running") {
    await audioCtx.suspend();
    processState.value = ProcessState.IDLE;

    return;
  }

  // resumes AudioContext if has been suspended
  await audioCtx.resume();

  microResult.value = [];
  processState.value = ProcessState.RUNNING;

  if (!stream) {
    alert("Could not get audio stream.");
    return;
  }
  const source = audioCtx.createMediaStreamSource(stream);
  const scriptNode = audioCtx.createScriptProcessor(16384, 1, 1);

  scriptNode.onaudioprocess = function (audioProcessingEvent) {
    const inputBuffer = audioProcessingEvent.inputBuffer;
    let inputData = inputBuffer.getChannelData(0);

    // Classify the audio
    const result = audioClassifier.classify(inputData);
    const categories = result?.[0]?.classifications?.[0]?.categories ?? [];
    microResult.value.unshift(
      ...categories
        .filter((c: any) => c.score > 0.1)
        .map((c: any) => ({
          category: c.categoryName,
          score: +(c.score * 100).toFixed(2),
        }))
    );
  };

  source.connect(scriptNode);
  scriptNode.connect(audioCtx.destination);
}

watch(selectedFile, () => {
  resultsError.value = "";
  resultsData.value = [];
  runAudioClassification();
});

onMounted(() => {
  createAudioClassifier();
});
</script>

<template>
  <UContainer>
    <UPage>
      <UPageHeader
        title="Audio Classification"
        description="Demonstration of audio classification using MediaPipe"
      />
      <UPageBody>
        <UTabs :items="tabs">
          <template #file-upload>
            <UFileUpload
              position="inside"
              layout="list"
              label="Drop your audio files here"
              description="M4A, MP3, WAV (max. 2MB)"
              accept="audio/*, .m4a"
              v-model="selectedFile"
            />

            <div v-if="resultsError">
              <div>{{ resultsError }}</div>
            </div>
            <UTable
              v-else-if="selectedFile"
              :loading="resultsLoading"
              :data="resultsData"
              :columns="columns"
              :ui="{ td: 'whitespace-pre' }"
            />
          </template>

          <template #microphone>
            <UContainer
              class="flex flex-col gap-4 items-center justify-center py-10"
            >
              <div>
                <UButton
                  :icon="
                    processState === ProcessState.RUNNING
                      ? 'i-lucide-mic-off'
                      : 'i-lucide-mic'
                  "
                  :color="
                    processState === ProcessState.RUNNING ? 'error' : 'primary'
                  "
                  @click="runStreamingAudioClassification"
                  >{{ processTitle }}</UButton
                >
              </div>
              <div
                v-if="microResult.length"
                class="flex flex-col items-start w-full"
              >
                <div
                  v-for="(result, index) in microResult"
                  :key="`${result.category}-${index}`"
                >
                  {{ result.category }} ({{ result.score }}%)
                </div>
              </div>
            </UContainer>
          </template>
        </UTabs>
      </UPageBody>
    </UPage>
  </UContainer>
</template>
