<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NEON STRIKER - GARAGE</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Orbitron', sans-serif; color: white; }
        #gui, #menu { position: absolute; inset: 0; display: flex; flex-direction: column; justify-content: center; align-items: center; pointer-events: none; }
        #menu { background: radial-gradient(circle, rgba(10,10,30,0.9) 0%, rgba(0,0,0,1) 100%); pointer-events: all; z-index: 10; }
        .panel { background: rgba(20, 20, 40, 0.85); padding: 40px; border: 2px solid #00f2fe; border-radius: 20px; text-align: center; min-width: 450px; box-shadow: 0 0 40px rgba(0, 242, 254, 0.3); }
        h1 { font-size: 42px; margin-bottom: 30px; text-transform: uppercase; letter-spacing: 5px; color: #00f2fe; text-shadow: 0 0 15px #00f2fe; }
        .btn-group { display: flex; gap: 20px; margin-bottom: 30px; justify-content: center; }
        button { background: transparent; border: 2px solid #00f2fe; color: #00f2fe; padding: 12px 30px; font-family: 'Orbitron'; cursor: pointer; transition: 0.3s; font-weight: bold; border-radius: 5px; }
        button:hover { background: #00f2fe; color: #000; box-shadow: 0 0 25px #00f2fe; }
        button.active { background: #00f2fe; color: #000; }
        #hud { position: absolute; inset: 0; pointer-events: none; display: none; }
        #scoreboard { position: absolute; top: 30px; left: 50%; transform: translateX(-50%); font-size: 36px; font-weight: 900; }
        #boost-container { position: absolute; bottom: 50px; right: 50px; width: 300px; }
        #boost-bar { height: 14px; background: rgba(0,242,254,0.1); border: 2px solid #00f2fe; border-radius: 20px; overflow: hidden; }
        #boost-fill { height: 100%; width: 100%; background: linear-gradient(90deg, #00f2fe, #7000ff); box-shadow: 0 0 20px #00f2fe; }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;900&display=swap" rel="stylesheet">
</head>
<body>

    <div id="menu">
        <div class="panel">
            <h1>CAR GARAGE*</h1>
            <div class="btn-group">
                <button id="btn-octane" class="active" onclick="selectCar('octane')">OCTANE-X</button>
                <button id="btn-dominus" onclick="selectCar('dominus')">DOMINUS-GT</button>
            </div>
            <button style="font-size: 24px; width: 100%; border-color: #7000ff; color: #7000ff;" onclick="startGame()">ENTER ARENA</button>
        </div>
    </div>

    <div id="hud">
        <div id="scoreboard"><span style="color:#00f2fe">BLUE</span> <span id="s-blue">0</span> | <span id="s-orange">0</span> <span style="color:#ff8c00">ORANGE</span></div>
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

        let gameState = 'MENU';
        const keys = {};
        let boost = 100;
        let ballCam = false;

        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const world = new CANNON.World();
        world.gravity.set(0, -25, 0);

        // Arena Visuals
        const grid = new THREE.GridHelper(300, 60, 0x00f2fe, 0x050515);
        scene.add(grid);
        const groundBody = new CANNON.Body({ mass: 0, shape: new CANNON.Plane() });
        groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
        world.addBody(groundBody);

        // Ball
        const ballBody = new CANNON.Body({ mass: 2, shape: new CANNON.Sphere(2.5) });
        ballBody.position.set(0, 5, -20);
        world.addBody(ballBody);
        const ballMesh = new THREE.Mesh(new THREE.SphereGeometry(2.5, 32, 32), new THREE.MeshStandardMaterial({ color: 0xffffff, emissive: 0x00f2fe, emissiveIntensity: 0.5 }));
        scene.add(ballMesh);

        let carBody, carMesh;

        function createCar(type) {
            if(carBody) world.removeBody(carBody);
            if(carMesh) scene.remove(carMesh);

            const isOctane = type === 'octane';
            const bodyColor = isOctane ? 0x00eaff : 0xff0044;
            
            // Hitbox
            const dim = isOctane ? {w:1.6, h:1.1, l:2.4} : {w:1.7, h:0.7, l:2.8};
            carBody = new CANNON.Body({ mass: 35, shape: new CANNON.Box(new CANNON.Vec3(dim.w, dim.h, dim.l)) });
            carBody.position.set(0, 2, 30);
            world.addBody(carBody);

            carMesh = new THREE.Group();
            
            // Iridescent Paint Logic (Matching your screenshot)
            const bodyMat = new THREE.MeshStandardMaterial({ 
                color: bodyColor, 
                metalness: 0.9, 
                roughness: 0.1, 
                emissive: isOctane ? 0x002244 : 0x440011,
                emissiveIntensity: 0.2 
            });

            // MAIN CHASSIS (Aggressive Taper)
            const mainBody = new THREE.Mesh(new THREE.BoxGeometry(dim.w*1.9, dim.h*0.8, dim.l*1.8), bodyMat);
            mainBody.position.y = 0.2;
            carMesh.add(mainBody);

            // THE CABIN (Sloped Windows)
            const cabinGeo = new THREE.BoxGeometry(dim.w*1.4, dim.h*0.7, dim.l*0.8);
            const cabin = new THREE.Mesh(cabinGeo, new THREE.MeshStandardMaterial({ color: 0x050505, metalness: 1 }));
            cabin.position.set(0, dim.h*0.8, isOctane ? 0.2 : 0.4);
            carMesh.add(cabin);

            // SPOILER / WING
            const wing = new THREE.Mesh(new THREE.BoxGeometry(dim.w*2, 0.15, 0.6), bodyMat);
            wing.position.set(0, dim.h*0.9, dim.l*0.85);
            carMesh.add(wing);

            // WHEELS (Beefy Tires)
            const wheelGeo = new THREE.CylinderGeometry(0.65, 0.65, 0.5, 24);
            const wheelMat = new THREE.MeshStandardMaterial({ color: 0x111111, roughness: 1 });
            const wheelPos = [ [1.6, -0.4, 1.4], [-1.6, -0.4, 1.4], [1.6, -0.4, -1.4], [-1.6, -0.4, -1.4] ];
            wheelPos.forEach(p => {
                const w = new THREE.Mesh(wheelGeo, wheelMat);
                w.rotation.z = Math.PI/2;
                w.position.set(p[0], p[1], p[2]);
                carMesh.add(w);
            });

            // NEON UNDERGLOW
            const glow = new THREE.PointLight(bodyColor, 10, 5);
            glow.position.set(0, -0.5, 0);
            carMesh.add(glow);

            scene.add(carMesh);
        }

        createCar('octane');
        scene.add(new THREE.DirectionalLight(0xffffff, 2).set(20, 50, 10));
        scene.add(new THREE.AmbientLight(0xffffff, 0.4));

        window.selectCar = (t) => {
            document.getElementById('btn-octane').className = t==='octane'?'active':'';
            document.getElementById('btn-dominus').className = t==='dominus'?'active':'';
            createCar(t);
        };

        window.startGame = () => {
            gameState = 'PLAYING';
            document.getElementById('menu').style.display = 'none';
            document.getElementById('hud').style.display = 'block';
        };

        window.addEventListener('keydown', e => {
            keys[e.code] = true;
            if(e.code === 'Space' && gameState === 'PLAYING') ballCam = !ballCam;
            if(e.code === 'KeyK' && Math.abs(carBody.velocity.y) < 0.2) carBody.velocity.y = 14;
        });
        window.addEventListener('keyup', e => keys[e.code] = false);

        function animate() {
            requestAnimationFrame(animate);
            if(gameState === 'PLAYING') {
                world.fixedStep();
                const fwd = new THREE.Vector3(0,0,-1).applyQuaternion(carMesh.quaternion);
                if(keys['KeyW']) carBody.velocity.addScaledVector(0.7, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), carBody.velocity);
                if(keys['KeyS']) carBody.velocity.addScaledVector(-0.4, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), carBody.velocity);
                if(keys['KeyA']) carBody.angularVelocity.y = 3.8;
                else if(keys['KeyD']) carBody.angularVelocity.y = -3.8;
                else carBody.angularVelocity.y *= 0.92;

                if(keys['ShiftLeft'] && boost > 0) {
                    carBody.velocity.addScaledVector(1.0, new CANNON.Vec3(fwd.x, fwd.y, fwd.z), carBody.velocity);
                    boost -= 0.7;
                } else if(boost < 100) boost += 0.15;

                carMesh.position.copy(carBody.position);
                carMesh.quaternion.copy(carBody.quaternion);
                ballMesh.position.copy(ballBody.position);
                document.getElementById('boost-fill').style.width = boost + "%";

                const camOff = ballCam ? new THREE.Vector3(0, 9, 17) : new THREE.Vector3(0, 6, 14);
                camera.position.lerp(carMesh.position.clone().add(camOff.applyQuaternion(carMesh.quaternion)), 0.1);
                camera.lookAt(ballCam ? ballMesh.position : carMesh.position);
            } else {
                carMesh.rotation.y += 0.015;
                camera.position.set(12, 6, 12);
                camera.lookAt(carMesh.position);
            }
            renderer.render(scene, camera);
        }
        animate();
    </script>
</body>
</html>
