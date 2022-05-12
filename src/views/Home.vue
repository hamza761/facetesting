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
// @ is an alias to /src
import makeMediaStreamCompatible from '../helpers/CompatibleMediaStream'
import * as faceapi from 'face-api.js'
let clearInterval = null;
export default {
  name: 'Home',
  data () {
    return {
      loading: false,
    }
  },
  methods: {
    ellipseDistanceFromPoint(x, y, cx, cy, theta, alpha, beta) {
      const cosTheta = Math.cos(theta)
      const sinTheta = Math.sin(theta)
      const u = cosTheta*(x-cx) + sinTheta*(y-cy)
      const v = -sinTheta*(x-cx) + cosTheta*(y-cy)
      return Math.sqrt(((u/alpha)**2 + (v/beta)**2))
    },
    startWorking () {
    this.loading = true
    const canvas = document.getElementById('canvas')
    const v1 = document.getElementById('video')
    const ctx = canvas.getContext('2d')
    // ctx.beginPath();
    ctx.ellipse(canvas.width/2, canvas.height/2, 90, 130, Math.PI , 0 , 2 * Math.PI);
    ctx.fill();
    ctx.stroke()
    ctx.globalCompositeOperation = 'source-out';
    ctx.fillStyle = "blue";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    // ctx.scale(5,5)
    // ctx.closePath()
    ctx.globalCompositeOperation = 'source-over';
    ctx.moveTo(0, 10)
    ctx.lineTo(100, 10)
    ctx.stroke()
    makeMediaStreamCompatible()
    navigator.mediaDevices.getUserMedia( { audio: false, video: { width: 720, height: 480 } }).then(stream => {
        if ("srcObject" in v1) {
    v1.srcObject = stream;
  } else {
    // Avoid using this in new browsers, as it is going away.
    v1.src = window.URL.createObjectURL(stream);
  }
    }).catch(error => {
        console.log(error)
    })
    v1.onloadedmetadata = async (e) => {
    // await faceapi.nets.ssdMobilenetv1.loadFromUri('/models')
    await faceapi.nets.tinyFaceDetector.loadFromUri('/models')
    await faceapi.nets.faceLandmark68Net.loadFromUri('/models')
    // await faceapi.nets.faceRecognitionNet.loadFromUri('/models')
    this.loading = false
    v1.play()
    clearInterval = setInterval(async () => {
        const result = await faceapi.detectSingleFace(v1, new faceapi.TinyFaceDetectorOptions()).withFaceLandmarks()
        if (result && result.landmarks) {
          const jawPoints = result.landmarks.getJawOutline()
          let allPointsSatisfied = true
          const widthToBeMultiplyBy = v1.width/720
          const heightToBeMultiplyBy = v1.height/480
          for (let index = 0; index < jawPoints.length; index++) {
            const element = jawPoints[index]
            // console.log(jawPoints[index].x*widthToBeMultiplyBy, jawPoints[index].y*heightToBeMultiplyBy)
            const rslt = this.ellipseDistanceFromPoint(jawPoints[index].x*widthToBeMultiplyBy, jawPoints[index].y*heightToBeMultiplyBy, canvas.width/2, canvas.height/2, Math.PI, 130, 90)
            console.log(rslt)
            if(rslt > 1){
              allPointsSatisfied = false
              break
            }
          }
          console.log(allPointsSatisfied)
        } else console.log('frame your face')
    }, 1000)
            // if (d && jawPoints && jawPoints[5] && jawPoints[6] && jawPoints[12] && jawPoints[13]) {
            //     const box = d.detection.box
            //     console.log(ctx.isPointInPath(jawPoints[5].x, jawPoints[5].y), ctx.isPointInPath(jawPoints[6].x, jawPoints[6].y), ctx.isPointInPath(jawPoints[12].x, jawPoints[12].y), ctx.isPointInPath(jawPoints[13].x, jawPoints[13].y), ctx.isPointInPath(jawPoints[0].x, jawPoints[0].y), ctx.isPointInPath(jawPoints[16].x, jawPoints[16].y))
            // }
  }
    }
  },
  mounted() {
    this.startWorking()
  },
}
</script>
<style scoped>
        #canvas-and-video-holding-div {
            position: relative;
        }
        .canvas {
            /* background-color:white; */
            position: absolute;
        }
        .main {
            display: flex;
            justify-content: center;
        }
</style>