<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import VertexShader from './shader/vertext.glsl?raw'
import FragmentShader from './shader/fragment.glsl?raw'
import matcap from '/black_cap2.png'
import loadingIco from './components/loadingIco.vue'
import pageWrap from './components/pageWrap.vue'

const webgl = ref(null)
let controls = null
const clock = new THREE.Clock()

// 場景、攝像機、渲染器
const scene = new THREE.Scene()
const sizes = {
    width: window.innerWidth,
    height: window.innerHeight
}

// 調整視窗大小
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

// audio
const isPlaying = ref(false)
const audio = ref(null)
const loading = ref(true)

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

let imageAspect = 1
let a1, a2
if (sizes.height / sizes.width > imageAspect) {
    a1 = (sizes.width / sizes.height) * imageAspect
    a2 = 1
} else {
    a1 = 1
    a2 = (sizes.height / sizes.width) * imageAspect
}

// 設置場景
const setupScene = () => {
    // 渲染器添加到 DOM
    webgl.value.appendChild(renderer.domElement)

    // 物件
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
    window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
    // 清理資源
    window.removeEventListener('resize', handleResize)
    renderer.setAnimationLoop(null) // 停止動畫循環
    renderer.dispose()
})
</script>

<template>
    <pageWrap>
        <div class="outline-none w-full h-full absolute z-0" ref="webgl"></div>
        <div class="absolute h-dvh inset-0 flex items-center justify-center mt-28 sm:mt-0">
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
    </pageWrap>
</template>
