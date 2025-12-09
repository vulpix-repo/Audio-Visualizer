<template>
    <div class="container">
        <video ref="videoRef" autoplay playsinline id="webcam"></video>
        <canvas ref="canvasRef" id="canvas"></canvas>
        
        <!-- pass hand data + audio to Sphere component -->
        <Sphere 
        :rightHandData="rightHandData" 
        :leftHandData="leftHandData"
        :audioURL="audioURL"
        />
        <Audio @audioSelected="handleAudioSelected" />

        <div ref="statusRef" id="status">Loading MediaPipe...</div>
    </div>
</template>

<script setup>
// workflow:
// HandTracking is the parent component -> webcam + mediapipe
// passes files from Audio.vue to Sphere.vue
// passes hand landmark data to Sphere.vue

import { onMounted, onBeforeUnmount, ref, reactive } from 'vue'
import Sphere from './Sphere.vue'
import Audio from './Audio.vue'

const videoRef = ref(null)
const canvasRef = ref(null)
const statusRef = ref(null)
const audioURL = ref(null)

let canvasCtx = null

// MediaPipe variables
let hands = null
let cameraHelper = null

// hand data passed to Sphere component
const rightHandData = reactive({
  active: false,
  thumbTip: null,
  indexTip: null,
  pinchDistance: 0
})

const leftHandData = reactive({
  active: false,
  indexTip: null
})

// audio handling
function handleAudioSelected(url) {
  audioURL.value = url
}

// helper function to update canvas size
function updateCanvasSize() {
    if (canvasRef.value) {
        canvasRef.value.width = window.innerWidth
        canvasRef.value.height = window.innerHeight
    }
}

// Handle window resize
function handleResize() {
    updateCanvasSize()
}

// Initialize webcam
async function initWebcam() {
    try {
        const stream = await navigator.mediaDevices.getUserMedia({
        video: {
            width: { ideal: 1920 },
            height: { ideal: 1080 },
            facingMode: 'user'
        }
})
    
    videoRef.value.srcObject = stream
    
    return new Promise((resolve) => {
        videoRef.value.onloadedmetadata = () => {
            updateCanvasSize()
            resolve(videoRef.value)
        }
    })
    } catch (error) {
        statusRef.value.textContent = `Error accessing webcam: ${error.message}`
        console.error('Error accessing webcam:', error)
        throw error
    }
}

// initialize MediaPipe Hands
async function initMediaPipeHands() {
    statusRef.value.textContent = 'Initializing MediaPipe Hands...'
    
    hands = new window.Hands({
        locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
        }
    })
    
    hands.setOptions({
        maxNumHands: 2,
        modelComplexity: 1,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
    })
    
    await hands.initialize()
    statusRef.value.textContent = 'Hand tracking ready!'
    
    return hands
}

// calculate distance between two 3D points
function calculateDistance(point1, point2) {
    const dx = point1.x - point2.x
    const dy = point1.y - point2.y
    const dz = point1.z - point2.z
    return Math.sqrt(dx * dx + dy * dy + dz * dz)
}

// draw hand landmarks
function drawLandmarks(landmarks, isLeft) {
    const screenSize = Math.min(window.innerWidth, window.innerHeight)
    const lineWidth = Math.max(2, Math.min(5, screenSize / 300))
    const pointSize = Math.max(2, Math.min(8, screenSize / 250))
    
    const connections = [
        [0, 1], [1, 2], [2, 3], [3, 4],
        [0, 5], [5, 6], [6, 7], [7, 8],
        [0, 9], [9, 10], [10, 11], [11, 12],
        [0, 13], [13, 14], [14, 15], [15, 16],
        [0, 17], [17, 18], [18, 19], [19, 20],
        [5, 9], [9, 13], [13, 17]
    ]
    
    const handColor = isLeft ? '#00FF00' : '#00FFFF'
    
    canvasCtx.lineWidth = lineWidth
    canvasCtx.strokeStyle = handColor
    
    connections.forEach(([i, j]) => {
        const start = landmarks[i]
        const end = landmarks[j]
        
        canvasCtx.beginPath()
        canvasCtx.moveTo(start.x * canvasRef.value.width, start.y * canvasRef.value.height)
        canvasCtx.lineTo(end.x * canvasRef.value.width, end.y * canvasRef.value.height)
        canvasCtx.stroke()
    })
    
    landmarks.forEach((landmark, index) => {
        let pointColor = handColor
        if (index === 4 || index === 8) {
        pointColor = '#FF0000'
        }
        
        canvasCtx.fillStyle = pointColor
        canvasCtx.beginPath()
        canvasCtx.arc(
        landmark.x * canvasRef.value.width,
        landmark.y * canvasRef.value.height,
        pointSize * 1.2,
        0,
        2 * Math.PI
        )
        canvasCtx.fill()
    })
}

// process video frames
function onResults(results) {
    canvasCtx.clearRect(0, 0, canvasRef.value.width, canvasRef.value.height)
    
    if (canvasRef.value.width !== window.innerWidth ||
        canvasRef.value.height !== window.innerHeight) {
        updateCanvasSize()
    }
    
    // Reset hand data
    rightHandData.active = false
    leftHandData.active = false
    
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        statusRef.value.textContent =
        results.multiHandLandmarks.length === 1 ? '1 hand detected' :
        `${results.multiHandLandmarks.length} hands detected`
        
        for (let handIndex = 0; handIndex < results.multiHandLandmarks.length; handIndex++) {
        const landmarks = results.multiHandLandmarks[handIndex]
        const handedness = results.multiHandedness[handIndex].label
        const isLeftHand = handedness === 'Left'
        
        drawLandmarks(landmarks, isLeftHand)
        
        if (!isLeftHand) { // because of mirrored video, right is actually left
            // LEFT HAND: Send pinch data to Sphere
            const thumbTip = landmarks[4]
            const indexTip = landmarks[8]
            const pinchDistance = calculateDistance(thumbTip, indexTip)
            
            leftHandData.active = true
            leftHandData.thumbTip = thumbTip
            leftHandData.indexTip = indexTip
            leftHandData.pinchDistance = pinchDistance
        } else {
            // RIGHT HAND: Send index finger position to Sphere
            const indexTip = landmarks[8]
            
            rightHandData.active = true
            rightHandData.indexTip = indexTip
        }
        }
    } else {
        statusRef.value.textContent = 'No hands detected'
    }
}

// start the application
async function startApp() {
    try {
        await initWebcam()
        const handsInstance = await initMediaPipeHands()
        
        handsInstance.onResults(onResults)
        
        cameraHelper = new window.Camera(videoRef.value, {
        onFrame: async () => {
            await handsInstance.send({ image: videoRef.value })
        },
        width: 1920,
        height: 1080
        })
        
        cameraHelper.start()
    } catch (error) {
        statusRef.value.textContent = `Error: ${error.message}`
        console.error('Error starting application:', error)
    }
}

onMounted(() => {
    canvasCtx = canvasRef.value.getContext('2d')
    updateCanvasSize()
    window.addEventListener('resize', handleResize)
    startApp()
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', handleResize)
    
    if (cameraHelper && cameraHelper.stop) {
        cameraHelper.stop()
    }
    
    if (videoRef.value && videoRef.value.srcObject) {
        const tracks = videoRef.value.srcObject.getTracks()
        tracks.forEach(track => track.stop())
    }
})
</script>

<style scoped>
.container {
    position: fixed;
    inset: 0;
    overflow: hidden;
}

#webcam {
    position: absolute;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transform: scaleX(-1);
}

#canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    transform: scaleX(-1);
    pointer-events: none;
}

#status {
    position: absolute;
    top: 20px;
    left: 20px;
    color: #fff;
    background-color: rgba(0, 0, 0, 0.5);
    padding: 10px;
    border-radius: 5px;
    font-family: Helvetica, sans-serif;
    z-index: 10;
}
</style>