<script setup lang="ts">
import * as RNBO from "@rnbo/js";
import {ref} from "vue";
import {Parameter} from "@rnbo/js";

async function loadPatch(context: AudioContext, url: string) {
  const response = await fetch(url);
  const patcher = await response.json();

  return await RNBO.createDevice({context, patcher});
}

const initialized = ref(false);

// ビデオ処理用途
const canvasRef = ref<HTMLCanvasElement>();
let canvasContext: CanvasRenderingContext2D | null = null;
const videoRef = ref<HTMLVideoElement>();
const brightness = ref("0");
const whitePercentage = ref(0);

async function setup() {
  const generatePulseURL = "rnbo/generate-pulse.export.json";
  const analyzePulseURL = "rnbo/analyze-pulse.export.json";


  // Create AudioContext
  // @ts
  (window as any).AudioContext = (window as any).AudioContext || (window as any).webkitAudioContext;
  const context = new AudioContext();

  // Create gain node and connect it to audio output
  const outputNode = context.createGain();
  outputNode.connect(context.destination);


  // create device
  const [generateDevice, analyzeDevice] = await Promise.all([loadPatch(context, generatePulseURL), loadPatch(context, analyzePulseURL)]);

  generateDevice.messageEvent.subscribe(ev => {
    if (ev.tag === "play") {
      document.body.style.backgroundColor = "white";


      setTimeout(() => {
        document.body.style.backgroundColor = "black";
      }, 60);
    }
  })

  generateDevice.node.connect(outputNode);

  console.log(analyzeDevice.outports);

  // const generatorTheta = generateDevice.parametersById.get("theta");
  //
  // const analyzeTheta = analyzeDevice.parametersById.get("theta");
  // analyzeTheta.changeEvent.subscribe((value: number) => {
  //   console.log(value);
  //   generatorTheta.value = value;
  // })

  const generateFreq = generateDevice.parametersById.get("freq");
  generateFreq.value = 7300;
  console.log(generateFreq);
  const analyzeFreq = analyzeDevice.parametersById.get("center_freq");
  analyzeFreq.value = 7300;

  // const analyzeSpeed = analyzeDevice.parametersById.get("speed");
  // analyzeSpeed.value = 0.05;

  let beforeDiff = 48000;
  const diffParams: Parameter[] = [...generateDevice.parameters.filter(p => p.id === "diff"), ...analyzeDevice.parameters.filter(p => p.id === "diff")];

  analyzeDevice.messageEvent.subscribe(ev => {
    if (ev.tag === "pulse") {
      console.log("pulse");
    } else if (ev.tag === "signal-list") {
      console.log("list:", ev.payload);
    } else if (ev.tag === "average") {
      console.log("avg: ", ev.payload);
      const average = ev.payload as number;

      if (average === 0 ) return;

      const diff = average / 1840 * 96000;
      const calcDiff = beforeDiff - (beforeDiff - diff) * 0.037;

      diffParams.forEach((p: Parameter) => p.value = calcDiff);

      beforeDiff = calcDiff;
      console.log(`diff: ${beforeDiff}`);
    }
  });

  // mic
  const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
      echoCancellation: false,
      noiseSuppression: true,
    }
  });
  const input = context.createMediaStreamSource(stream);
  input.connect(analyzeDevice.node);


  // (Optional) Extract the name and rnbo version of the patcher from the description
  // (Optional) Automatically create sliders for the device parameters
  document.body.onclick = () => {
    context.resume();
  }

  context.createOscillator().start(0);


  if (!canvasRef.value) {
    console.error("initialized error: canvas ref not found");
    return;
  }

  canvasContext = canvasRef.value?.getContext("2d", {willReadFrequently: true});

  if (!canvasContext) {
    console.error("initialized error: failed to initialize canvas context");
    return;
  }

  // function startBrightnessCalculation() {
  //   setInterval(calculateAverageBrightness, 16);
  // }
  //
  // // カメラ起動と明度計算の開始
  // startCamera().then(startBrightnessCalculation);


  // complete initialized
  initialized.value = true;

  document.body.addEventListener("click", () => {
    document.body.requestFullscreen({navigationUI: "hide"});
  });
}

</script>

<template>
  <video id="video" autoplay playsinline ref="videoRef" width="120" height="90" style="display: none"></video>
  <canvas v-show="initialized" ref="canvasRef"  style="display: none"></canvas>
  <button v-if="!initialized" @click="setup" ref="setup-button">setup</button>
  <h1 style="display: none">brightness: {{ brightness }}, white: {{ whitePercentage.toFixed(2) }}</h1>
</template>

<style scoped>
</style>
