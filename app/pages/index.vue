<script setup lang="ts">
import { AudioClassifier, FilesetResolver } from "@mediapipe/tasks-audio";
import type { TableColumn } from "@nuxt/ui";

type Category = {
  time: number;
  category: string;
  confidence: number;
};

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
    cell: ({ row }) => row.getValue("category"),
  },
  {
    accessorKey: "confidence",
    header: "Confidence",
    cell: ({ row }) =>
      `${((row.getValue("confidence") as number) * 100).toFixed(2)}%`,
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
    alert("Выберите аудиофайл");
    return;
  }
  if (!audioClassifier) {
    alert("Классификатор еще загружается. Попробуйте позже");
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
      category: item.classifications[0].categories[0].categoryName,
      confidence: item.classifications[0].categories[0].score.toFixed(2),
    };
  });
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
        />
      </UPageBody>
    </UPage>
  </UContainer>
</template>
