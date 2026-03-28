# magic-particles
<style>
  body { margin: 0; background: #000; overflow: hidden; }
  #video-input { 
    position: absolute; bottom: 20px; right: 20px; 
    width: 120px; opacity: 0.15; transform: scaleX(-1); 
    border-radius: 8px; filter: grayscale(1); pointer-events: none;
  }
</style>

<div id="container"></div>
<video id="video-input"></video>

<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<script>
/**
 * 漫天虹雨版：爱心唤醒，五彩粒子从天而降
 */
let scene, camera, renderer, particles, geometry;
const particleCount = 5000;
let targetPositions = new Float32Array(particleCount * 3);
const videoElement = document.getElementById('video-input');

// 彩虹雨系统
let rainDrops = []; 
const rainGroup = new THREE.Group();

const samplerCanvas = document.createElement('canvas');
const sCtx = samplerCanvas.getContext('2d');
samplerCanvas.width = 120; samplerCanvas.height = 120;

function getPointsFromShape(type) {
  sCtx.clearRect(0, 0, 120, 120);
  sCtx.fillStyle = "white";
  sCtx.textAlign = "center";
  if (type === 'heart') {
    sCtx.font = "80px Arial";
    sCtx.fillText("❤", 60, 85);
  } else if (['1','2','3'].includes(type)) {
    sCtx.font = "bold 90px Arial";
    sCtx.fillText(type, 60, 90);
  } else return null;

  const imgData = sCtx.getImageData(0, 0, 120, 120).data;
  const points = [];
  for (let y = 0; y < 120; y++) {
    for (let x = 0; x < 120; x++) {
      if (imgData[(y * 120 + x) * 4] > 128) {
        points.push({x: (x - 60) * 0.18, y: (60 - y) * 0.18});
      }
    }
  }
  return points;
}

function init() {
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 12;

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.getElementById('container').appendChild(renderer.domElement);
  
  scene.add(rainGroup);

  geometry = new THREE.BufferGeometry();
  const initialPos = new Float32Array(particleCount * 3);
  for(let i=0; i<particleCount; i++) {
    initialPos[i*3] = (Math.random()-0.5)*35;
    initialPos[i*3+1] = (Math.random()-0.5)*35;
    initialPos[i*3+2] = (Math.random()-0.5)*10;
    targetPositions[i*3] = initialPos[i*3];
    targetPositions[i*3+1] = initialPos[i*3+1];
    targetPositions[i*3+2] = initialPos[i*3+2];
  }
  geometry.setAttribute('position', new THREE.BufferAttribute(initialPos, 3));

  const material = new THREE.PointsMaterial({
    size: 0.07, color: 0xffffff, transparent: true,
    blending: THREE.AdditiveBlending, depthWrite: false
  });

  particles = new THREE.Points(geometry, material);
  scene.add(particles);
}

// 核心：在屏幕上方生成落下的彩虹粒子
function spawnRainbowRain() {
  const count = 3; // 每帧生成的粒子数，数值越大雨越密
  for (let i = 0; i < count; i++) {
    const geo = new THREE.SphereGeometry(0.08, 6, 6);
    // 生成五彩斑斓的颜色
    const mat = new THREE.MeshBasicMaterial({
      color: new THREE.Color(`hsl(${Math.random() * 360}, 100%, 65%)`),
      transparent: true,
      blending: THREE.AdditiveBlending,
      opacity: 0.8
    });
    const p = new THREE.Mesh(geo, mat);
    
    // 从屏幕上方随机位置出发
    p.position.set(
      (Math.random() - 0.5) * 25, // X轴随机
      15,                         // Y轴固定在上方
      (Math.random() - 0.5) * 10  // Z轴随机
    );
    
    // 缓慢落下的速度
    p.userData.speed = Math.random() * 0.05 + 0.03;
    p.userData.drift = (Math.random() - 0.5) * 0.02; // 轻微的左右晃动
    p.userData.life = 1.0;

    rainGroup.add(p);
    rainDrops.push(p);
  }
}

function morphTo(shapeType) {
  const points = getPointsFromShape(shapeType);
  for (let i = 0; i < particleCount; i++) {
    if (points && points.length > 0) {
      const p = points[i % points.length];
      targetPositions[i * 3] = p.x + (Math.random()-0.5)*0.15;
      targetPositions[i * 3 + 1] = p.y + (Math.random()-0.5)*0.15;
      targetPositions[i * 3 + 2] = (Math.random()-0.5)*0.4;
    } else {
      targetPositions[i * 3] = (Math.random()-0.5)*30;
      targetPositions[i * 3 + 1] = (Math.random()-0.5)*25;
      targetPositions[i * 3 + 2] = (Math.random()-0.5)*15;
    }
  }
}

let currentShape = "idle", stateCounter = 0, pendingShape = "";

function onResults(results) {
  if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
    const marks = results.multiHandLandmarks[0];
    const tx = (0.5 - marks[9].x) * 18, ty = (0.5 - marks[9].y) * 14;
    particles.position.x += (tx - particles.position.x) * 0.2;
    particles.position.y += (ty - particles.position.y) * 0.2;

    const getDist = (p1, p2) => Math.sqrt((p1.x-p2.x)**2 + (p1.y-p2.y)**2);
    const baseDist = getDist(marks[0], marks[9]);
    let fUp = 0;
    [8, 12, 16, 20].forEach(tip => { if (getDist(marks[tip], marks[0]) > baseDist * 1.3) fUp++; });
    if (getDist(marks[4], marks[0]) > baseDist * 1.1) fUp++;

    let detected = (fUp === 0) ? "reset" : (fUp >= 4 ? "heart" : fUp.toString());

    if (detected === pendingShape) {
      stateCounter++;
      if (stateCounter > 4 && detected !== currentShape) {
        currentShape = detected;
        if(detected === "heart") {
          morphTo("heart");
          particles.material.color.set(0xff3366);
        } else if (detected === "reset") {
          morphTo("idle");
          particles.material.color.set(0x555555);
        } else {
          morphTo(detected);
          particles.material.color.set(0x00ffdd);
        }
      }
    } else { pendingShape = detected; stateCounter = 0; }
  } else if (currentShape !== "idle") {
    currentShape = "idle"; morphTo("idle"); particles.material.color.set(0x333333);
  }
}

init();
const hands = new Hands({ locateFile: (f) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.8, minTrackingConfidence: 0.8 });
hands.onResults(onResults);
const cameraUtils = new Camera(videoElement, { onFrame: async () => { await hands.send({ image: videoElement }); }, width: 640, height: 480 });
cameraUtils.start();

function animate() {
  requestAnimationFrame(animate);
  const posArr = geometry.attributes.position.array;
  for (let i = 0; i < particleCount * 3; i++) {
    posArr[i] += (targetPositions[i] - posArr[i]) * 0.12;
  }
  geometry.attributes.position.needsUpdate = true;
  particles.rotation.y += 0.003;

  // --- 触发彩虹雨 ---
  if (currentShape === "heart") {
    spawnRainbowRain();
  }

  // 更新雨滴逻辑
  for (let i = rainDrops.length - 1; i >= 0; i--) {
    const r = rainDrops[i];
    r.position.y -= r.userData.speed;     // 缓慢向下落
    r.position.x += r.userData.drift;     // 随风摆动
    
    // 如果掉出屏幕下方，则销毁
    if (r.position.y < -15) {
      rainGroup.remove(r);
      rainDrops.splice(i, 1);
    }
  }

  renderer.render(scene, camera);
}
animate();
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
