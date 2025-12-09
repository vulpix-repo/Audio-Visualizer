<template>
    <div ref="threeRef" id="three-canvas"></div>
</template>

<script setup>
import { onMounted, onBeforeUnmount, ref, watchEffect } from 'vue'

// props received from HandTracking component
const props = defineProps({
    rightHandData: {
        type: Object,
        required: true
    },
    leftHandData: {
        type: Object,
        required: true
    }
})

const threeRef = ref(null)

// Three.js variables
let scene, camera, renderer
let sphere, solidMesh, wireframeMesh

// sphere control variables
let lastColorChangeTime = 0
const colorChangeDelay = 500
let currentSphereSize = 1.0
let targetSphereSize = 1.0
const smoothingFactor = 0.15



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
    
    camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    )
    camera.position.z = 5
    
    renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true
    })
    renderer.setSize(window.innerWidth, window.innerHeight)
    renderer.setClearColor(0x000000, 0)
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

// animation loop - handles all sphere updates
function animate() {
    requestAnimationFrame(animate)
    
    if (sphere) {
        // Continuous rotation
        sphere.rotation.x += 0.003
        sphere.rotation.y += 0.008
        
        // Pulsing glow effect
        const time = Date.now() * 0.001
        const pulseIntensity = 0.1 * Math.sin(time * 2) + 0.9
        
        if (solidMesh && solidMesh.material) {
        solidMesh.material.opacity = 0.4 + 0.1 * pulseIntensity
        }
        
        // handle left hand pinch (resize) - updated every frame
        if (props.leftHandData.active) {
        const pinchDistance = props.leftHandData.pinchDistance
        
        // map pinch distance to sphere size
        if (pinchDistance < 0.05) {
            targetSphereSize = 0.2
        } else if (pinchDistance > 0.25) {
            targetSphereSize = 2.0
        } else {
            targetSphereSize = 0.2 + (pinchDistance - 0.05) * (2.0 - 0.2) / (0.25 - 0.05)
        }
        
        // smooth transition
        currentSphereSize = currentSphereSize + (targetSphereSize - currentSphereSize) * smoothingFactor
        sphere.scale.set(currentSphereSize, currentSphereSize, currentSphereSize)
        }
        
        // handle right hand touch (color change) - checked every frame
        if (props.rightHandData.active && props.rightHandData.indexTip) {
        if (isPointInSphere(props.rightHandData.indexTip)) {
            const currentTime = Date.now()
            if (currentTime - lastColorChangeTime > colorChangeDelay) {
            const newColor = getRandomNeonColor()
            if (solidMesh && solidMesh.material) {
                solidMesh.material.color.setHex(newColor)
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
</style>