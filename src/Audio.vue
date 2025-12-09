<template>
    <label for="audio" id="label">Upload .mp3 Audio</label>
    <input type="file" id="audio" accept=".mp3" @change="handleAudioUpload" ref="audioRef"/>
    <p id="instructions">
        - Left hand: Pinch thumb and index finger to adjust volume<br>
        - Right hand: Touch the sphere with your index finger to play/pause
    </p>
</template>

<script setup>
import { ref, defineEmits } from 'vue'

const audioRef = ref(null)
const emit = defineEmits(['audioSelected'])

// audio selection
function handleAudioUpload(event){
    const file = event.target.files[0]
    if (file) {
        // check if mp3
        if(file.name.includes('.mp3')){
            const audioURL = URL.createObjectURL(file)
            emit('audioSelected', audioURL)
        }
        else{
        alert('Invalid .mp3 file')
        }
    }
}
</script>

<style scoped>

#instructions {
    position:absolute;
    bottom: 20px;
    left: 20px;
    font-family: Helvetica, sans-serif;
    font-size: 16px;
    color: #fff;
    background-color: rgba(0, 0, 0, 0.5);
    border-radius: 5px;
    padding: 10px;
}

#label {
    position:absolute;
    bottom: 100px;
    left: 20px;
    background:rgba(0, 0, 0, 0.5);
    color: white;
    padding: 10px 16px;
    border-radius: 6px;
    font-family: Helvetica, sans-serif;
    font-size: 16px;
    cursor: pointer;
    display: inline-block;
    transition: background 0.2s, transform 0.1s;
    z-index: 20;
}

#label:hover {
    background: rgba(0, 0, 0, 0.70);
}

#label:active {
    transform: scale(0.97);
}


#audio{
    display: none;
}
</style>
