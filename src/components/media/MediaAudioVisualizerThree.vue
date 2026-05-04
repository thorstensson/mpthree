<script setup lang="ts">
import { ref, onMounted, onUnmounted, inject } from "vue"
import { useEventListener } from "@vueuse/core"
import * as THREE from "three"
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js"
import { RenderPass } from "three/addons/postprocessing/RenderPass.js"
import { AfterimagePass } from "three/addons/postprocessing/AfterimagePass.js"
import { OutputPass } from "three/addons/postprocessing/OutputPass.js"

const container = ref<HTMLDivElement>()
const analyser = ref<AnalyserNode>()

let renderer: THREE.WebGLRenderer | null = null
let scene: THREE.Scene | null = null
let camera: THREE.PerspectiveCamera | null = null
let composer: EffectComposer | null = null
let particles: THREE.Points | null = null
let animationId: number | null = null
let audioCtx: AudioContext
let dataArray: Uint8Array
let bufferLength: number
let isPlaying = false
let audioContextInitialized = false

// Audio-reactive uniforms — three frequency bands
const uniforms = {
  uTime: { value: 0 },
  uPlaying: { value: 0 },
  uBass: { value: 0 },
  uMid: { value: 0 },
  uHigh: { value: 0 },
  uGlobalAmp: { value: 0 },
  uCamZ: { value: 1000 },
}

/** Pre-generated circle sprite texture */
let spriteTexture: THREE.Texture | null = null

const audioEl = inject<{ value: HTMLAudioElement | null }>("audioEl")

/**
 * Create Web Audio API analyser
 */
const createAnalyserData = () => {
  if (!audioEl?.value || !(audioEl.value instanceof HTMLMediaElement)) {
    console.warn("Audio element not available yet")
    return false
  }

  if (audioContextInitialized) return true

  try {
    audioCtx = new AudioContext()
    const audioSrc = audioCtx.createMediaElementSource(
      audioEl.value as HTMLMediaElement
    )

    analyser.value = audioCtx.createAnalyser()
    audioSrc.connect(analyser.value)
    analyser.value.connect(audioCtx.destination)
    analyser.value.fftSize = 256

    bufferLength = analyser.value.frequencyBinCount
    dataArray = new Uint8Array(bufferLength)

    if (audioCtx.state === "suspended") {
      audioCtx.resume()
    }

    audioContextInitialized = true
    console.log("Three.js visualizer: Audio context initialized successfully")
    return true
  } catch (error) {
    console.error("Failed to create audio context:", error)
    return false
  }
}

/**
 * Generate a soft circle sprite as a canvas texture
 */
const generateSpriteTexture = () => {
  const canvas = document.createElement("canvas")
  canvas.width = 64
  canvas.height = 64
  const ctx = canvas.getContext("2d")!

  const gradient = ctx.createRadialGradient(32, 32, 0, 32, 32, 32)
  gradient.addColorStop(0, "rgba(255,255,255,1)")
  gradient.addColorStop(0.3, "rgba(255,255,255,0.6)")
  gradient.addColorStop(1, "rgba(255,255,255,0)")

  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, 64, 64)

  spriteTexture = new THREE.CanvasTexture(canvas)
}

/**
 * Initialize Three.js scene
 */
const initThree = () => {
  if (!container.value) return

  generateSpriteTexture()

  const width = window.innerWidth
  const height = window.innerHeight

  renderer = new THREE.WebGLRenderer({
    alpha: true,
    antialias: false,
    powerPreference: "high-performance",
  })
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setClearColor(0x000000, 0)
  container.value.appendChild(renderer.domElement)

  scene = new THREE.Scene()

  camera = new THREE.PerspectiveCamera(60, width / height, 1, 10000)
  camera.position.z = 1000

  // --- Particle System ---
  const particleCount = 25000
  const radius = 600

  const geometry = new THREE.BufferGeometry()
  const positions = new Float32Array(particleCount * 3)
  const baseHues = new Float32Array(particleCount) // store base hue per particle
  const offsets = new Float32Array(particleCount)

  // Muted teal palette — accent colors from the project
  // accent1: #677e84 (hsl ~190, 13%, 46%)
  // accent2: #84a6a5 (hsl ~175, 16%, 58%)
  const hueMin = 170
  const hueMax = 200

  for (let i = 0; i < particleCount; i++) {
    const pos = getRandomPointOnSphere(radius)
    positions[i * 3] = pos.x
    positions[i * 3 + 1] = pos.y
    positions[i * 3 + 2] = pos.z

    // Spread each particle's base hue across the muted teal range
    baseHues[i] = hueMin + (i / particleCount) * (hueMax - hueMin)

    offsets[i] = i / particleCount
  }

  geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3))
  geometry.setAttribute("baseHue", new THREE.BufferAttribute(baseHues, 1))
  geometry.setAttribute("offset", new THREE.BufferAttribute(offsets, 1))

  // Vertex shader with per-band audio reactivity + muted color
  const vertexShader = /* glsl */ `
    attribute float baseHue;
    attribute float offset;
    varying vec3 vColor;
    varying float vAlpha;

    uniform float uTime;
    uniform float uPlaying;
    uniform float uBass;
    uniform float uMid;
    uniform float uHigh;
    uniform float uGlobalAmp;
    uniform float uCamZ;

    // Convert HSL to RGB
    vec3 hsl2rgb(float h, float s, float l) {
      vec3 rgb = clamp(abs(mod(h * 6.0 + vec3(0.0, 4.0, 2.0), 6.0) - 3.0) - 1.0, 0.0, 1.0);
      return l + s * (rgb - 0.5) * (1.0 - abs(2.0 * l - 1.0));
    }

    void main() {
      vec3 pos = position;

      // --- Position animation ---
      float t = mod(uTime * 0.1 + offset, 1.0);

      // Idle: gentle wobble
      float idleWobble = sin(uTime * 0.5 + offset * 6.28) * 5.0;
      vec3 idlePos = pos + normalize(pos) * idleWobble;

      // Spiral animates at constant smooth speed — no audio jitter
      float st = mod(uTime * 0.1 + offset, 1.0);
      float accTime = st * st;
      float angle = accTime * 40.0;

      vec3 playPos = vec3(
        pos.x * accTime + sin(angle) * 20.0,
        pos.y * accTime + cos(angle) * 20.0,
        pos.z * accTime * 1.75
      );

      float blend = smoothstep(0.0, 1.0, uPlaying);
      vec3 finalPos = mix(idlePos, playPos, blend);

      // --- Color: wide teal spectrum ---
      // Mids shift toward cyan-green
      // Highs shift toward blue
      // Bass drives brightness
      float hue = 180.0;
      hue += uHigh * 70.0;    // highs push toward blue (~250)
      hue -= uMid * 40.0;     // mids push toward cyan (~140)

      float total = uBass + uMid + uHigh;
      float sat = 0.1 + total * 0.7;
      float light = 0.15 + uBass * 0.6 + uHigh * 0.15;

      hue = clamp(hue, 130.0, 270.0);

      // Mute colors when camera zooms in
      float camFactor = clamp((uCamZ - 200.0) / 500.0, 0.35, 1.0);
      sat *= camFactor;
      light *= camFactor;

      sat = min(sat, 0.8);
      light = min(light, 0.85);

      vColor = hsl2rgb(hue / 360.0, sat, light);

      // --- Alpha ---
      float fadeIn = uPlaying > 0.5 ? 1.0 : 0.15;
      if (uPlaying > 0.5) {
        // Only visible during the spiral pass — fade in/out tight
        fadeIn = smoothstep(0.0, 0.15, st) * (1.0 - smoothstep(0.6, 0.9, st));
        fadeIn = mix(fadeIn, 1.0, uBass * 0.7);
      }
      vAlpha = fadeIn;

      vec4 mvPosition = modelViewMatrix * vec4(finalPos, 1.0);
      gl_PointSize = 5.0 * (300.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `

  const fragmentShader = /* glsl */ `
    varying vec3 vColor;
    varying float vAlpha;
    uniform sampler2D uTexture;

    void main() {
      vec4 texColor = texture2D(uTexture, gl_PointCoord);
      gl_FragColor = vec4(vColor, texColor.a * vAlpha);
    }
  `

  const material = new THREE.ShaderMaterial({
    uniforms: {
      uTime: uniforms.uTime,
      uPlaying: uniforms.uPlaying,
      uBass: uniforms.uBass,
      uMid: uniforms.uMid,
      uHigh: uniforms.uHigh,
      uGlobalAmp: uniforms.uGlobalAmp,
      uCamZ: uniforms.uCamZ,
      uTexture: { value: spriteTexture! },
    },
    vertexShader,
    fragmentShader,
    transparent: true,
    depthWrite: false,
    blending: THREE.AdditiveBlending,
  })

  particles = new THREE.Points(geometry, material)
  scene.add(particles)

  // --- Post-processing ---
  composer = new EffectComposer(renderer)

  const renderPass = new RenderPass(scene, camera)
  composer.addPass(renderPass)

  const afterimagePass = new AfterimagePass(0.82)
  composer.addPass(afterimagePass)

  const outputPass = new OutputPass()
  composer.addPass(outputPass)

  window.addEventListener("resize", onWindowResize)
}

/**
 * Get a random point on a sphere surface
 */
const getRandomPointOnSphere = (r: number) => {
  const angle = Math.random() * Math.PI * 2
  const u = Math.random() * 2 - 1
  return new THREE.Vector3(
    Math.cos(angle) * Math.sqrt(1 - u * u) * r,
    Math.sin(angle) * Math.sqrt(1 - u * u) * r,
    u * r
  )
}

/**
 * Handle window resize
 */
const onWindowResize = () => {
  if (!camera || !renderer || !composer) return

  const width = window.innerWidth
  const height = window.innerHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
  composer.setSize(width, height)
}

/**
 * Start the animation
 */
const startAnimation = () => {
  isPlaying = true
}

/**
 * Stop the animation
 */
const stopAnimation = () => {
  isPlaying = false
}

/**
 * Extract frequency bands from the full spectrum
 */
const updateFrequencyBands = () => {
  if (!analyser.value) return

  analyser.value.getByteFrequencyData(dataArray)

  const len = dataArray.length
  const third = Math.floor(len / 3)

  let bass = 0
  let mid = 0
  let high = 0

  for (let i = 0; i < len; i++) {
    if (i < third) {
      bass += dataArray[i]
    } else if (i < third * 2) {
      mid += dataArray[i]
    } else {
      high += dataArray[i]
    }
  }

  const bassNorm = bass / third / 255
  const midNorm = mid / third / 255
  const highNorm = high / (len - third * 2) / 255

  // Smooth with previous values (low-pass filter)
  uniforms.uGlobalAmp.value = (bassNorm + midNorm + highNorm) / 3
  uniforms.uBass.value = uniforms.uBass.value * 0.7 + bassNorm * 0.3
  uniforms.uMid.value = uniforms.uMid.value * 0.7 + midNorm * 0.3
  uniforms.uHigh.value = uniforms.uHigh.value * 0.7 + highNorm * 0.3
}

/**
 * Animation loop
 */
const animate = (time: number) => {
  animationId = requestAnimationFrame(animate)

  if (!composer || !particles || !scene) return

  if (isPlaying && analyser.value) {
    updateFrequencyBands()
  } else {
    // Decay all bands smoothly when idle
    uniforms.uBass.value *= 0.96
    uniforms.uMid.value *= 0.96
    uniforms.uHigh.value *= 0.96
    uniforms.uGlobalAmp.value *= 0.96
  }

  uniforms.uTime.value = time * 0.001
  uniforms.uPlaying.value = isPlaying ? 1 : 0

  particles.rotation.z = time * 0.0003

  // Camera breathing — fast zoom in on bass, slow drift back
  if (camera) {
    const targetZ = 1000 - uniforms.uBass.value * 700
    camera.position.z += (targetZ - camera.position.z) * 0.08
    uniforms.uCamZ.value = camera.position.z
  }

  composer.render()
}

onMounted(() => {
  initThree()

  if (audioEl?.value) {
    audioEl.value.addEventListener("playing", startAnimation)
    audioEl.value.addEventListener("pause", stopAnimation)
    audioEl.value.addEventListener("ended", stopAnimation)
  }

  const cleanup = useEventListener(document, "mousedown", () => {
    cleanup()
    if (createAnalyserData()) {
      console.log("Three.js visualizer: Audio context created")
    }
  })

  animationId = requestAnimationFrame(animate)
})

onUnmounted(() => {
  if (animationId) {
    cancelAnimationFrame(animationId)
  }

  if (audioEl?.value) {
    audioEl.value.removeEventListener("playing", startAnimation)
    audioEl.value.removeEventListener("pause", stopAnimation)
    audioEl.value.removeEventListener("ended", stopAnimation)
  }

  if (spriteTexture) spriteTexture.dispose()
  if (renderer) {
    renderer.dispose()
    renderer.domElement.remove()
  }
  if (composer) composer.dispose()

  window.removeEventListener("resize", onWindowResize)
})
</script>

<template>
  <div ref="container" class="three-container"></div>
</template>

<style scoped lang="scss">
.three-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 0;
  pointer-events: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
}
</style>
