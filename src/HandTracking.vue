<template>
    <div class="container">
        <video ref="videoRef" autoplay playsinline id="webcam"></video>
        <canvas ref="canvasRef" id="canvas"></canvas>
        <div ref="threeRef" id="three-canvas"></div>
        <div ref="statusRef" id="status">Loading MediaPipe...</div>
    </div>
    <p id="links-para">
        - Left hand: Pinch thumb and index finger to resize the 3D sphere<br>
        - Right hand: Touch the sphere with your index finger to change its color
    </p>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'

const videoRef = ref(null)
const canvasRef = ref(null)
const threeRef = ref(null)
const statusRef = ref(null)

let canvasCtx = null

// three.js variables
let scene, camera, renderer
let sphere, solidMesh, wireframeMesh

// hand tracking variables
let rightHandActive = false
let leftHandActive = false
let lastColorChangeTime = 0
const colorChangeDelay = 500
let currentSphereSize = 1.0
let targetSphereSize = 1.0
const smoothingFactor = 0.15

// mediapipe helper variables
let hands = null
let cameraHelper = null

// helper function to update canvas size
function updateCanvasSize() {
    if (canvasRef.value) {
        canvasRef.value.width = window.innerWidth
        canvasRef.value.height = window.innerHeight
    }
}

// handle window resize
function handleResize() {
    updateCanvasSize()
    if (renderer) {
        renderer.setSize(window.innerWidth, window.innerHeight)
    }
    if (camera) {
        camera.aspect = window.innerWidth / window.innerHeight
        camera.updateProjectionMatrix()
    }
}

// initialize webcam
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

// generate random neon colors
function getRandomNeonColor() {
    const neonColors = [
    0xFF00FF, // Magenta
    0x00FFFF, // Cyan
    0xFF3300, // Neon Orange
    0x39FF14, // Neon Green
    0xFF0099, // Neon Pink
    0x00FF00, // Lime
    0xFF6600, // Neon Orange-Red
    0xFFFF00  // Yellow
    ]
    return neonColors[Math.floor(Math.random() * neonColors.length)]
}

// initialize Three.js
function initThreeJS() {
    scene = new THREE.Scene()
  
    // fov, aspect, near, far
    camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
        )
    camera.position.z = 5
  
    renderer = new THREE.WebGLRenderer({antialias: true, alpha: true })
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setClearColor(0x000000, 0) // transparent background
    threeRef.value.appendChild(renderer.domElement)
  
    const geometry = new THREE.SphereGeometry(2, 32, 32)
  
     sphere = new THREE.Group()
  
    const solidMaterial = new THREE.MeshBasicMaterial({
        color: 0xff00ff,
        transparent: true,
        opacity: 0.5
    })
    solidMesh = new THREE.Mesh(geometry, solidMaterial)
    sphere.add(solidMesh)
  
    const wireframeMaterial = new THREE.MeshBasicMaterial({
        color: 0xffffff,
        wireframe: true,
        transparent: false
    })
    wireframeMesh = new THREE.Mesh(geometry, wireframeMaterial)
    sphere.add(wireframeMesh)
    scene.add(sphere)
  
    const ambientLight = new THREE.AmbientLight(0xffffff, 1.0)
    scene.add(ambientLight)
  
    animate()
}

// animation loop
function animate() {
    requestAnimationFrame(animate)
    
    if (sphere) {
        sphere.rotation.x += 0.003
        sphere.rotation.y += 0.008
        
        const time = Date.now() * 0.001
        const pulseIntensity = 0.1 * Math.sin(time * 2) + 0.9
        
        if (solidMesh && solidMesh.material) {
            solidMesh.material.opacity = 0.4 + 0.1 * pulseIntensity
        }
    }
    
    renderer.render(scene, camera)
}

// calculate distance between two 3D points
function calculateDistance(point1, point2) {
    const dx = point1.x - point2.x
    const dy = point1.y - point2.y
    const dz = point1.z - point2.z
    return Math.sqrt(dx * dx + dy * dy + dz * dz)
}

// check if point is in sphere
function isPointInSphere(point) {
    const worldX = (point.x - 0.5) * 10
    const worldY = (0.5 - point.y) * 10
    const worldZ = 0
    
    const spherePos = new THREE.Vector3()
    sphere.getWorldPosition(spherePos)
    
    const distance = Math.sqrt(
        Math.pow(worldX - spherePos.x, 2) +
        Math.pow(worldY - spherePos.y, 2) +
        Math.pow(worldZ - spherePos.z, 2)
    )
    
    const currentSize = sphere.scale.x * 2
    return distance < currentSize * 1
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

// draw hand landmarks
function drawLandmarks(landmarks, isLeft) {
    const screenSize = Math.min(window.innerWidth, window.innerHeight)
    const lineWidth = Math.max(2, Math.min(5, screenSize / 300))
    const pointSize = Math.max(2, Math.min(8, screenSize / 250))
    
    // mediapipe connections
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
  
    rightHandActive = false
    leftHandActive = false
  
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        statusRef.value.textContent =
        results.multiHandLandmarks.length === 1 ? '1 hand detected' :
        `${results.multiHandLandmarks.length} hands detected`
        
        for (let handIndex = 0; handIndex < results.multiHandLandmarks.length; handIndex++) {
        const landmarks = results.multiHandLandmarks[handIndex]
        const handedness = results.multiHandedness[handIndex].label
        const isLeftHand = handedness === 'Left'
        
        drawLandmarks(landmarks, isLeftHand)
        
        if (!isLeftHand) { // because of the mirroring, right hand is left
            // LEFT HAND: Control sphere size
            const thumbTip = landmarks[4]
            const indexTip = landmarks[8]
            
            const pinchDistance = calculateDistance(thumbTip, indexTip)
            
            if (pinchDistance < 0.05) {
            targetSphereSize = 0.2
            } else if (pinchDistance > 0.25) {
            targetSphereSize = 2.0
            } else {
            targetSphereSize = 0.2 + (pinchDistance - 0.05) * (2.0 - 0.2) / (0.25 - 0.05)
            }
            
            currentSphereSize = currentSphereSize + (targetSphereSize - currentSphereSize) * smoothingFactor
            
            if (sphere) {
            sphere.scale.set(currentSphereSize, currentSphereSize, currentSphereSize)
            }
            
            rightHandActive = true
        } else {
            // RIGHT HAND: Change color
            const indexTip = landmarks[8]
            
            if (isPointInSphere(indexTip)) {
            const currentTime = Date.now()
            if (currentTime - lastColorChangeTime > colorChangeDelay) {
                const newColor = getRandomNeonColor()
                if (solidMesh && solidMesh.material) {
                solidMesh.material.color.setHex(newColor)
                }
                lastColorChangeTime = currentTime
            }
            
            leftHandActive = true
            }
        }
        }
    } else {
        statusRef.value.textContent = 'No hands detected'
    }
}

// Start the application
async function startApp() {
    try {
        await initWebcam()
        initThreeJS()
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
    
    // Stop camera
    if (cameraHelper && cameraHelper.stop) {
        cameraHelper.stop()
    }
    
    // Stop webcam stream
    if (videoRef.value && videoRef.value.srcObject) {
        const tracks = videoRef.value.srcObject.getTracks()
        tracks.forEach(track => track.stop())
    }
    
    // Clean up Three.js
    if (renderer) {
        renderer.dispose()
    }
})
</script>

<style scoped>
body {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: #000;
}

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

#three-canvas {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 5;
}

#status {
  position: absolute;
  top: 20px;
  left: 20px;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.5);
  padding: 10px;
  border-radius: 5px;
  font-family: Arial, sans-serif;
  z-index: 10;
}

#links-para {
  position: absolute;
  bottom: 0px;
  left: 0px;
  font-family: Helvetica, sans-serif;
  font-size: 16px;
  background-color: rgba(255, 255, 255, 0.9);
  padding: 10px;
  margin-bottom: 0;
  z-index: 10;
}
</style>