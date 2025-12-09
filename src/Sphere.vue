<template>
    <div ref="threeRef" id="three-canvas"></div>
    <div ref="musicStatusRef" id="music_status">No Music Uploaded</div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, watch } from 'vue'

// props received from HandTracking component
const props = defineProps({
    rightHandData: {
        type: Object,
        required: true
    },
    leftHandData: {
        type: Object,
        required: true
    },
    audioURL: {
        type: String,
        default: null
    }
})

const threeRef = ref(null)

// Three.js variables
let scene, camera, renderer
let sphere, solidMesh, wireframeMesh

// sphere control variables
let lastColorChangeTime = 0 // aka last touch time
const colorChangeDelay = 500 // aka touch delay
let currentSphereSize = 1.0
let targetSphereSize = 1.0
const smoothingFactor = 0.15

// audio variables
let audio = null
let audioCtx = null
let analyzer = null
let dataArray = null
let bufferLength = 0
let noise = new SimplexNoise()
const musicStatusRef = ref(null)

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

// helper functions for audio vis
function fractionate(val, minVal, maxVal) {
    return (val - minVal) / (maxVal - minVal)
}

function modulate(val, minVal, maxVal, outMin, outMax) {
    const fr = fractionate(val, minVal, maxVal)
    const delta = outMax - outMin
    return outMin + (fr * delta)
}

function avg(arr) {
    const total = arr.reduce((sum, b) => sum + b, 0)
    return total / arr.length
}

function max(arr) {
    return arr.reduce((a, b) => Math.max(a, b), 0)
}

function updateMusicStatus() {
    if (!props.audioURL) {
        musicStatusRef.value.textContent = "No Music Uploaded"
    } else if (!audio) {
        musicStatusRef.value.textContent = "Music Uploaded - Paused"
    } else {
        musicStatusRef.value.textContent = audio.paused ? "Music Uploaded - Paused" : "Music Uploaded - Playing"
    }
}

// initialize Three.js
function initThreeJS() {
    scene = new THREE.Scene()
    
    camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    )
    camera.position.z = 100
    
    renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true
    })
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setClearColor(0x000000, 0)
    threeRef.value.appendChild(renderer.domElement)
    
    const geometry = new THREE.IcosahedronGeometry(15, 6)
    
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

// sphere warping based on audio
function warpSphere(mesh, bassFr, treFr){
    if(!mesh || !mesh.geometry) return

    const offset = mesh.geometry.parameters.radius
    const amp = 12
    const time = window.performance.now()
    const positionAttribute = mesh.geometry.getAttribute('position')

    for (let i = 0; i<positionAttribute.count; i++) {
        const vertex = new THREE.Vector3()
        vertex.fromBufferAttribute(positionAttribute, i)        
        vertex.normalize()
        const rf = 0.00003

        // introduce 2 layers of noise for more complex ripples
        const noise1 = noise.noise3D(
            vertex.x + time * rf * 7, 
            vertex.y + time * rf * 8, 
            vertex.z + time * rf * 9)
        
        const noise2 = noise.noise3D(
            vertex.x + time * rf * 3, 
            vertex.y + time * rf * 4, 
            vertex.z + time * rf * 5)
        
        const combinedNoise = (noise1 * 0.7 + noise2 * 0.3) * amp * treFr * 3
        const distance = (offset + bassFr * 2) + combinedNoise
        
        vertex.multiplyScalar(distance)
        positionAttribute.setXYZ(i, vertex.x, vertex.y, vertex.z)
    }
    positionAttribute.needsUpdate = true
    mesh.geometry.computeVertexNormals()
}

// animation loop - handles all sphere updates
function animate() {
    requestAnimationFrame(animate)
    
    if (sphere) {
        // continuous rotation
        sphere.rotation.x += 0.003
        sphere.rotation.y += 0.008
        
        // audio vis
        if (analyzer && dataArray){
            analyzer.getByteFrequencyData(dataArray)
      
            const lowerHalf = dataArray.slice(0, Math.floor(dataArray.length / 2))
            const upperHalf = dataArray.slice(Math.floor(dataArray.length / 2), dataArray.length)
            
            const lowerMax = max(lowerHalf)
            const upperAvg = avg(upperHalf)
            
            const lowerMaxFr = lowerMax / lowerHalf.length
            const upperAvgFr = upperAvg / upperHalf.length
      
            warpSphere(
                solidMesh, 
                modulate(Math.pow(lowerMaxFr, 0.8), 0, 1, 0, 15),
                modulate(upperAvgFr, 0, 1, 0, 8)
            )

            warpSphere(
                wireframeMesh, 
                modulate(Math.pow(lowerMaxFr, 0.8), 0, 1, 0, 15),
                modulate(upperAvgFr, 0, 1, 0, 8)
            )
        }
        
        // handle left hand pinch (resize) - updated every frame
        if (props.leftHandData.active && audio) {
            const pinchDistance = props.leftHandData.pinchDistance
            
            // map pinch distance to sphere size and volume
            let volume
            if (pinchDistance < 0.05) {
                targetSphereSize = 0.1
                volume = 0.0
            } else if (pinchDistance > 0.25) {
                targetSphereSize = 1.0
                volume = 1.0
            } else {
                targetSphereSize = 0.2 + (pinchDistance - 0.05) * (2.0 - 0.2) / (0.25 - 0.05)
                volume = 0.2 + (pinchDistance - 0.05) * (2.0 - 0.2) / (0.25 - 0.05)                
            }
        audio.volume = volume
        
        // smooth transition
        currentSphereSize = currentSphereSize + (targetSphereSize - currentSphereSize) * smoothingFactor
        currentSphereSize = Math.min(Math.max(currentSphereSize, 0.1), 1.0) // clamp
        sphere.scale.set(currentSphereSize, currentSphereSize, currentSphereSize)
        }
        
        // handle right hand touch (color change) - checked every frame
        if (props.rightHandData.active && props.rightHandData.indexTip && audio) {
            if (isPointInSphere(props.rightHandData.indexTip)) {
                const currentTime = Date.now()

                if (currentTime - lastColorChangeTime > colorChangeDelay) {
                    const newColor = getRandomNeonColor()
                    if (solidMesh && solidMesh.material) {
                        solidMesh.material.color.setHex(newColor)
                    }

                    if (audio.paused){
                        audio.play()
                        updateMusicStatus()
                    } else{
                        audio.pause()
                        updateMusicStatus()
                    }
                lastColorChangeTime = currentTime
                }
            }
        }
    }
    
    // render the scene
    renderer.render(scene, camera)
}

// handle window resize
function handleResize() {
    if (renderer) {
        renderer.setSize(window.innerWidth, window.innerHeight)
    }
    if (camera) {
        camera.aspect = window.innerWidth / window.innerHeight
        camera.updateProjectionMatrix()
    }
}

// check if point is in sphere
function isPointInSphere(point) {
    const worldX = (point.x - 0.5) * 200
    const worldY = (0.5 - point.y) * 200
    const worldZ = 0
    
    const spherePos = new THREE.Vector3()
    sphere.getWorldPosition(spherePos)
    
    const distance = Math.sqrt(
        Math.pow(worldX - spherePos.x, 2) +
        Math.pow(worldY - spherePos.y, 2) +
        Math.pow(worldZ - spherePos.z, 2)
    )
    
    const currentSize = sphere.scale.x * 15
    return distance < currentSize
}

// initialize audio

watch(() => props.audioURL, (newURL) => {
    updateMusicStatus()
    if (newURL) {
        // clean up old audio
        if (audio) {
            audio.pause()
            audio = null
        }
        
        // create new audio
        audio = new Audio(newURL)
        
        // set up audio context and analyzer
        if (!audioCtx) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)()
        }
        
        const src = audioCtx.createMediaElementSource(audio)
        analyzer = audioCtx.createAnalyser()
        
        src.connect(analyzer)
        analyzer.connect(audioCtx.destination)
        
        analyzer.fftSize = 512
        bufferLength = analyzer.frequencyBinCount
        dataArray = new Uint8Array(bufferLength)
    }
})

onMounted(() => {
    initThreeJS()
    window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', handleResize)
    
    if (renderer) {
        renderer.dispose()
    }
})
</script>

<style scoped>
#three-canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    z-index: 5;
}

#music_status{ /* place it below to the hand tracking status */
    position: absolute;
    top: 65px;
    left: 20px;
    color: #fff;
    background-color: rgba(0, 0, 0, 0.5);
    padding: 10px;
    border-radius: 5px;
    font-family: Helvetica, sans-serif;
    z-index: 10;
}
</style>