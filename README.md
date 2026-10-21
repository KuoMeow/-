```html
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>3D 情緒陀螺大戰 - 社會技巧遊戲</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body {
            font-family: 'Inter', 'Noto Sans TC', sans-serif;
            background: linear-gradient(135deg, #fdfbfb 0%, #ebedee 100%);
            margin: 0;
            overflow: hidden; /* Prevent scrolling */
            touch-action: none;
        }
        
        #canvas-container {
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            z-index: 1;
        }

        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            z-index: 10;
            pointer-events: none; /* Let clicks pass through to canvas if needed */
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            padding: 2rem 1rem;
        }

        .pointer-events-auto {
            pointer-events: auto;
        }

        /* Modal animation */
        @keyframes popIn {
            0% { transform: scale(0.8) translateY(20px); opacity: 0; }
            100% { transform: scale(1) translateY(0); opacity: 1; }
        }
        .modal-animate {
            animation: popIn 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
        }

        .glass-panel {
            background: rgba(255, 255, 255, 0.85);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.5);
        }
    </style>
</head>
<body>

    <!-- 3D Canvas Container -->
    <div id="canvas-container"></div>

    <!-- UI Layer -->
    <div id="ui-layer">
        
        <!-- Header -->
        <div class="text-center pointer-events-auto max-w-lg mx-auto glass-panel rounded-3xl p-4 shadow-lg w-full">
            <h1 class="text-2xl md:text-3xl font-extrabold text-indigo-600 drop-shadow-sm mb-1">🌪️ 3D 情緒陀螺大戰 🌪️</h1>
            <p class="text-gray-600 font-medium text-sm md:text-base">按下「開始戰鬥」，看看誰會留到最後！</p>
        </div>

        <!-- Controls (Bottom) -->
        <div class="text-center pointer-events-auto pb-4">
            <button id="startBtn" onclick="startBattle()" class="bg-gradient-to-r from-blue-500 to-indigo-600 hover:from-blue-600 hover:to-indigo-700 text-white font-bold py-4 px-12 rounded-full text-xl transition transform hover:scale-105 active:scale-95 shadow-xl border-4 border-white/30">
                ⚔️ 開始戰鬥！
            </button>
        </div>

    </div>

    <!-- Modal Overlay (Hidden by default, fixed full screen) -->
    <div id="scenarioModal" class="fixed inset-0 z-50 flex items-center justify-center bg-indigo-900/60 backdrop-blur-sm hidden pointer-events-auto">
        <div class="bg-white rounded-3xl p-8 m-4 text-center shadow-2xl max-w-md w-full modal-animate border-4 border-amber-300">
            <div class="text-6xl mb-4" id="modalEmoji">👑</div>
            <h2 class="text-3xl font-extrabold text-amber-500 mb-6 drop-shadow-sm" id="modalTitle">紅蓮火焰 獲勝！</h2>
            
            <div class="bg-indigo-50 rounded-2xl p-6 mb-8 border-2 border-indigo-100 shadow-inner">
                <p class="text-indigo-800 text-sm font-bold mb-2 uppercase tracking-wider">💭 任務挑戰 💭</p>
                <p id="scenarioText" class="text-gray-800 text-xl leading-relaxed font-bold">
                    <!-- Scenario text goes here -->
                </p>
            </div>
            
            <button onclick="resetGame()" class="w-full bg-gradient-to-r from-amber-400 to-orange-500 hover:from-amber-500 hover:to-orange-600 text-white font-bold py-4 px-6 rounded-2xl transition transform hover:scale-105 active:scale-95 shadow-lg text-lg">
                🔄 再玩一次
            </button>
        </div>
    </div>

    <script>
        // --- SEL Scenarios (Traditional Chinese for Kids) ---
        const scenarios = [
            "當你好朋友不讓你加入他們的遊戲時，你心裡有什麼感覺？你會怎麼做？",
            "如果同學不小心弄壞了你的作品，但他馬上道歉了，你會怎麼回應？",
            "在小組討論時，你的意見和大家不一樣，這時候你可以怎麼表達？",
            "你在遊戲中輸了，覺得很難過或生氣，這時候你可以怎麼讓自己平靜下來？",
            "看到同學一個人坐在角落看起來很孤單，你會去跟他說什麼嗎？",
            "有人對你開了一個讓你不舒服的玩笑，你可以用什麼態度告訴他你不喜歡？"
        ];

        // --- Three.js Setup ---
        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        scene.background = new THREE.Color('#e0e7ff'); // Light indigo background
        scene.fog = new THREE.Fog('#e0e7ff', 50, 150);

        // Camera
        const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
        camera.position.set(0, 45, 55);
        camera.lookAt(0, 0, 0);

        // Renderer
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.shadowMap.enabled = true;
        renderer.shadowMap.type = THREE.PCFSoftShadowMap;
        container.appendChild(renderer.domElement);

        // Lighting
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);

        const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
        dirLight.position.set(20, 50, 20);
        dirLight.castShadow = true;
        dirLight.shadow.mapSize.width = 1024;
        dirLight.shadow.mapSize.height = 1024;
        dirLight.shadow.camera.near = 0.5;
        dirLight.shadow.camera.far = 100;
        dirLight.shadow.camera.left = -30;
        dirLight.shadow.camera.right = 30;
        dirLight.shadow.camera.top = 30;
        dirLight.shadow.camera.bottom = -30;
        scene.add(dirLight);

        // --- Arena (Stadium) ---
        const arenaRadius = 25;
        const arenaGroup = new THREE.Group();
        
        // Base plate (Bowl shape simulation via shallow cone/cylinder)
        const baseGeo = new THREE.CylinderGeometry(arenaRadius, arenaRadius - 2, 2, 64);
        const baseMat = new THREE.MeshStandardMaterial({ 
            color: '#f8fafc', 
            roughness: 0.2, 
            metalness: 0.1 
        });
        const baseMesh = new THREE.Mesh(baseGeo, baseMat);
        baseMesh.position.y = -1;
        baseMesh.receiveShadow = true;
        arenaGroup.add(baseMesh);

        // Rim
        const rimGeo = new THREE.TorusGeometry(arenaRadius, 1.5, 16, 64);
        const rimMat = new THREE.MeshStandardMaterial({ 
            color: '#6366f1', 
            roughness: 0.5, 
            metalness: 0.3 
        });
        const rimMesh = new THREE.Mesh(rimGeo, rimMat);
        rimMesh.rotation.x = Math.PI / 2;
        rimMesh.position.y = 0;
        rimMesh.receiveShadow = true;
        rimMesh.castShadow = true;
        arenaGroup.add(rimMesh);

        // Inner lines (decorative)
        const innerRingGeo = new THREE.RingGeometry(10, 10.5, 32);
        const innerRingMat = new THREE.MeshBasicMaterial({ color: '#cbd5e1', side: THREE.DoubleSide });
        const innerRing = new THREE.Mesh(innerRingGeo, innerRingMat);
        innerRing.rotation.x = Math.PI / 2;
        innerRing.position.y = 0.05;
        arenaGroup.add(innerRing);

        scene.add(arenaGroup);

        // --- Tops (Beyblades) Creation ---
        function createTop(primaryColor, secondaryColor, name, startX) {
            const topGroup = new THREE.Group();
            
            // Physics / Game data
            topGroup.userData = {
                name: name,
                vx: 0,
                vz: 0,
                radius: 3.5,
                mass: 1,
                isSpinning: true,
                isKnockedOver: false,
                spinSpeed: 0.5,
                colorName: primaryColor == '#ef4444' ? '紅蓮' : '蒼龍'
            };

            // Materials
            const matPrimary = new THREE.MeshStandardMaterial({ color: primaryColor, roughness: 0.3, metalness: 0.7 });
            const matSecondary = new THREE.MeshStandardMaterial({ color: secondaryColor, roughness: 0.2, metalness: 0.9 });
            const matTip = new THREE.MeshStandardMaterial({ color: '#334155', roughness: 0.8 });

            // 1. Tip (Cone)
            const tipGeo = new THREE.ConeGeometry(0.8, 2, 16);
            const tip = new THREE.Mesh(tipGeo, matTip);
            tip.position.y = 1; // Base of cone is at y=0, tip points up. We need to invert it.
            tip.rotation.x = Math.PI;
            tip.castShadow = true;
            topGroup.add(tip);

            // 2. Middle Body (Cylinder)
            const bodyGeo = new THREE.CylinderGeometry(2.5, 0.8, 1.5, 32);
            const body = new THREE.Mesh(bodyGeo, matSecondary);
            body.position.y = 2.75;
            body.castShadow = true;
            topGroup.add(body);

            // 3. Attack Ring (Torus/Thick Cylinder)
            const ringGeo = new THREE.TorusGeometry(2.5, 1, 16, 32);
            const ring = new THREE.Mesh(ringGeo, matPrimary);
            ring.rotation.x = Math.PI / 2;
            ring.position.y = 3.5;
            ring.castShadow = true;
            topGroup.add(ring);

            // 4. Top Cap
            const capGeo = new THREE.CylinderGeometry(1.5, 2.5, 0.5, 32);
            const cap = new THREE.Mesh(capGeo, matSecondary);
            cap.position.y = 4.25;
            cap.castShadow = true;
            topGroup.add(cap);

            // Set initial position
            topGroup.position.set(startX, 0, 0);
            scene.add(topGroup);
            
            return topGroup;
        }

        // Create two tops
        const top1 = createTop('#ef4444', '#f59e0b', '紅蓮火焰 (紅)', -12); // Red/Gold
        const top2 = createTop('#3b82f6', '#94a3b8', '蒼藍水龍 (藍)', 12);  // Blue/Silver
        const tops = [top1, top2];

        // --- Game Logic ---
        let isBattling = false;
        let battleTimeout;
        const startBtn = document.getElementById('startBtn');
        const modal = document.getElementById('scenarioModal');
        const scenarioText = document.getElementById('scenarioText');
        const modalTitle = document.getElementById('modalTitle');

        function startBattle() {
            if (isBattling) return;
            isBattling = true;
            startBtn.classList.add('hidden');

            // Reset states
            tops.forEach(t => {
                t.userData.isSpinning = true;
                t.userData.isKnockedOver = false;
                t.userData.spinSpeed = 0.4 + Math.random() * 0.2;
                t.rotation.z = 0;
                t.rotation.x = 0;
                t.position.y = 0;
                
                // Random high initial velocity
                t.userData.vx = (Math.random() - 0.5) * 2;
                t.userData.vz = (Math.random() - 0.5) * 2;
            });

            // Random battle duration between 5 to 8 seconds
            const duration = 5000 + Math.random() * 3000;
            battleTimeout = setTimeout(endBattle, duration);
        }

        function endBattle() {
            isBattling = false;
            
            // Randomly pick a loser
            const loserIdx = Math.random() > 0.5 ? 0 : 1;
            const loser = tops[loserIdx];
            const winner = tops[1 - loserIdx];

            // Make loser fly out or get knocked over
            loser.userData.isSpinning = false;
            loser.userData.isKnockedOver = true;
            
            // Give loser a sudden burst to simulate getting hit hard
            loser.userData.vx = (loser.position.x > 0 ? 1 : -1) * 2;
            loser.userData.vz = (loser.position.z > 0 ? 1 : -1) * 2;

            // Winner slows down but stays standing
            winner.userData.vx *= 0.2;
            winner.userData.vz *= 0.2;

            // Show UI after dramatic pause
            setTimeout(() => {
                showScenario(winner.userData.name);
            }, 2000);
        }

        function showScenario(winnerName) {
            const randomScenario = scenarios[Math.floor(Math.random() * scenarios.length)];
            modalTitle.innerText = `🏆 ${winnerName} 獲勝！`;
            scenarioText.innerText = randomScenario;
            modal.classList.remove('hidden');
        }

        function resetGame() {
            modal.classList.add('hidden');
            startBtn.classList.remove('hidden');
            
            // Reset positions
            top1.position.set(-12, 0, 0);
            top2.position.set(12, 0, 0);

            tops.forEach(t => {
                t.userData.vx = 0;
                t.userData.vz = 0;
                t.userData.isSpinning = true;
                t.userData.isKnockedOver = false;
                t.rotation.z = 0;
                t.rotation.x = 0;
                t.position.y = 0;
            });
        }

        // --- Physics & Animation Loop ---
        function updatePhysics() {
            // 1. Update positions based on velocity
            tops.forEach(t => {
                t.position.x += t.userData.vx;
                t.position.z += t.userData.vz;

                // Simulated gravity/slope pulling towards center
                if (!t.userData.isKnockedOver) {
                    t.userData.vx -= t.position.x * 0.003;
                    t.userData.vz -= t.position.z * 0.003;
                    
                    // Friction
                    t.userData.vx *= 0.99;
                    t.userData.vz *= 0.99;
                } else {
                    // High friction if knocked over
                    t.userData.vx *= 0.90;
                    t.userData.vz *= 0.90;
                }

                // Handle Knocked over visual (tilt)
                if (t.userData.isKnockedOver) {
                    // Slowly fall over
                    if (t.rotation.z < Math.PI / 2.2) {
                        t.rotation.z += 0.1;
                    }
                    t.position.y = 1; // lay slightly above ground
                } else {
                    // Wobble slightly based on speed
                    const speed = Math.hypot(t.userData.vx, t.userData.vz);
                    t.rotation.z = Math.sin(Date.now() * 0.01) * (speed * 0.05);
                    t.rotation.x = Math.cos(Date.now() * 0.01) * (speed * 0.05);
                }

                // Spin
                if (t.userData.isSpinning) {
                    t.rotation.y += t.userData.spinSpeed;
                }

                // Arena Boundary (Wall collision)
                const distFromCenter = Math.hypot(t.position.x, t.position.z);
                const maxRadius = arenaRadius - t.userData.radius;
                
                if (distFromCenter > maxRadius) {
                    // Push back
                    const angle = Math.atan2(t.position.z, t.position.x);
                    t.position.x = Math.cos(angle) * maxRadius;
                    t.position.z = Math.sin(angle) * maxRadius;
                    
                    // Bounce normal
                    const normalX = Math.cos(angle);
                    const normalZ = Math.sin(angle);
                    const dot = t.userData.vx * normalX + t.userData.vz * normalZ;
                    
                    t.userData.vx -= 2 * dot * normalX;
                    t.userData.vz -= 2 * dot * normalZ;
                    
                    // Add spark/chaos on wall hit if battling
                    if (isBattling) {
                        t.userData.vx += (Math.random() - 0.5) * 0.5;
                        t.userData.vz += (Math.random() - 0.5) * 0.5;
                    }
                }
            });

            // 2. Check Top vs Top Collision
            if (!top1.userData.isKnockedOver && !top2.userData.isKnockedOver) {
                const dx = top2.position.x - top1.position.x;
                const dz = top2.position.z - top1.position.z;
                const distance = Math.hypot(dx, dz);
                const minDistance = top1.userData.radius + top2.userData.radius;

                if (distance < minDistance) {
                    // Separate
                    const overlap = (minDistance - distance) / 2;
                    const angle = Math.atan2(dz, dx);
                    
                    top1.position.x -= Math.cos(angle) * overlap;
                    top1.position.z -= Math.sin(angle) * overlap;
                    top2.position.x += Math.cos(angle) * overlap;
                    top2.position.z += Math.sin(angle) * overlap;

                    // Elastic Collision
                    const normalX = dx / distance;
                    const normalZ = dz / distance;

                    const relVx = top1.userData.vx - top2.userData.vx;
                    const relVz = top1.userData.vz - top2.userData.vz;

                    const speed = relVx * normalX + relVz * normalZ;

                    if (speed > 0) {
                        const impulse = (2 * speed) / (top1.userData.mass + top2.userData.mass);
                        
                        top1.userData.vx -= impulse * top2.userData.mass * normalX;
                        top1.userData.vz -= impulse * top2.userData.mass * normalZ;
                        top2.userData.vx += impulse * top1.userData.mass * normalX;
                        top2.userData.vz += impulse * top1.userData.mass * normalZ;

                        // Add randomness (attack power)
                        if (isBattling) {
                            const power = 0.5;
                            top1.userData.vx += (Math.random() - 0.5) * power;
                            top1.userData.vz += (Math.random() - 0.5) * power;
                            top2.userData.vx += (Math.random() - 0.5) * power;
                            top2.userData.vz += (Math.random() - 0.5) * power;
                        }
                    }
                }
            }
        }

        // Animation Loop
        function animate() {
            requestAnimationFrame(animate);
            updatePhysics();
            renderer.render(scene, camera);
        }

        // Handle Window Resize
        window.addEventListener('resize', onWindowResize, false);
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        // Start loop
        window.onload = function() {
            animate();
        };

    </script>
</body>
</html>
```
