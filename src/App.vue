<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
// import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import VertexShader from './shader/vertext.glsl?raw'
import FragmentShader from './shader/fragment.glsl?raw'
import matcap from '/black_cap2.png'
import loadingIco from './components/loadingIco.vue'
import pageWrap from './components/pageWrap.vue'
import Header from './components/header.vue'
import footerInfo from './components/footerInfo.vue'

// Ref
const webgl = ref(null)
let controls = null

// Scene
const scene = new THREE.Scene()

// Clock for consistent animations
const clock = new THREE.Clock()

// Loading Manager
const loadingManager = new THREE.LoadingManager(
    () => {
        loading.value = false
        isPlaying.value = false
    },
    (file, loaded, total) => {
        const progress = loaded / total
        console.log(`Loading audio: ${progress * 100}%`)
    }
)

// Resize
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

// Camera
let frustumSize = 1
const camera = new THREE.OrthographicCamera(
    frustumSize / -2,
    frustumSize / 2,
    frustumSize / 2,
    frustumSize / -2,
    0.1,
    1000
)
camera.position.z = 1
scene.add(camera)

// Renderer
const renderer = new THREE.WebGLRenderer({ antialias: true })
renderer.setSize(sizes.width, sizes.height)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))

// Handle Resize
window.addEventListener('resize', () => {
    // Update sizes
    sizes.width = window.innerWidth
    sizes.height = window.innerHeight

    // Update camera
    camera.aspect = sizes.width / sizes.height
    camera.updateProjectionMatrix()

    // Update renderer
    renderer.setSize(sizes.width, sizes.height)
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

// Audio --------------------------------------
const isPlaying = ref(false)
const audio = ref(null)
const loading = ref(true)

const file = '/knekksans.mp3'
const fftSize = 256
const listener = new THREE.AudioListener()
audio.value = new THREE.Audio(listener)
const audioLoader = new THREE.AudioLoader(loadingManager)

audioLoader.load(file, function (buffer) {
    audio.value.setBuffer(buffer)
})

const togglePlay = () => {
    if (isPlaying.value) {
        pauseAudio()
    } else {
        playAudio()
    }
}

const playAudio = () => {
    listener.context
        .resume()
        .then(() => {
            audio.value.play()
            isPlaying.value = true
        })
        .catch((error) => {
            console.error('Failed to resume audio context:', error)
        })
}

const pauseAudio = () => {
    audio.value.pause()
    isPlaying.value = false
}

const analyser = new THREE.AudioAnalyser(audio.value, fftSize)

// Object Handle Resize
let imageAspect = 1
let a1, a2
if (sizes.height / sizes.width > imageAspect) {
    a1 = (sizes.width / sizes.height) * imageAspect
    a2 = 1
} else {
    a1 = 1
    a2 = (sizes.height / sizes.width) * imageAspect
}

// Scene Setup
const setupScene = () => {
    // Canvas
    webgl.value.appendChild(renderer.domElement)

    // Object
    const geometry = new THREE.PlaneGeometry(1, 1, 1, 1)
    const material = new THREE.ShaderMaterial({
        vertexShader: VertexShader,
        fragmentShader: FragmentShader,
        uniforms: {
            time: { value: 2.5 },
            progress: { value: 0.0 },
            uFrequency: { value: 0.0 },
            mouse: { value: new THREE.Vector2(0, 0) },
            matcap: { value: new THREE.TextureLoader().load(matcap) },
            resolution: { value: new THREE.Vector4(sizes.width, sizes.height, a1, a2) }
        },
        wireframe: false
    })
    const plane = new THREE.Mesh(geometry, material)
    scene.add(plane)

    const tick = () => {
        const elapsedTime = clock.getElapsedTime()

        material.uniforms.time.value = elapsedTime
        material.uniforms.uFrequency.value = analyser.getAverageFrequency()

        renderer.render(scene, camera)

        window.requestAnimationFrame(tick)
    }
    tick()
}

onMounted(() => {
    setupScene()
})

onBeforeUnmount(() => {
    window.addEventListener('resize')
    renderer.setAnimationLoop(null) // 停止動畫循環
    renderer.dispose()
    if (audio.value) {
        audio.value.stop() // 停止音頻播放
        audio.value.disconnect() // 斷開音頻連接
        audio.value.buffer = null
        listener.remove(audio.value)
    }
})
</script>

<template>
    <pageWrap>
        <Header />
        <div class="outline-none w-full h-dvh" ref="webgl"></div>
        <div class="absolute inset-0 h-dvh flex items-center justify-center mt-36 sm:mt-0">
            <div v-if="loading" class="z-10">
                <loadingIco />
            </div>
            <button
                v-else
                @click="togglePlay"
                class="z-10 border border-zinc-700 cursor-pointer hover:border-rose-600 rounded-2xl w-20 h-8 sm:w-20 sm:h-8 bg-stone-950 text-gray-200 opacity-90 font-extralight text-xs"
            >
                {{ isPlaying ? 'PAUSE' : 'PLAY' }}
            </button>
        </div>
        <footerInfo>
            <template v-slot:first></template>
            <template v-slot:second>
                <a
                    href="https://www.youtube.com/watch?v=3OKPA8ne57A&ab_channel=RayFerrofluid"
                    target="_blank"
                    class="underline-offset-2"
                    >Ferrofluid Speaker Audio Visualizer.
                </a>
            </template>
            <template v-slot:github>
                <a href="https://github.com/pccathome/audio-visualiser" target="_blank" class="underline-offset-2"
                    >GitHub</a
                >
            </template>
        </footerInfo>
    </pageWrap>
</template>
