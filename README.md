<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>ROCKET LEAGUE 2.0 - PRO PHYSICS</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #gui, #menu, #settings-menu { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.95) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.9); padding: 40px; border: 3px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 400px; box-shadow: 0 0 50px rgba(0, 242, 254, 0.4); }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 12px 25px; font-family: 'Orbitron'; cursor: pointer; font-weight: bold; border-radius: 8px; margin: 5px; pointer-events: all; }
        button:hover { background: #00f2fe; color: #000; }
        #hud { position: absolute; inset: 0; pointer-events: none; display: none; }
        #top-bar { position: absolute; top: 20px; left: 50%; transform: translateX(-50%); text-align: center; }
        #scoreboard { font-size: 48px; font-weight: 900; background: rgba(0,0,0,0.5); padding: 10px 30px; border-radius: 10px; border: 1px solid #00f2fe; }
        #settings-icon { position: absolute; bottom: 20px; right: 20px; font-size: 30px; cursor: pointer; pointer-events: all; }
        #settings-menu { display: none; background: rgba(0,0,0,0.85); pointer-events: all; z-index: 100; }
        #goal-flash { position: absolute; inset: 0; background: white; opacity: 0; pointer-events: none; transition: opacity 0.1s; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>
    <div id="goal-flash"></div>
    <div id="menu">
        <div class="panel">
            <h1>ROCKET LEAGUE 2.0</h1>
            <p style="color: #00f2fe;">WASD: Drive/Aerial | SPACE: Jump | SHIFT: Boost | R: Reset</p>
            <button onclick="startGame()">ENTER ARENA</button>
        </div>
    </div>
    <div id="hud">
        <div id="top-bar"><div id="scoreboard"><span id="s-blue" style="color:#00f2fe">0</span> - <span id="s-orange" style="color:#ff8c00">0</span></div></div>
        <div id="settings-icon" onclick="toggleSettings()">⚙️</div>
    </div>
    <div id="settings-menu">
        <div class="panel">
            <h2>BOT SETTINGS</h2>
            <button onclick="botDifficulty = 0.02">EASY</button>
            <button onclick="botDifficulty = 0.09">HARD</button>
            <br><br><button onclick="toggleSettings()">CLOSE</button>
        </div>
    </div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let gameState = 'MENU', botActive = true, botDifficulty = 0.05, score = { blue: 0, orange: 0 }, shake = 0;
        const keys = {};

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -20, 0);

        scene.add(new THREE.GridHelper(400, 80, 0x00f2fe, 0x050515));
        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
        world.addBody(groundBody);

        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 5, 0);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x00f2fe }));
        scene.add(ballMesh);

        function createCar(color, posZ) {
            const body = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(1.6, 1.1, 2.4)) });
            body.position.set(0, 2, posZ);
            body.angularDamping = 0.9; // Prevents wild spinning
            world.addBody(body);
            const mesh = new THREE.Mesh(new THREE.BoxGeometry(3.2, 2.2, 4.8), new THREE.MeshStandardMaterial({ color }));
            scene.add(mesh);
            return { body, mesh };
        }
        const player = createCar(0x00eaff, 30);
        const bot = createCar(0xff8c00, -30);

        scene.add(new THREE.AmbientLight(0xffffff, 0.6));
        const sun = new THREE.DirectionalLight(0xffffff, 1);
        sun.position.set(10, 50, 10);
        scene.add(sun);

        window.startGame = () => { gameState = 'PLAYING'; document.getElementById('menu').style.display = 'none'; document.getElementById('hud').style.display = 'block'; };
        window.toggleSettings = () => { const m = document.getElementById('settings-menu'); m.style.display = m.style.display === 'flex' ? 'none' : 'flex'; };

        window.addEventListener('keydown', e => {
            keys[e.code] = true;
            if(e.code === 'Space' && Math.abs(player.body.velocity.y) < 0.5) {
                player.body.velocity.y = 12; // Jump up
                if(keys['KeyW']) player.body.velocity.z -= 10; // Forward dodge feel
            }
            if(e.code === 'KeyR') resetMatch();
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        function resetMatch() {
            ballBody.position.set(0, 10, 0); ballBody.velocity.set(0,0,0);
            player.body.position.set(0, 2, 30); player.body.velocity.set(0,0,0); player.body.quaternion.set(0,0,0,1);
            bot.body.position.set(0, 2, -30); bot.body.velocity.set(0,0,0);
        }

        function update() {
            requestAnimationFrame(update);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(player.mesh.quaternion);
                const isAirborne = Math.abs(player.body.position.y) > 1.5;

                // Driving & Aerial Pitch
                if(keys['KeyW']) {
                    if(!isAirborne) player.body.velocity.addScaledVector(1.0, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), player.body.velocity);
                    else player.body.angularVelocity.x = -3; // Pitch Down
                }
                if(keys['KeyS']) {
                    if(!isAirborne) player.body.velocity.addScaledVector(-0.6, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), player.body.velocity);
                    else player.body.angularVelocity.x = 3; // Pitch Up
                }
                
                // Steering & Air Roll
                if(keys['KeyA']) {
                    if(!isAirborne) player.body.angularVelocity.y = 4;
                    else player.body.angularVelocity.z = -3; // Roll Left
                }
                if(keys['KeyD']) {
                    if(!isAirborne) player.body.angularVelocity.y = -4;
                    else player.body.angularVelocity.z = 3; // Roll Right
                }

                if(keys['ShiftLeft']) player.body.velocity.addScaledVector(0.7, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), player.body.velocity);

                // Bot AI
                if(botActive) {
                    const b2b = new THREE.Vector3().subVectors(ballMesh.position, bot.mesh.position);
                    bot.body.quaternion.setFromEuler(0, Math.atan2(b2b.x, b2b.z) + Math.PI, 0);
                    const bf = new THREE.Vector3(0,0,-1).applyQuaternion(bot.mesh.quaternion);
                    bot.body.velocity.addScaledVector(botDifficulty * 22, new CANNON.Vec3(bf.x, bf.y, bf.z), bot.body.velocity);
                }

                // Goals
                if(ballBody.position.z < -60) { score.blue++; resetMatch(); }
                if(ballBody.position.z > 60) { score.orange++; resetMatch(); }
                document.getElementById('s-blue').innerText = score.blue;
                document.getElementById('s-orange').innerText = score.orange;

                player.mesh.position.copy(player.body.position); player.mesh.quaternion.copy(player.body.quaternion);
                bot.mesh.position.copy(bot.body.position); ballMesh.position.copy(ballBody.position);
                camera.position.lerp(player.mesh.position.clone().add(new THREE.Vector3(0,8,18).applyQuaternion(player.mesh.quaternion)), 0.1);
                camera.lookAt(ballMesh.position);
            }
            renderer.render(scene, camera);
        }
        update();
    </script>
</body>
</html>
