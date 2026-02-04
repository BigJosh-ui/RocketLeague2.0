# RocketLeague
Rocket league made with gemini*
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER PRO</title>
    <style>
        body { margin: 0; overflow: hidden; background: #050510; font-family: 'Orbitron', sans-serif; }
        #gui { position: absolute; inset: 0; pointer-events: none; }
        #scoreboard { position: absolute; top: 20px; left: 50%; transform: translateX(-50%); color: #fff; font-size: 32px; text-shadow: 0 0 15px cyan; }
        #boost-container { position: absolute; bottom: 40px; right: 40px; width: 250px; }
        #boost-bar { height: 15px; background: rgba(0,255,255,0.2); border: 2px solid cyan; border-radius: 10px; overflow: hidden; }
        #boost-fill { height: 100%; width: 100%; background: linear-gradient(90deg, #00f2fe 0%, #4facfe 100%); transition: width 0.1s; box-shadow: 0 0 20px cyan; }
        #ball-cam-indicator { position: absolute; bottom: 40px; left: 40px; color: #ff00ff; border: 2px solid #ff00ff; padding: 10px; font-size: 14px; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@700&display=swap" rel="stylesheet">
</head>
<body>
    <div id="gui">
        <div id="scoreboard"><span style="color:cyan">BLUE</span> <span id="s-blue">0</span> | <span id="s-orange">0</span> <span style="color:orange">ORANGE</span></div>
        <div id="ball-cam-indicator">BALL CAM: OFF (SPACE)</div>
        <div id="boost-container">
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

        // --- CONSTANTS ---
        const ARENA = { w: 60, h: 35, l: 100 };
        let score = { blue: 0, orange: 0 };
        let ballCam = false;
        let boost = 100;
        const keys = {};

        // --- SETUP ---
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -25, 0);

        // --- LIGHTING ---
        scene.add(new THREE.AmbientLight(0x4040ff, 0.5));
        const sun = new THREE.DirectionalLight(0xffaa00, 1.2);
        sun.position.set(50, 50, 20);
        scene.add(sun);

        // --- ARENA & WALLS ---
        const arenaGeo = new THREE.BoxGeometry(ARENA.w, ARENA.h, ARENA.l);
        const arenaMat = new THREE.MeshStandardMaterial({ color: 0x111122, side: THREE.BackSide, wireframe: true });
        const arenaMesh = new THREE.Mesh(arenaGeo, arenaMat);
        arenaMesh.position.y = ARENA.h / 2;
        scene.add(arenaMesh);

        function addWall(w, h, l, x, y, z) {
            const shape = new CANNON.Box(new CANNON.Vec3(w/2, h/2, l/2));
            const body = new CANNON.Body({ mass: 0, shape });
            body.position.set(x, y, z);
            world.addBody(body);
        }
        addWall(ARENA.w, 1, ARENA.l, 0, -0.5, 0); // Floor
        addWall(ARENA.w, ARENA.h, 1, 0, ARENA.h/2, ARENA.l/2); // Wall North
        addWall(ARENA.w, ARENA.h, 1, 0, ARENA.h/2, -ARENA.l/2); // Wall South
        addWall(1, ARENA.h, ARENA.l, ARENA.w/2, ARENA.h/2, 0); // Wall East
        addWall(1, ARENA.h, ARENA.l, -ARENA.w/2, ARENA.h/2, 0); // Wall West

        // --- THE BALL ---
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 10, 0);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x222222 }));
        scene.add(ballMesh);

        // --- THE CAR ---
        const carBody = new CANNON.Body({ mass: 25, shape: new CANNON.Box(new CANNON.Vec3(1.5, 0.6, 2.2)) });
        carBody.position.set(0, 2, 40);
        world.addBody(carBody);
        const carMesh = new THREE.Group();
        carMesh.add(new THREE.Mesh(new THREE.BoxGeometry(3, 1.2, 4.5), new THREE.MeshStandardMaterial({ color: 0x00ffff, metalness: 0.8 })));
        scene.add(carMesh);

        // Raycaster for Wall-Stick
        const raycaster = new THREE.Raycaster();
        const downVec = new THREE.Vector3(0, -1, 0);

        // --- CONTROLS ---
        window.addEventListener('keydown', e => { 
            keys[e.code] = true;
            if(e.code === 'Space') ballCam = !ballCam;
            if(e.code === 'KeyK' && carBody.position.y < 5) carBody.velocity.y = 15; // Jump
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        function resetBall() {
            ballBody.position.set(0, 10, 0);
            ballBody.velocity.set(0, 0, 0);
            ballBody.angularVelocity.set(0, 0, 0);
        }

        // --- GAME LOOP ---
        function update() {
            world.fixedStep();

            // 1. Wall-Stick / Gravity Logic
            raycaster.set(carMesh.position, downVec.clone().applyQuaternion(carMesh.quaternion));
            const intersects = raycaster.intersectObject(arenaMesh);
            if(intersects.length > 0 && intersects[0].distance < 3) {
                // Apply "fake" down-force toward the surface
                const forceDir = intersects[0].face.normal.clone().negate();
                carBody.applyForce(new CANNON.Vec3(forceDir.x * 500, forceDir.y * 500, forceDir.z * 500), carBody.position);
            }

            // 2. Movement
            const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(carMesh.quaternion);
            if(keys['KeyW']) carBody.velocity.addScaledVector(0.6, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
            if(keys['KeyS']) carBody.velocity.addScaledVector(-0.4, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
            if(keys['KeyA']) carBody.angularVelocity.y = 3.5;
            else if(keys['KeyD']) carBody.angularVelocity.y = -3.5;
            else carBody.angularVelocity.y *= 0.9;

            // 3. Boost
            if(keys['ShiftLeft'] && boost > 0) {
                carBody.velocity.addScaledVector(1.0, new CANNON.Vec3(forward.x, forward.y, forward.z), carBody.velocity);
                boost -= 0.5;
            } else if(boost < 100) boost += 0.1;

            // 4. Goal Detection
            if(ballBody.position.z > ARENA.l/2 - 2) { score.orange++; resetBall(); }
            if(ballBody.position.z < -ARENA.l/2 + 2) { score.blue++; resetBall(); }

            // Sync
            carMesh.position.copy(carBody.position);
            carMesh.quaternion.copy(carBody.quaternion);
            ballMesh.position.copy(ballBody.position);
            
            // Camera
            if(ballCam) {
                const offset = new THREE.Vector3(0, 6, 14).applyQuaternion(carMesh.quaternion);
                camera.position.lerp(carMesh.position.clone().add(offset), 0.15);
                camera.lookAt(ballMesh.position);
            } else {
                const offset = new THREE.Vector3(0, 5, 12).applyQuaternion(carMesh.quaternion);
                camera.position.lerp(carMesh.position.clone().add(offset), 0.15);
                camera.lookAt(carMesh.position.clone().add(forward.multiplyScalar(5)));
            }

            // UI
            document.getElementById('s-blue').innerText = score.blue;
            document.getElementById('s-orange').innerText = score.orange;
            document.getElementById('boost-fill').style.width = boost + "%";
            document.getElementById('ball-cam-indicator').innerText = ballCam ? "BALL CAM: ON" : "BALL CAM: OFF";

            renderer.render(scene, camera);
            requestAnimationFrame(update);
        }

        update();
    </script>
</body>
</html>
