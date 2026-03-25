<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>B2 Stealth Simulator - Pro</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Arial Black', sans-serif; touch-action: none; }
       
        /* UI 레이어 */
        #ui {
            position: absolute; top: 20px; left: 20px; color: white;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.8); pointer-events: none;
            z-index: 10; display: none;
        }
        .stat { font-size: 24px; font-weight: bold; margin-bottom: 5px; }

        /* 시작 화면 */
        #overlay {
            position: absolute; inset: 0; background: rgba(0,0,0,0.7);
            display: flex; flex-direction: column; justify-content: center; align-items: center;
            z-index: 100; color: white; text-align: center;
        }
        #start-btn {
            background: #ffcc00; color: #000; border: none; padding: 20px 50px;
            font-size: 30px; font-weight: bold; border-radius: 50px; cursor: pointer;
            transition: transform 0.2s; box-shadow: 0 0 20px rgba(255,204,0,0.5);
        }
        #start-btn:hover { transform: scale(1.1); }

        /* 레벨업 알림 */
        #level-up-msg {
            position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%) scale(0);
            color: #00f2ff; font-size: 80px; font-weight: 900; text-shadow: 0 0 20px #00f2ff;
            z-index: 50; pointer-events: none; transition: transform 0.5s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            text-align: center;
        }
        #level-up-msg.show { transform: translate(-50%, -50%) scale(1); }

        /* 게임 오버 */
        #game-over {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.9); color: white; padding: 40px;
            border-radius: 20px; text-align: center; display: none; z-index: 100;
            border: 2px solid #ff4444; width: 300px;
        }
        #game-over h1 { color: #ff4444; margin-bottom: 10px; }
        #game-over button {
            background: #444; color: white; border: none; padding: 15px 30px;
            font-size: 18px; border-radius: 10px; cursor: pointer; margin-top: 20px;
        }
    </style>
</head>
<body>

    <!-- 시작 오버레이 -->
    <div id="overlay">
        <h1 style="font-size: 48px; margin-bottom: 20px;">B2 STEALTH MISSION</h1>
        <p style="margin-bottom: 30px; font-family: sans-serif;">조작: 화면 드래그 | 목표: 10개 링 통과하여 레벨업</p>
        <button id="start-btn">게임 시작</button>
    </div>

    <!-- 인게임 UI -->
    <div id="ui">
        <div class="stat">LEVEL: <span id="level">1</span></div>
        <div class="stat">RINGS: <span id="rings">0</span> / 10</div>
        <div class="stat">DISTANCE: <span id="distance">0</span>m</div>
    </div>

    <!-- 레벨업 메시지 -->
    <div id="level-up-msg">LEVEL UP!<br><span id="next-level" style="font-size: 40px;"></span></div>

    <!-- 게임 오버 -->
    <div id="game-over">
        <h1>MISSION FAILED</h1>
        <p id="final-stats"></p>
        <button ON-CLICK="location.reload()">재도전</button>
    </div>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script>
        let scene, camera, renderer, player, clouds = [], rings = [], missiles = [];
        let level = 1;
        let ringsPassed = 0;
        let distance = 0;
        let isStarted = false;
        let isGameOver = false;
        let targetX = 0, targetY = 0;
        let clock = new THREE.Clock();

        const SPEED = 0.9;
        const RING_SIZE = 10;

        init();

        function init() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x87ceeb);
            scene.fog = new THREE.Fog(0x87ceeb, 100, 600);

            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 8, 25);

            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);

            const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
            scene.add(ambientLight);
            const sunLight = new THREE.DirectionalLight(0xffffff, 1);
            sunLight.position.set(50, 100, 50);
            scene.add(sunLight);

            // 지면
            const groundGeo = new THREE.PlaneGeometry(3000, 3000, 40, 40);
            const groundMat = new THREE.MeshPhongMaterial({ color: 0x4d6d33, flatShading: true });
            const ground = new THREE.Mesh(groundGeo, groundMat);
            ground.rotation.x = -Math.PI / 2;
            ground.position.y = -40;
            scene.add(ground);

            createPlayer();

            // 리스너
            window.addEventListener('resize', onWindowResize, false);
            document.addEventListener('mousemove', onInputMove);
            document.addEventListener('touchmove', (e) => onInputMove(e.touches[0]), { passive: false });
            document.getElementById('start-btn').addEventListener('click', startGame);

            // 초기 환경
            for(let i=0; i<20; i++) spawnCloud(Math.random() * -1000);
            spawnRing(-150);
            spawnRing(-400);

            animate();
        }

        function startGame() {
            document.getElementById('overlay').style.display = 'none';
            document.getElementById('ui').style.display = 'block';
            isStarted = true;
            clock.start();
        }

        function createPlayer() {
            player = new THREE.Group();
            const shape = new THREE.Shape();
            shape.moveTo(0, 0);
            shape.lineTo(12, 0);
            shape.lineTo(3, 5);
            shape.lineTo(0, 4.5);
            shape.lineTo(-3, 5);
            shape.lineTo(-12, 0);
            shape.lineTo(0, 0);

            const extrudeSettings = { depth: 0.8, bevelEnabled: true, bevelThickness: 0.3, bevelSize: 0.3 };
            const wingGeo = new THREE.ExtrudeGeometry(shape, extrudeSettings);
            const wingMat = new THREE.MeshPhongMaterial({ color: 0x1a1a1a });
            const wing = new THREE.Mesh(wingGeo, wingMat);
            wing.rotation.x = Math.PI / 2;
            player.add(wing);

            const cockpitGeo = new THREE.BoxGeometry(2, 1, 3);
            const cockpit = new THREE.Mesh(cockpitGeo, wingMat);
            cockpit.position.set(0, 0.5, 2.5);
            player.add(cockpit);

            scene.add(player);
        }

        function spawnCloud(zPos = -800) {
            const geo = new THREE.SphereGeometry(Math.random() * 15 + 10, 8, 8);
            const mat = new THREE.MeshPhongMaterial({ color: 0xffffff, transparent: true, opacity: 0.5 });
            const cloud = new THREE.Mesh(geo, mat);
            cloud.position.set((Math.random() - 0.5) * 800, Math.random() * 100 + 40, zPos);
            scene.add(cloud);
            clouds.push(cloud);
        }

        function spawnRing(zPos = -450) {
            const ringGeo = new THREE.TorusGeometry(RING_SIZE, 1.2, 16, 40);
            const ringMat = new THREE.MeshPhongMaterial({ color: 0xffcc00, emissive: 0xffaa00, emissiveIntensity: 0.3 });
            const ring = new THREE.Mesh(ringGeo, ringMat);
            ring.position.set((Math.random() - 0.5) * 90, (Math.random() - 0.5) * 50 + 10, zPos);
            ring.userData.passed = false; // 고정된 상태 유지를 위해 별도 회전 없음
            scene.add(ring);
            rings.push(ring);
        }

        function showLevelUp() {
            const msg = document.getElementById('level-up-msg');
            document.getElementById('next-level').innerText = `LEVEL ${level}`;
            msg.classList.add('show');
            setTimeout(() => msg.classList.remove('show'), 2000);
        }

        function spawnMissile() {
            const group = new THREE.Group();
            const bodyGeo = new THREE.CylinderGeometry(1.2, 1.2, 8, 8);
            const mat = new THREE.MeshPhongMaterial({ color: 0xff3300 });
            const body = new THREE.Mesh(bodyGeo, mat);
            body.rotation.x = Math.PI / 2;
            group.add(body);

            const tipGeo = new THREE.ConeGeometry(1.2, 2.5, 8);
            const tip = new THREE.Mesh(tipGeo, mat);
            tip.rotation.x = Math.PI / 2; tip.position.z = 5;
            group.add(tip);

            group.position.set((Math.random() - 0.5) * 150, (Math.random() - 0.5) * 120, player.position.z - 600);
            scene.add(group);
            missiles.push({ mesh: group, speed: 0.4 + (level * 0.25) });
        }

        function onInputMove(e) {
            if (!isStarted || isGameOver) return;
            targetX = ((e.clientX / window.innerWidth) * 2 - 1) * 70;
            targetY = -((e.clientY / window.innerHeight) * 2 - 1) * 50;
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function gameOver() {
            isGameOver = true;
            document.getElementById('game-over').style.display = 'block';
            document.getElementById('final-stats').innerText = `LEVEL: ${level} | DISTANCE: ${Math.floor(distance)}m`;
        }

        function animate() {
            requestAnimationFrame(animate);
            if (!isStarted || isGameOver) {
                renderer.render(scene, camera);
                return;
            }

            distance += SPEED;
            document.getElementById('distance').innerText = Math.floor(distance);

            // 플레이어 조작
            player.position.x += (targetX - player.position.x) * 0.08;
            player.position.y += (targetY - player.position.y) * 0.08;
            player.rotation.z = (player.position.x - targetX) * 0.03;
            player.rotation.x = (targetY - player.position.y) * 0.03;

            // 카메라
            camera.position.x += (player.position.x - camera.position.x) * 0.03;
            camera.position.y += (player.position.y + 8 - camera.position.y) * 0.03;
            camera.lookAt(player.position.x, player.position.y + 2, player.position.z - 25);

            // 배경
            clouds.forEach(c => {
                c.position.z += SPEED * 3;
                if (c.position.z > camera.position.z + 50) {
                    c.position.z = player.position.z - 800;
                    c.position.x = (Math.random() - 0.5) * 800;
                }
            });

            // 링 (고정 및 색상 변경)
            rings.forEach((ring, index) => {
                ring.position.z += SPEED * 2.5;

                if (!ring.userData.passed) {
                    const dist = player.position.distanceTo(ring.position);
                    if (dist < RING_SIZE + 2) {
                        ring.userData.passed = true;
                        ring.material.color.set(0x00f2ff); // 파란색으로 변경
                        ring.material.emissive.set(0x00aaff);
                       
                        ringsPassed++;
                        if (ringsPassed >= 10) {
                            level++;
                            ringsPassed = 0;
                            document.getElementById('level').innerText = level;
                            showLevelUp();
                        }
                        document.getElementById('rings').innerText = ringsPassed;
                    }
                }

                if (ring.position.z > camera.position.z + 20) {
                    scene.remove(ring);
                    rings.splice(index, 1);
                    spawnRing(player.position.z - 500);
                }
            });

            // 미사일
            if (Math.random() < 0.01 + (level * 0.008)) {
                spawnMissile();
            }

            missiles.forEach((m, index) => {
                m.mesh.position.z += (SPEED * 2.5) + m.speed;
                m.mesh.position.x += (player.position.x - m.mesh.position.x) * 0.005;
                m.mesh.position.y += (player.position.y - m.mesh.position.y) * 0.005;

                const dist = player.position.distanceTo(m.mesh.position);
                if (dist < 4.5) gameOver();

                if (m.mesh.position.z > camera.position.z + 20) {
                    scene.remove(m.mesh);
                    missiles.splice(index, 1);
                }
            });

            renderer.render(scene, camera);
        }
    </script>
</body>
</html>
