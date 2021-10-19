<template>
  <q-page>
    <div
      class="row items-center justify-evenly q-py-lg"
      style="font-size: 20px; font-weight: bold"
    >
      <div class="col"></div>
      <div class="col-6 text-center">سر شما به سمت {{ faceDirection }} است</div>
      <div class="col"></div>
    </div>
    <div class="row items-center justify-evenly">
      <div class="col"></div>
      <div class="col">
        <canvas ref="cc" class="canvas" />
        <video
          ref="video"
          autoplay
          playsinline
          @playing="videoPlaying = true"
          @pause="videoPlaying = false"
          class="camera-gestures-camera-feed"
        ></video>
      </div>
      <div class="col"></div>
    </div>
  </q-page>
</template>

<script lang="ts">
import { defineComponent, ref } from "vue";
import * as ml5 from "ml5";
import fp from "../plugin/gestureFinder";
import * as faceLandmarksDetection from "@tensorflow-models/face-landmarks-detection";
import { Console } from "console";
require("@tensorflow/tfjs-backend-webgl");

export default defineComponent({
  name: "PageIndex",
  components: {},
  setup() {},
  data() {
    return {
      landmarkColors: {
        thumb: "red",
        indexFinger: "blue",
        middleFinger: "yellow",
        ringFinger: "green",
        pinky: "pink",
        palmBase: "white",
      },
      theref: this.$refs,
      config: {
        video: { width: 640, height: 480, fps: 30 },
      },
      gestureStrings: {
        thumbs_up: "👍",
        victory: "✌🏻",
      },
      faceDirection: "",
      gesture: "",
      knn: null,
      mobilenet: null,
      busyLoadingMobilenet: false,
      mediaStream: {},
      videoPlaying: false,
      // can be "training", "testing" or "predicting"
      state: "training",
      preparing: false,
      // -1 indicates nothing, -2 indicates neutral
      currentGestureIndex: -1,
      timeStartedWaiting: null,
      timeToFinishWaiting: null,
      progress: 0,
      gestureVerifyingCorrectSamples: 0,
      gestureVerifyingIncorrectSamples: 0,
      verifyingRetried: false,
      lastGestureIndexDetected: -1,
      lastGestureDetectedTime: null,

      model: null,
      canvs: Object,
    };
  },
  methods: {
    async ml5analyze() {
      setTimeout(async () => {
        let predictions = [];
        const facemesh = await ml5.facemesh(
          this.$refs.video,
          this.modelLoaded()
        );
        // configure gesture estimator
        // add "✌🏻" and "👍" as sample gestures
        const knownGestures = [
          fp.Gestures.VictoryGesture,
          fp.Gestures.ThumbsUpGesture,
        ];
        const GE = new fp.GestureEstimator(knownGestures);

        facemesh.on("predict", (results: any) => {
          let _this = this;
          predictions = results;
          if (results.length > 0) {
            this.canvs.clearRect(
              0,
              0,
              this.config.video.width,
              this.config.video.height
            );

            for (let i = 0; i < results.length; i++) {
              // draw colored dots at each predicted joint position
              // for (let part in results[i].annotations) {
              //   for (let point of results[i].annotations[part]) {
              //     _this.drawPoint(
              //       _this.canvs,
              //       point[0],
              //       point[1],
              //       1,
              //       // _this.landmarkColors[part]
              //       "blue"
              //     );
              //   }
              // }
              function dist(x1: any, x2: any, y1: any, y2: any) {
                let a = x2 - x1;
                let b = y2 - y1;
                return Math.sqrt(a * a - b * b);
              }

              let distleft = dist(
                predictions[i].annotations.noseTip[0][0],
                predictions[i].annotations.leftCheek[0][0],
                predictions[i].annotations.noseTip[0][1],
                predictions[i].annotations.leftCheek[0][1]
              );
              let distright = dist(
                predictions[i].annotations.noseTip[0][0],
                predictions[i].annotations.rightCheek[0][0],
                predictions[i].annotations.noseTip[0][1],
                predictions[i].annotations.rightCheek[0][1]
              );
              console.log(predictions[i].annotations);
              if (distleft > distright) {
                this.faceDirection = "چپ";
              } else {
                this.faceDirection = "راست";
              }
              // console.log(predictions[i].annotations);
              // // using a minimum confidence of 7.5 (out of 10)
              // const est = GE.estimate(predictions[i].landmarks, 7.5);

              // if (est.gestures.length > 0) {
              //   // find gesture with highest confidence
              //   let result = est.gestures.reduce((p, c) => {
              //     return p.confidence > c.confidence ? p : c;
              //   });
              //   console.log(result);
              //   this.gesture = this.gestureStrings[result.name];
              // }
            }
            this.clear();
            // this.draw(predictions[0].annotations.palmBase);
          }
        });
      }, 1);
    },
    drawPoint(ctx: any, x: any, y: any, r: any, color: any) {
      // ctx.clearRect(0, 0, 500, 375);

      ctx.beginPath();
      ctx.arc(x, y, r, 0, 2 * Math.PI);
      ctx.fillStyle = color;
      ctx.fill();
    },
    async modelLoaded() {
      console.log("model loaded");
    },
    draw(base: any) {
      // this.clear();
      this.canvs.fillStyle = "#0000ff";
      this.canvs.clearRect(
        0,
        0,
        this.config.video.width,
        this.config.video.height
      );

      this.canvs.width = this.config.video.width;
      this.canvs.height = this.config.video.height;
      this.canvs.beginPath();
      this.canvs.arc(base[0][0], base[0][1], 5, 0, Math.PI * 2, true);
      this.canvs.closePath();
      this.canvs.fill();
    },
    clear() {
      this.canvs.globalCompositeOperation = "destination-out";
      this.canvs.fillStyle = "rgba(0,0,0,0)";
      this.canvs.fillRect(
        0,
        0,
        this.config.video.width,
        this.config.video.height
      );
      this.canvs.globalCompositeOperation = "source-over";
    },
  },
  mounted: async function () {
    // this.knn = createKnnClassifier();
    // this.mobilenet = await loadMobilenet();
    this.busyLoadingMobilenet = false;
    this.mediaStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: false,
    });
    this.$refs.video.srcObject = this.mediaStream;
    this.$refs.video.play();

    this.canvs = this.$refs.cc.getContext("2d");
    this.canvs.canvas.width = this.config.video.width;
    this.canvs.canvas.height = this.config.video.height;
    // console.log(this.canvs);
    // this.animationFrameId = requestAnimationFrame(this.animate);
    // this.updateState();
    this.ml5analyze();
  },
});
</script>
<style scoped>
video.camera-gestures-camera-feed {
  /* transform: rotateY(180deg); */
  /* -webkit-transform: rotateY(180deg); Safari and Chrome */
  /* -moz-transform: rotateY(180deg); Firefox */
}
.canvas {
  position: absolute;
  z-index: 10;
}
</style>
