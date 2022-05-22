<template>
<div>
    <div class="main">
        <video width="576px" height="400px" id="video"></video>
        <canvas width="576px" height="400px" id="canvas"></canvas>
    </div>
    <video width="200px" height="200px" controls id="recordedvideo"></video>
    <button @click="delChunks">
        del Chunks
    </button>
    <button @click="stop">
        stop
    </button>
</div>  
</template>

<script>
var chunks = []
var mediaRecorder = null
export default {
    methods: {
        delChunks() {
            chunks = []
        },
    drawEllipsea(ellipseRadiusX, ellipseRadiusY, shouldClearCanvas, borderColor) {
        const canvas = this.$refs.canvas;
      if (shouldClearCanvas)
        canvasContext.clearRect(0, 0, canvas.width, canvas.height);
      canvasContext.beginPath();
      canvasContext.fillStyle = "rgba(0, 0, 0, 0.42)";
      canvasContext.fillRect(0, 0, canvas.width, canvas.height);
      canvasContext.globalCompositeOperation = "destination-out";
      canvasContext.ellipse(
        canvas.width / 2,
        canvas.height / 2,
        ellipseRadiusX,
        ellipseRadiusY,
        Math.PI,
        0,
        2 * Math.PI
      );
      canvasContext.fill();
      canvasContext.globalCompositeOperation = "source-over";
      canvasContext.strokeStyle = borderColor;
      canvasContext.stroke();
},
    // used to draw ellipse on canvas

    drawCanvasaContent(rX, rY, shouldClearCanvas, borderColor, shouldAnimate) {
        if (shouldAnimate) {
            const drawEllipseWithAnimation = (intervalX, intervalY ,stopAnimationFrameId, checkGreater) => {
                let stopId = stopAnimationFrameId
                const ellipseAnimation = () => {
                    radiusX += intervalX
                    radiusY += intervalY
                    const conditionTobeCompared = checkGreater ? radiusX >= rX : radiusX <= rX
                    this.drawEllipse(radiusX, radiusY, shouldClearCanvas, borderColor)
                    if (conditionTobeCompared) {
                        console.log('cancelled')
                        cancelAnimationFrame(stopId)
                        return  
                    }
                    stopId = requestAnimationFrame(ellipseAnimation)
                }
                return ellipseAnimation
            }
            const intervalX = (rX - radiusX)/30
            const intervalY = (rY - radiusY)/30
            const AnimationFrameId = requestAnimationFrame(drawEllipseWithAnimation(intervalX, intervalY, AnimationFrameId, intervalX > 0))
        } else {
            this.drawEllipse(rX, rY, shouldClearCanvas, borderColor)
        }
    },
        findResolutionFromStream (stream) {
            return new Promise ((resolve, reject) => {
                if (stream.getVideoTracks) return resolve({data:stream.getVideoTracks()[0].getSettings(), supported: true})
                const tempVideoElement = document.createElement('video')
                if ('srcObject' in tempVideoElement) tempVideoElement.srcObject = stream
                else tempVideoElement.src = URL.createObjectURL(stream)
                tempVideoElement.onloadedmetadata = function () {
                    resolve({ data:{height: this.videoHeight, width: this.videoWidth}, supported: false})
                }
            })
        },
        stop () {
            mediaRecorder.stop()
        }
},
// {height:7680, width: 4320}
// aspectRatio: {
//             exact: width / height,
//           },
mounted() {
    const video = document.getElementById('video')
    navigator.mediaDevices.getUserMedia({
        audio: false, video: {width: {min: 720}, height: {min:480}}
    }).then(async res => {
        const r = await this.findResolutionFromStream(res)
        console.log(r)
        mediaRecorder = new MediaRecorder(res)
        mediaRecorder.start()
        mediaRecorder.onstop = () => {
            const blob = new Blob(chunks, {
                type: 'video/webm'
            })
            document.getElementById('recordedvideo').src = URL.createObjectURL(blob)
        }
        mediaRecorder.ondataavailable = (e) => {
            console.log(chunks)
            chunks.push(e.data)
        }
        video.srcObject = res
        video.onloadedmetadata = function() {
        console.log('resolution: ' + this.videoWidth + ' x ' + this.videoHeight);
    };
        video.play()
    }).catch(error => {
        console.log(error)
    })
},
}
</script>
  <!-- const mime = MediaRecorder.isTypeSupported("video/webm; codecs=vp9") 
             ? "video/webm; codecs=vp9" 
             : "video/webm" -->
<style scoped>
.main {
    position: relative;
    height: 200px;
    width:200px;
    background: red;
    border-radius: 32px;
    /* overflow: hidden; */
}
#video {
    top: 0px;
    position:absolute;
    /* transform: scaleX(-1); */
}
#canvas {
    top: 0px;
    position:absolute;
}
</style>