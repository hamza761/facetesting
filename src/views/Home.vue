<template>
  <div class="main">
    <div id="canvas-and-video-holding-div">
      <canvas class="canvas" width="500px" height="350px" id="canvas"></canvas>
      <video height="350px" width="500px" id="video"></video>
    </div>
  </div>
</template>

<script>
// https://stackoverflow.com/questions/11041547/finding-the-distance-of-a-point-to-an-ellipse-wether-its-inside-or-outside-of-e
// https://math.stackexchange.com/questions/3670465/calculate-distance-from-point-to-ellipse-edge
// @ is an alias to /src
import makeMediaStreamCompatible from "../helpers/CompatibleMediaStream";
import * as faceapi from "face-api.js";
import { Point } from 'face-api.js';
let clearInterval = null;
export default {
  name: "Home",
  data() {
    return {
      loading: false,
    };
  },
  methods: {
    ellipseDistanceFromPoint(x, y, cx, cy, theta, alpha, beta) {
      const cosTheta = Math.cos(theta);
      const sinTheta = Math.sin(theta);
      const u = cosTheta * (x - cx) + sinTheta * (y - cy);
      const v = -sinTheta * (x - cx) + cosTheta * (y - cy);
      return Math.sqrt((u / alpha) ** 2 + (v / beta) ** 2);
    },
    calculateCenterOfBox (x1,y1,x2,y2) {
      return {x:x1+x2/2, y:y1+y2/2}
    },
    calculateDistanceBetweenTwoPoints (x1, y1, x2, y2) {
      const x = (x2-x1)**2
      const y = (y2-y1)**2
      return Math.sqrt(x+y)
    },
    calculateFaceMatricsFromJawPoints (jawPoints) {
          let summary = {
              minDistance: +Infinity,
              maxDistance: -Infinity,
              doesAllPointsWithInTheOval: true,
              doesAllPointsRangeCloseToOval: true
            }
      const canvas = document.getElementById("canvas");
      const v1 = document.getElementById("video");
      const widthToBeMultiplyBy = v1.width / 720;
      const heightToBeMultiplyBy = v1.height / 480;
      jawPoints.forEach(point => {
        const calculatedDistance = this.ellipseDistanceFromPoint(
                  point.x * widthToBeMultiplyBy,
                  point.y * heightToBeMultiplyBy,
                  canvas.width / 2,
                  canvas.height / 2,
                  Math.PI,
                  90,
                  130
                );
        if(calculatedDistance < summary.minDistance) summary.minDistance = calculatedDistance
        if(calculatedDistance > summary.maxDistance) summary.maxDistance = calculatedDistance
        if(summary.doesAllPointsWithInTheOval && calculatedDistance > 1) summary.doesAllPointsWithInTheOval = false
        if(summary.doesAllPointsRangeCloseToOval && !(calculatedDistance > 1 && calculatedDistance < 1.50)) summary.doesAllPointsRangeCloseToOval = false
      });
      return summary
    },
    startWorking() {
      this.loading = true;
      const canvas = document.getElementById("canvas");
      const v1 = document.getElementById("video");
      const ctx = canvas.getContext("2d");
      // ctx.beginPath();
      ctx.ellipse(
        canvas.width / 2,
        canvas.height / 2,
        90,
        130,
        Math.PI,
        0,
        2 * Math.PI
      );
      ctx.fill();
      ctx.stroke();
      ctx.globalCompositeOperation = "source-out";
      ctx.fillStyle = "blue";
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      // ctx.scale(5,5)
      // ctx.closePath()
      ctx.globalCompositeOperation = "source-over";
      ctx.moveTo(0, 10);
      ctx.lineTo(100, 10);
      ctx.stroke();
      makeMediaStreamCompatible();
      navigator.mediaDevices
        .getUserMedia({ audio: false, video: { width: 720, height: 480 } })
        .then((stream) => {
          if ("srcObject" in v1) {
            v1.srcObject = stream;
          } else {
            // Avoid using this in new browsers, as it is going away.
            v1.src = window.URL.createObjectURL(stream);
          }
        })
        .catch((error) => {
          console.log(error);
        });
      v1.onloadedmetadata = async (e) => {
        // await faceapi.nets.ssdMobilenetv1.loadFromUri('/models')
        await faceapi.nets.tinyFaceDetector.loadFromUri("/models");
        await faceapi.nets.faceLandmark68Net.loadFromUri("/models");
        // await faceapi.nets.faceRecognitionNet.loadFromUri('/models')
        this.loading = false;
        v1.play();
        clearInterval = setInterval(async () => {
          const result = await faceapi
            .detectAllFaces(v1, new faceapi.TinyFaceDetectorOptions())
            .withFaceLandmarks();
            console.log(result)
            // let summary = {
            //   minDistance: +Infinity,
            //   maxDistance: -Infinity,
            //   doesAllPointsWithInTheOval: null,
            //   doesAllPointsOutsideTheOval: null,
            // }
            const minDistance = {
              value: +Infinity,
              index: null
            }
          if (result && result.length) {
            const widthToBeMultiplyBy = v1.width / 720;
            const heightToBeMultiplyBy = v1.height / 480;
            let allPointsSatisfied = true;
            for (let outer = 0; outer < result.length; outer++) {
              const singleFace = result[outer];
              const faceCenter = this.calculateCenterOfBox(singleFace.detection.box.topLeft.x, singleFace.detection.box.topLeft.y, singleFace.detection.box.bottomRight.x, singleFace.detection.box.bottomRight.y)
              const centerWithinEllipse = this.ellipseDistanceFromPoint(
                  faceCenter.x * widthToBeMultiplyBy,
                  faceCenter.y * heightToBeMultiplyBy,
                  canvas.width / 2,
                  canvas.height / 2,
                  Math.PI,
                  90,
                  130
                );
                const distanceFromCenterToCenter = this.calculateDistanceBetweenTwoPoints(faceCenter.x, faceCenter.y, canvas.width / 2, canvas.height / 2)
                if(minDistance.value > distanceFromCenterToCenter) {
                  minDistance.value = distanceFromCenterToCenter
                  minDistance.index = outer
                }
              const jawPoints = singleFace.landmarks.getJawOutline();
              for (let index = 0; index < jawPoints.length; index++) {
                // if (centerWithinEllipse > 1) {
                //   allPointsSatisfied = false;
                //   break
                // }
                const element = jawPoints[index];
                // console.log(jawPoints[index].x*widthToBeMultiplyBy, jawPoints[index].y*heightToBeMultiplyBy)
                const rslt = this.ellipseDistanceFromPoint(
                  jawPoints[index].x * widthToBeMultiplyBy,
                  jawPoints[index].y * heightToBeMultiplyBy,
                  canvas.width / 2,
                  canvas.height / 2,
                  Math.PI,
                  90,
                  130
                );
                console.log(rslt, index, "index");
                if (rslt > 1 || rslt < 0.80) {
                  allPointsSatisfied = false;
                  break;
                }
              }
              console.log(allPointsSatisfied);
              if (allPointsSatisfied) break;
              else {
                console.log('in else part')
                const minSingleFace = result[minDistance.index]
                console.log(minSingleFace.landmarks.getJawOutline())
                const summ = this.calculateFaceMatricsFromJawPoints(minSingleFace.landmarks.getJawOutline())
                console.log(summ)
              }
            }
          } else console.log("frame your face");
        }, 1000);
        // if (d && jawPoints && jawPoints[5] && jawPoints[6] && jawPoints[12] && jawPoints[13]) {
        //     const box = d.detection.box
        //     console.log(ctx.isPointInPath(jawPoints[5].x, jawPoints[5].y), ctx.isPointInPath(jawPoints[6].x, jawPoints[6].y), ctx.isPointInPath(jawPoints[12].x, jawPoints[12].y), ctx.isPointInPath(jawPoints[13].x, jawPoints[13].y), ctx.isPointInPath(jawPoints[0].x, jawPoints[0].y), ctx.isPointInPath(jawPoints[16].x, jawPoints[16].y))
        // }
      };
    },
  },
  mounted() {
    this.startWorking();
  },
};
</script>
<style scoped>
#canvas-and-video-holding-div {
  position: relative;
}
#canvas-and-video-holding-div video {
  object-fit: cover;
  transform: rotateY(180deg);
}
.canvas {
  /* background-color:white; */
  position: absolute;
  transform: rotateY(180deg);
  z-index: 1;
}
.main {
  display: flex;
  justify-content: center;
}
</style>
