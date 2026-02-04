<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER - GARAGE</title>
    <style>
        body { margin: 0; overflow: hidden; background: #050510; font-family: 'Orbitron', sans-serif; color: white; }
        #gui, #menu { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        
        /* Menu Styling */
        #menu { background: rgba(5, 5, 20, 0.9); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.8); padding: 30px; border: 2px solid cyan; border-radius: 15px; text-align: center; min-width: 400px; }
        h1 { font-size: 40px; margin-bottom: 20px; text-shadow: 0 0 10px cyan; }
        button { 
            background: transparent; border: 2px solid cyan; color: cyan; padding: 10px 25px; 
            font-family: 'Orbitron'; cursor: pointer; margin: 10px; transition: 0.3s; 
        }
        button:hover { background: cyan; color: black; box-shadow: 0 0 20px cyan; }
        button.active { background: cyan; color: black; }

        /* HUD Styling */
        #hud { position: absolute; inset: 0; pointer-events: none; display: none; }
        #scoreboard { position: absolute; top: 20px; left: 50%; transform: translateX(-50%); font-size: 32px; }
        #boost-container { position: absolute; bottom: 40px; right: 40px; width: 250px; }
        #boost-bar { height: 12px; background: rgba(0,255,255,0.1); border: 1px solid cyan; border-radius: 5px; overflow: hidden; }
        #boost-fill { height: 100%; width: 100%; background: cyan; transition: width 0.1s; }
        
        .setting-row { display: flex; justify-content: space-between; margin: 10px 0; font-size: 14px; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700&display=swap" rel="stylesheet">
</head>
<body>

    <div id="menu">
        <div class="panel" id="garage-panel">
            <h1>CAR GARAGE*</h1>
            <div style="margin-bottom: 20px;">
                <button id="btn-octane" class="active" onclick="selectCar('octane')">OCTANE TYPE</button>
                <button id="btn-dominus" onclick="selectCar('dominus')">DOMINUS TYPE</button>
            </div>
            
            <div id="settings-area" style="border-top: 1px solid #333; padding-top: 20px;">
                <h3>SETTINGS</h3>
                <div class="setting-row"><span>Drive</span> <span>W / S</span></div>
                <div class="setting-row"><span>Steer</span> <span>A / D</span></div>
                <div class="setting-row"><span>Jump</span> <span>K</span></div>
                <div class="setting-row"><span>Boost</span> <span>L-SHIFT</span></div>
                <div class="setting-row"><span>Ball Cam</span> <span>SPACE</span></div>
            </div>

            <button style="margin-top: 30px; font-size: 20px; width: 100%;" onclick="startGame()">START MATCH</button>
        </div>
    </div>

    <div id="hud">
        <div id="scoreboard"><span style="color:cyan">BLUE</span> <span id="s-blue">0</span> | <span id="s-orange">0</span> <span style="color:orange">ORANGE</span></div>
        <div id="boost-container">
            <div style="font-size: 12px; margin-bottom: 5px;">BOOST</div>
            <div id="boost-bar"><div id="boost-fill"></div></div>
        </div>
    </div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "cannon": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/+esm"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import * as CANNON from 'cannon';

        let gameState = 'MENU';
        let selectedCarType = 'octane';
        const keys = {};
        let score = { blue: 0, orange: 0 };
        let boost = 100;
        let ballCam = false;

        // --- SCENE SETUP ---
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -25, 0);

        // --- ARENA ---
        const grid = new THREE.GridHelper(200, 40, 0x00ffff, 0x001111);
        scene.add(grid);
        
        // Physics Floor
        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
        world.addBody(groundBody);

        // --- THE BALL ---
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 10, -20);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x444444 }));
        scene.add(ballMesh);

        // --- CAR LOGIC ---
        let carBody, carMesh;

        function createCar(type) {
            if(carBody) world.removeBody(carBody);
            if(carMesh) scene.remove(carMesh);

            const isOctane = type === 'octane';
            // Hitbox sizes: Octane (shorter/taller) vs Dominus (longer/flatter)
            const dim = isOctane ? {w:1.4, h:0.9, l:2.0} : {w:1.5, h:0.6, l:2.5};
            
            carBody = new CANNON.Body({ mass: 25, shape: new CANNON.Box(new CANNON.Vec3(dim.w, dim.h, dim.l)) });
            carBody.position.set(0, 2, 20);
            world.addBody(carBody);

            carMesh = new THREE.Group();
            const bodyGeo = new THREE.BoxGeometry(dim.w*2, dim.h*2, dim.l*2);
            const bodyMat = new THREE.MeshStandardMaterial({ color: isOctane ? 0x0099ff : 0xff0055, metalness: 0.7, roughness: 0.2 });
            const mainBody = new THREE.Mesh(bodyGeo, bodyMat);
            
            // Simple Octane "spoiler" or Dominus "hood" additions
            const extraGeo = isOctane ? new THREE.BoxGeometry(dim.w*2, 0.5, 1) : new THREE.BoxGeometry(dim.w*1.8, 0.2, 1);
            const extra = new THREE.Mesh(extraGeo, bodyMat);
            extra.position.set(0, dim.h, isOctane ? dim.l - 0.5 : -dim.l + 1);
            
            carMesh.add(mainBody, extra);
            scene.add(carMesh);
        }

        // Initialize with Octane
        createCar('octane');

        // --- LIGHTS ---
        const ambient = new THREE.AmbientLight(0xffffff, 0.4);
        scene.add(ambient);
        const light = new THREE.DirectionalLight(0xffffff, 1);
        light.position.set(10, 20, 10);
        scene.add(light);

        // --- INTERFACE FUNCTIONS ---
        window.selectCar = (type) => {
            selectedCarType = type;
            document.getElementById('btn-octane').classList.toggle('active', type === 'octane');
            document.getElementById('btn-dominus').classList.toggle('active', type === 'dominus');
            createCar(type);
        };

        window.startGame = () => {
            gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        // --- CONTROLS ---
        window.addEventListener('keydown', e => {
            keys[e.code] = true;
            if(e.code === 'Space' && gameState === 'PLAYING') ballCam = !ballCam;
            if(e.code === 'KeyK' && carBody.position.y < 1.5) carBody.velocity.y = 12;
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        // --- GAME LOOP ---
        function animate() {
            requestAnimationFrame(animate);

            if(gameState === 'PLAYING') {
                world.fixedStep();

                // Movement Logic
                const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(carMesh.quaternion);
                if(keys['KeyW']) carBody.velocity.addScaledVector(0.6, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
                if(keys['KeyS']) carBody.velocity.addScaledVector(-0.4, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
                
                if(keys['KeyA']) carBody.angularVelocity.y = 3.0;
                else if(keys['KeyD']) carBody.angularVelocity.y = -3.0;
                else carBody.angularVelocity.y *= 0.9;

                // Boost
                if(keys['ShiftLeft'] && boost > 0) {
                    carBody.velocity.addScaledVector(0.8, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
                    boost -= 0.5;
                } else if(boost < 100) boost += 0.1;

                // Update Visuals
                carMesh.position.copy(carBody.position);
                carMesh.quaternion.copy(carBody.quaternion);
                ballMesh.position.copy(ballBody.position);

                // UI
                document.getElementById('boost-fill').style.width = boost + "%";

                // Camera
                const offset = ballCam ? new THREE.Vector3(0, 8, 15) : new THREE.Vector3(0, 5, 12);
                const idealCam = carMesh.position.clone().add(offset.applyQuaternion(carMesh.quaternion));
                camera.position.lerp(idealCam, 0.1);
                camera.lookAt(ballCam ? ballMesh.position : carMesh.position);

            } else {
                // Menu "Showcase" Rotation
                carMesh.rotation.y += 0.01;
                camera.position.set(10, 5, 10);
                camera.lookAt(carMesh.position);
            }

            renderer.render(scene, camera);
        }

        animate();
    </script>
</body>
</html>
