<script setup>
import { ref, onMounted, onBeforeUnmount, computed } from 'vue';

// 1. 引入 Three.js 核心
import * as THREE from 'three';

// 2. 引入 Three.js 扩展库 (注意路径是 examples/jsm/...)
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { FontLoader } from 'three/examples/jsm/loaders/FontLoader.js';
import { TextGeometry } from 'three/examples/jsm/geometries/TextGeometry.js';
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';

// 3. 引入 MediaPipe Hands
import { Hands } from '@mediapipe/hands';

// --- Refs & State ---
const videoElement = ref(null);
const canvasContainer = ref(null);
const isLoading = ref(true);
const loadingMessage = ref('INITIALIZING SYSTEMS...');

const systemStatus = ref('OFFLINE');
const fingerCount = ref(0);
const particleCount = ref(20000);

const currentMode = ref('rain');
const currentShapeNumber = ref(1);

// --- Three.js Variables ---
let scene, camera, renderer, composer, controls;
let geometry, particleSystem;
let positions, velocities, colors;
let loadedFont = null;
const numberPoints = {}; 
let animationId = null;

// --- Constants ---
const PARTICLE_COUNT = 20000;
const RAIN_SPEED = 1.2; 
const AGGREGATION_SPEED = 0.08; 
const COLOR_RAIN = 0x0088ff;
const COLOR_SHAPE = 0x00ffff; 
const COLOR_FREEZE = 0x0088ff;

// --- MediaPipe Variables ---
let hands;
let lastFingerCount = -1;
let frameStabilityCounter = 0;
const STABILITY_THRESHOLD = 8;

// --- Computed ---
const statusClass = computed(() => {
    if (systemStatus.value === 'OFFLINE') return 'status-off';
    if (systemStatus.value === 'SEARCHING') return 'status-search';
    return 'status-on';
});

const modeDisplay = computed(() => {
    if (currentMode.value === 'rain') return "SLOW RAIN";
    if (currentMode.value === 'freeze') return "TIME STOP";
    if (currentMode.value === 'shape') return `SOLID FORM: ${currentShapeNumber.value}`;
    return "UNKNOWN";
});

const modeColor = computed(() => {
    if (currentMode.value === 'rain' || currentMode.value === 'freeze') return "#00aaff";
    return "#00ffff";
});

// --- Lifecycle ---
onMounted(() => {
    initThreeJS();
    initMediaPipe();
    window.addEventListener('resize', onWindowResize);
});

onBeforeUnmount(() => {
    cancelAnimationFrame(animationId);
    window.removeEventListener('resize', onWindowResize);
    if (hands) hands.close();
});

// --- Three.js Functions ---
function initThreeJS() {
    scene = new THREE.Scene();
    scene.fog = new THREE.FogExp2(0x000000, 0.003);

    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.set(0, 0, 110);
    camera.lookAt(0, 0, 0);

    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    
    if (canvasContainer.value) {
        canvasContainer.value.appendChild(renderer.domElement);
    }

    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.autoRotate = false;
    
    const ambientLight = new THREE.AmbientLight(0x404040);
    scene.add(ambientLight);
    
    createParticles();
    initPostProcessing();
    loadFontAndStart();
}

function createParticles() {
    geometry = new THREE.BufferGeometry();
    positions = new Float32Array(PARTICLE_COUNT * 3);
    colors = new Float32Array(PARTICLE_COUNT * 3);
    velocities = new Float32Array(PARTICLE_COUNT * 3);

    const spread = 200;
    const color = new THREE.Color(COLOR_RAIN);

    for (let i = 0; i < PARTICLE_COUNT; i++) {
        positions[i * 3] = (Math.random() - 0.5) * spread;
        positions[i * 3 + 1] = (Math.random() - 0.5) * spread * 2;
        positions[i * 3 + 2] = (Math.random() - 0.5) * spread;
        velocities[i * 3 + 1] = - (Math.random() * 0.5 + 0.5) * RAIN_SPEED;
        colors[i * 3] = color.r;
        colors[i * 3 + 1] = color.g;
        colors[i * 3 + 2] = color.b;
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    
    // 注意：TextureLoader 在 Vite 中加载本地资源可能需要 import 图片路径
    // 这里暂时使用网络图片，生产环境建议下载到 assets 文件夹
    const sprite = new THREE.TextureLoader().load('https://threejs.org/examples/textures/sprites/disc.png');

    const material = new THREE.PointsMaterial({
        vertexColors: true,
        size: 0.7,
        map: sprite,
        transparent: true,
        opacity: 0.8,
        blending: THREE.AdditiveBlending,
        depthWrite: false
    });

    particleSystem = new THREE.Points(geometry, material);
    scene.add(particleSystem);
}

function initPostProcessing() {
    const renderScene = new RenderPass(scene, camera);
    const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
    bloomPass.threshold = 0;
    bloomPass.strength = 1.5;
    bloomPass.radius = 0.5;

    composer = new EffectComposer(renderer);
    composer.addPass(renderScene);
    composer.addPass(bloomPass);
}

function loadFontAndStart() {
    const loader = new FontLoader();
    // 同样，字体文件建议下载到本地 public 目录，这里用 CDN 演示
    loader.load('https://cdn.jsdelivr.net/npm/three@0.128.0/examples/fonts/helvetiker_bold.typeface.json', function (font) {
        loadedFont = font;
        for(let i=1; i<=4; i++) {
            numberPoints[i] = generateHighDefPoints(i.toString());
        }
        loadingMessage.value = "SYSTEM READY";
        setTimeout(() => {
            isLoading.value = false;
        }, 500);
        animate();
    });
}

function generateHighDefPoints(text) {
    const points = [];
    let densityMultiplier = 3;
    if (text === '1') densityMultiplier = 14; 
    if (text === '4') densityMultiplier = 8; 
    
    const textGeo = new TextGeometry(text, {
        font: loadedFont,
        size: 80, height: 5, curveSegments: 6, bevelEnabled: false 
    });
    textGeo.center();
    
    const pos = textGeo.attributes.position;
    const count = pos.count;
    
    for (let i = 0; i < count; i += 3) {
        const ax = pos.getX(i);   const ay = pos.getY(i);   const az = pos.getZ(i);
        const bx = pos.getX(i+1); const by = pos.getY(i+1); const bz = pos.getZ(i+1);
        const cx = pos.getX(i+2); const cy = pos.getY(i+2); const cz = pos.getZ(i+2);

        for(let j=0; j<densityMultiplier; j++) {
            const r1 = Math.random();
            const r2 = Math.random();
            const sqrtR1 = Math.sqrt(r1);
            const u = 1 - sqrtR1;
            const v = sqrtR1 * (1 - r2);
            const w = sqrtR1 * r2;
            
            points.push(
                u*ax + v*bx + w*cx,
                u*ay + v*by + w*cy,
                u*az + v*bz + w*cz
            );
        }
    }
    return points;
}

function animate() {
    animationId = requestAnimationFrame(animate);
    const time = Date.now() * 0.001;
    const pos = geometry.attributes.position.array;
    const col = geometry.attributes.color.array;
    
    const cRain = new THREE.Color(COLOR_RAIN);
    const cShape = new THREE.Color(COLOR_SHAPE);
    const cFreeze = new THREE.Color(COLOR_FREEZE);
    const cDim = new THREE.Color(0x002244); 

    let targets = null;
    if (currentMode.value === 'shape') {
        targets = numberPoints[currentShapeNumber.value];
    }

    for (let i = 0; i < PARTICLE_COUNT; i++) {
        const idx = i * 3;
        if (currentMode.value === 'shape' && targets) {
            const targetIdx = i % (targets.length / 3); 
            if (i < targets.length / 3) {
                const tx = targets[targetIdx * 3];
                const ty = targets[targetIdx * 3 + 1];
                const tz = targets[targetIdx * 3 + 2];
                pos[idx] += (tx - pos[idx]) * AGGREGATION_SPEED;
                pos[idx+1] += (ty - pos[idx+1]) * AGGREGATION_SPEED;
                pos[idx+2] += (tz - pos[idx+2]) * AGGREGATION_SPEED;
                col[idx] = cShape.r; col[idx+1] = cShape.g; col[idx+2] = cShape.b;
            } else {
                pos[idx+1] += velocities[idx+1];
                if (pos[idx+1] < -100) {
                    pos[idx+1] = 100;
                    pos[idx] = (Math.random() - 0.5) * 200;
                    pos[idx+2] = (Math.random() - 0.5) * 200;
                }
                col[idx] = cDim.r; col[idx+1] = cDim.g; col[idx+2] = cDim.b;
            }
        } else if (currentMode.value === 'freeze') {
            const ix = i % 100;
            pos[idx + 1] += Math.sin(time * 2 + ix) * 0.05;
            col[idx] = cFreeze.r; col[idx+1] = cFreeze.g; col[idx+2] = cFreeze.b;
        } else {
            pos[idx + 1] += velocities[idx + 1];
            if (pos[idx + 1] < -100) {
                pos[idx + 1] = 100;
                pos[idx] = (Math.random() - 0.5) * 200;
                pos[idx + 2] = (Math.random() - 0.5) * 200;
            }
            col[idx] = cRain.r; col[idx+1] = cRain.g; col[idx+2] = cRain.b;
        }
    }
    geometry.attributes.position.needsUpdate = true;
    geometry.attributes.color.needsUpdate = true;
    controls.update();
    composer.render();
}

function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    composer.setSize(window.innerWidth, window.innerHeight);
}

// --- MediaPipe Logic ---
async function initMediaPipe() {
    // 关键修改：在 npm 环境中，MediaPipe 需要知道 WASM 文件的位置
    // 最简单的方法是使用 CDN 链接作为 locateFile
    hands = new Hands({
        locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
    });
    
    hands.setOptions({
        maxNumHands: 1,
        modelComplexity: 0,
        minDetectionConfidence: 0.5,
        minTrackingConfidence: 0.5
    });
    hands.onResults(onHandsResults);

    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
        try {
            const stream = await navigator.mediaDevices.getUserMedia({ 
                video: { width: 640, height: 480, facingMode: 'user' } 
            });
            if (videoElement.value) {
                videoElement.value.srcObject = stream;
                videoElement.value.onloadedmetadata = () => {
                    videoElement.value.play();
                    detectHands();
                };
            }
        } catch (e) {
            console.error("Camera failed:", e);
            systemStatus.value = "NO CAM";
        }
    }
}

async function detectHands() {
    if (videoElement.value && videoElement.value.currentTime > 0 && !videoElement.value.paused && !videoElement.value.ended) {
        await hands.send({image: videoElement.value});
    }
    requestAnimationFrame(detectHands);
}

function onHandsResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        systemStatus.value = "ONLINE";
        const fingers = countFingers(results.multiHandLandmarks[0]);
        fingerCount.value = fingers;
        
        if (fingers === lastFingerCount) {
            frameStabilityCounter++;
            if (frameStabilityCounter > STABILITY_THRESHOLD) {
                updateModeByFingers(fingers);
            }
        } else {
            lastFingerCount = fingers;
            frameStabilityCounter = 0;
        }
    } else {
        systemStatus.value = "SEARCHING";
    }
}

function countFingers(landmarks) {
    let count = 0;
    if (landmarks[8].y < landmarks[6].y) count++;
    if (landmarks[12].y < landmarks[10].y) count++;
    if (landmarks[16].y < landmarks[14].y) count++;
    if (landmarks[20].y < landmarks[18].y) count++;
    if (Math.hypot(landmarks[4].x - landmarks[17].x, landmarks[4].y - landmarks[17].y) > 
        Math.hypot(landmarks[3].x - landmarks[17].x, landmarks[3].y - landmarks[17].y)) {
        count++;
    }
    return count;
}

function updateModeByFingers(count) {
    if (
        (count === 0 && currentMode.value === 'rain') ||
        (count === 5 && currentMode.value === 'freeze') ||
        (count >= 1 && count <= 4 && currentMode.value === 'shape' && currentShapeNumber.value === count)
    ) return;

    if (count === 0) {
        currentMode.value = 'rain';
    } else if (count >= 1 && count <= 4) {
        currentMode.value = 'shape';
        currentShapeNumber.value = count;
    } else if (count === 5) {
        currentMode.value = 'freeze';
    }
}
</script>

<template>
    <div id="rain-app">
        <!-- Loading Overlay -->
        <transition name="fade">
            <div v-if="isLoading" class="loader">
                <svg width="50" height="50" viewBox="0 0 50 50">
                    <circle cx="25" cy="25" r="20" stroke="#00ffff" stroke-width="4" fill="none" stroke-dasharray="100" stroke-dashoffset="0">
                        <animate attributeName="stroke-dashoffset" from="0" to="200" dur="2s" repeatCount="indefinite" />
                    </circle>
                </svg>
                <div class="loader-text">{{ loadingMessage }}</div>
            </div>
        </transition>

        <!-- Video Input -->
        <div class="video-container">
            <div class="video-label">OPERATOR FEED /// LIVE</div>
            <video ref="videoElement" playsinline autoplay muted></video>
        </div>

        <!-- UI Overlay -->
        <div id="ui-layer">
            <div class="hud-header">
                <h1>Rain Controller v3.0 (Vue)</h1>
                <div style="font-size: 10px; opacity: 0.7; margin-top: 5px;">> ADAPTIVE DENSITY AGGREGATION</div>
            </div>
            
            <div class="status-panel">
                <div class="status-item">
                    <span>SYSTEM STATUS</span> 
                    <span :class="statusClass" class="status-value">{{ systemStatus }}</span>
                </div>
                <div class="status-item">
                    <span>MODE</span> 
                    <span :style="{ color: modeColor }" class="status-value">{{ modeDisplay }}</span>
                </div>
                <div class="status-item">
                    <span>FINGERS</span> 
                    <span class="status-value">{{ fingerCount }}</span>
                </div>
                <div class="status-item">
                    <span>PARTICLES</span> 
                    <span class="status-value">{{ particleCount.toLocaleString() }}</span>
                </div>
            </div>
        </div>

        <!-- Three.js Canvas -->
        <div id="canvas-container" ref="canvasContainer"></div>
    </div>
</template>

<style scoped>


#rain-app {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    position: relative;
    background: #000;
}

.video-container {
    position: absolute;
    bottom: 20px;
    right: 20px;
    z-index: 20;
}

video {
    width: 320px;
    height: 240px;
    object-fit: cover;
    transform: scaleX(-1);
    border: 2px solid #00ffff;
    box-shadow: 0 0 15px rgba(0, 255, 255, 0.3), inset 0 0 20px rgba(0,0,0,0.5);
    background: #001122;
    border-radius: 4px;
    display: block;
}

body {
            margin: 0;
            overflow: hidden;
            background-color: #000;
            font-family: 'Courier New', Courier, monospace;
        }

        /* Video: Bottom Right (PIP Style) */
        #input-video {
            position: absolute;
            bottom: 20px;
            right: 20px;
            width: 320px;
            height: 240px;
            object-fit: cover;
            transform: scaleX(-1); /* Mirror */
            z-index: 20;
            border: 2px solid #00ffff;
            box-shadow: 0 0 15px rgba(0, 255, 255, 0.3), inset 0 0 20px rgba(0,0,0,0.5);
            background: #001122;
            border-radius: 4px;
        }

        #video-label {
            position: absolute;
            bottom: 265px;
            right: 20px;
            color: #00ffff;
            font-size: 12px;
            text-shadow: 0 0 5px #00ffff;
            z-index: 21;
            background: rgba(0,0,0,0.7);
            padding: 2px 5px;
        }

        #canvas-container {
            width: 100vw;
            height: 100vh;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
            background: radial-gradient(circle at center, #111 0%, #000 80%);
        }

        /* UI Overlay */
        #ui-layer {
            position: absolute;
            z-index: 10;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            padding: 20px;
            box-sizing: border-box;
        }

        .hud-header {
            text-align: left;
            color: #00ffff;
            text-shadow: 0 0 10px #00ffff;
            pointer-events: auto;
            margin-left: 20px;
            margin-top: 10px;
        }
        
        h1 {
            font-size: 24px;
            margin: 0;
            letter-spacing: 4px;
            text-transform: uppercase;
            border-left: 4px solid #00ffff;
            padding-left: 15px;
            background: linear-gradient(90deg, rgba(0,255,255,0.1), transparent);
        }

        .status-panel {
            position: absolute;
            top: 20px;
            right: 20px;
            border: 1px solid rgba(0, 255, 255, 0.3);
            padding: 15px;
            background: rgba(0, 10, 20, 0.8);
            color: #fff;
            font-size: 12px;
            text-align: right;
            border-right: 4px solid #00ffff;
            width: 250px;
        }

        .status-item {
            margin-bottom: 5px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            border-bottom: 1px solid rgba(0,255,255,0.1);
            padding-bottom: 2px;
        }

        .status-value {
            color: #00ffff;
            font-weight: bold;
        }

        /* Loading */
        #loader {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: #000;
            z-index: 100;
            display: flex;
            justify-content: center;
            align-items: center;
            flex-direction: column;
            color: #00ffff;
            transition: opacity 0.5s;
        }
        
        .loader-text {
            margin-top: 20px;
            animation: blink 1s infinite;
        }

        @keyframes blink {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.5; }
        }

</style>