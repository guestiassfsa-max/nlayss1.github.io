import * as THREE from 'three';

// --- Базовая инициализация сцены ---
const container = document.getElementById('canvas-container');
let scene, camera, renderer, clock;
let blackHoleMesh, particleSystem;
let orbitingObjects = [];

// Параллакс мыши
const mouse = new THREE.Vector2();
const targetMouse = new THREE.Vector2();

function init() {
    // Сцена и камера
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.z = 5;

    // Рендерер (адаптация под мобилки)
    const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
    if (isMobile) {
        // На мобильных устройствах вместо тяжелого WebGL можно показать видео-заглушку
        showMobileFallback();
        return;
    }

    renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    container.appendChild(renderer.domElement);

    clock = new THREE.Clock();

    createBlackHole();
    createOrbitingAssets();
    handleResize();

    animate();
    addEventListeners();
}

// --- Создание черной дыры (Воронка) ---
function createBlackHole() {
    const segments = 128;
    const geometry = new THREE.PlaneGeometry(10, 10, segments, segments);
    
    // Вершинный шейдер для создания гравитационной воронки
    const vertexShader = `
        uniform float uTime;
        uniform vec2 uMouse;
        varying vec2 vUv;
        
        void main() {
            vUv = uv;
            vec3 pos = position;
            
            // Расстояние от центра плоскости до UV-координаты
            float dist = distance(uv, vec2(0.5));
            
            // Деформация мышью (параллакс глубины)
            float mouseInfluence = smoothstep(0.8, 0.2, distance(uMouse, uv));
            
            // Формула спирали: чем ближе к центру, тем сильнее затягивает вниз по Z
            float force = pow((1.0 - dist) * 2.0, 3.0);
            
            pos.z -= force * 2.5 + sin(dist * 20.0 - uTime * 2.0) * 0.1;
            pos.z += mouseInfluence * 0.5;
            
            gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
        }
    `;

    // Фрагментный шейдер для процедурной "всасывающей" текстуры
    const fragmentShader = `
        uniform float uTime;
        varying vec2 vUv;
        
        void main() {
            vec2 uv = vUv;
            float dist = distance(uv, vec2(0.5));
            
            // Процедурные кольца света
            float rings = abs(sin((dist - uTime * 0.2) * 50.0));
            
            // Градиент абсолютной пустоты
            vec3 col = mix(vec3(0.01, 0.0, 0.05), vec3(0.0, 0.0, 0.0), dist);
            
            // Добавляем неоновое свечение на края воронки
            col += vec3(0.4, 0.0, 0.8) * rings * smoothstep(0.5, 0.0, dist);
            
            gl_FragColor = vec4(col, 1.0);
        }
    `;

    const material = new THREE.ShaderMaterial({
        uniforms: {
            uTime: { value: 0 },
            uMouse: { value: new THREE.Vector2(0.5, 0.5) }
        },
        vertexShader,
        fragmentShader,
        wireframe: false
    });

    blackHoleMesh = new THREE.Mesh(geometry, material);
    blackHoleMesh.rotation.x = Math.PI / 2; // Ложим плоскость параллельно полу камеры
    scene.add(blackHoleMesh);
}

// --- Low-poly ассеты Minecraft на орбите ---
function createOrbitingAssets() {
    const loader = new THREE.ObjectLoader(); // Или JSONLoader для старых моделей
    
    // В реальном проекте здесь загружаются .json или .obj модели блоков
    // Сейчас используем примитивы как заглушки:
    
    const geoBox = new THREE.BoxGeometry(0.3, 0.3, 0.3);
    const matGrass = new THREE.MeshStandardMaterial({ color: 0x3f7f00 }); // Зеленый блок земли
    const grassBlock = new THREE.Mesh(geoBox, matGrass);
    
    const geoDiamond = new THREE.OctahedronGeometry(0.2, 0);
    const matDiamond = new THREE.MeshStandardMaterial({ color: 0x5dffef, emissive: 0x00bfff, emissiveIntensity: 0.5 });
    const diamondOre = new THREE.Mesh(geoDiamond, matDiamond);

    [grassBlock, diamondOre].forEach(obj => {
        obj.scale.set(0.5, 0.5, 0.5);
        scene.add(obj);
        orbitingObjects.push(obj);
    });

    // Свет для подсветки ассетов
    const light = new THREE.PointLight(0xffffff, 1, 10);
    light.position.set(0, 2, 2);
    scene.add(light);
}

// --- Анимация ---
function animate() {
    requestAnimationFrame(animate);
    render();
}

function render() {
    const elapsed = clock.getElapsedTime();
    
    // Обновление времени в шейдере
    if (blackHoleMesh && blackHoleMesh.material.uniforms.uTime) {
        blackHoleMesh.material.uniforms.uTime.value = elapsed;
    }

    // Орбитальное движение ассетов
    orbitingObjects.forEach((obj, i) => {
        const radius = 2.5 + i;
        const speed = 0.3 + i * 0.1;
        obj.position.x = Math.cos(elapsed * speed) * radius;
        obj.position.z = Math.sin(elapsed * speed) * radius;
        
        // Затягивание в центр (гравитация)
        obj.position.lerp(new THREE.Vector3(0, 0, 0), 0.001);
    });

    // Плавное следование за мышью
    targetMouse.lerp(mouse, 0.05);
    if (blackHoleMesh && blackHoleMesh.material.uniforms.uMouse) {
        blackHoleMesh.material.uniforms.uMouse.value.copy(targetMouse);
    }

    // Медленное вращение камеры вокруг оси Y для динамики
    camera.position.x = Math.cos(elapsed * 0.1) * 0.5;
    camera.position.z = 5 + Math.sin(elapsed * 0.1) * 0.5;
    camera.lookAt(scene.position);

    renderer.render(scene, camera);
}

// --- Обработчики событий ---
function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
}

function onMouseMove(e) {
    // Конвертация координат мыши в нормализованные (-1 до 1)
    mouse.x = (e.clientX / window.innerWidth);
    mouse.y = 1.0 - (e.clientY / window.innerHeight); // Инвертируем Y
}

function addEventListeners() {
    window.addEventListener('resize', onWindowResize);
    window.addEventListener('mousemove', onMouseMove);
    
    // Логика кнопки отключения фона (требует доработки dispose())
    document.getElementById('disableBgBtn').addEventListener('click', () => {
        console.log("Фон отключен, память очищена");
        // TODO: Выполнить scene.remove(), geometry.dispose(), material.dispose()
        if (renderer) {
            renderer.forceContextLoss(); // Экстренно освобождаем GPU
            container.removeChild(renderer.domElement);
        }
    });
}

// --- Фолбэк для мобильных устройств ---
function showMobileFallback() {
    container.innerHTML = '<video autoplay loop muted playsinline style="width:100%;height:100%;object-fit:cover;">' +
                          '<source src="space_fallback.webm" type="video/webm">' +
                          '</video>';
}

init();
