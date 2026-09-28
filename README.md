# santanosplayz.github.io
no
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Universal Online 3D Model Viewer</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #1a1a1a;
            color: #ffffff;
            overflow: hidden;
            height: 100vh;
            display: flex;
            flex-direction: column;
        }
        header {
            background-color: #2d2d2d;
            padding: 15px;
            text-align: center;
            box-shadow: 0 4px 6px rgba(0,0,0,0.3);
            z-index: 10;
        }
        h1 {
            font-size: 1.5rem;
            color: #4caf50;
            margin-bottom: 10px;
        }
        .controls {
            display: flex;
            justify-content: center;
            gap: 10px;
            flex-wrap: wrap;
        }
        input[type="text"] {
            padding: 8px 12px;
            width: 300px;
            border: none;
            border-radius: 4px;
            background-color: #404040;
            color: white;
        }
        button {
            padding: 8px 16px;
            border: none;
            border-radius: 4px;
            background-color: #4caf50;
            color: white;
            cursor: pointer;
            font-weight: bold;
            transition: background 0.2s;
        }
        button:hover {
            background-color: #45a049;
        }
        #download-btn {
            background-color: #008cba;
        }
        #download-btn:hover {
            background-color: #007bc0;
        }
        #viewer-container {
            flex: 1;
            position: relative;
            background: radial-gradient(circle, #333333 0%, #111111 100%);
        }
        #drop-zone {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            justify-content: center;
            align-items: center;
            pointer-events: none;
            border: 4px dashed transparent;
            transition: all 0.3s;
        }
        #drop-zone.active {
            border-color: #4caf50;
            background-color: rgba(76, 175, 80, 0.1);
        }
        .status-msg {
            background-color: rgba(0,0,0,0.7);
            padding: 20px;
            border-radius: 8px;
            text-align: center;
            pointer-events: auto;
        }
    </style>
    <!-- Import Three.js and loaders from CDN -->
    <script src="https://cloudflare.com"></script>
    <script src="https://jsdelivr.net"></script>
    <script src="https://jsdelivr.net"></script>
    <script src="https://jsdelivr.net"></script>
</head>
<body>

    <header>
        <h1>Universal Online 3D Model Viewer</h1>
        <div class="controls">
            <input type="text" id="url-input" placeholder="Paste direct 3D file URL link (.glb or .gltf)...">
            <button id="load-url-btn">Load Link</button>
            <button id="download-btn" style="display: none;">Download/Export Model</button>
        </div>
    </header>

    <div id="viewer-container">
        <div id="drop-zone">
            <div class="status-msg" id="status-text">
                <p style="font-size: 1.2rem; margin-bottom: 5px;">Drag & Drop a .glb file anywhere</p>
                <p style="color: #888; font-size: 0.9rem;">Or paste a web link in the top bar</p>
            </div>
        </div>
    </div>

    <script>
        // --- 3D Scene Setup ---
        const container = document.getElementById('viewer-container');
        const scene = new THREE.Scene();
        
        // Camera configuration
        const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 1000);
        camera.position.set(0, 2, 5);

        // WebGL Renderer configuration
        const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
        renderer.setSize(container.clientWidth, container.clientHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        renderer.shadowMap.enabled = true;
        container.appendChild(renderer.domElement);

        // Interactive mouse controls
        const controls = new THREE.OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.05;

        // Realistic Lighting system
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
        scene.add(ambientLight);

        const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
        dirLight.position.set(5, 10, 7);
        scene.add(dirLight);

        // Core Tracking variables
        let loadedModel = null;
        const loader = new THREE.GLTFLoader();
        const exporter = new THREE.GLTFExporter();

        const statusText = document.getElementById('status-text');
        const downloadBtn = document.getElementById('download-btn');

        // --- Render Animation Loop ---
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }
        animate();

        // Handle browser window resize dynamically
        window.addEventListener('resize', () => {
            camera.aspect = container.clientWidth / container.clientHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(container.clientWidth, container.clientHeight);
        });

        // --- Model Loading Engine ---
        function clearCurrentModel() {
            if (loadedModel) {
                scene.remove(loadedModel);
                loadedModel = null;
                downloadBtn.style.display = 'none';
            }
        }

        function handleLoadedModel(gltf) {
            clearCurrentModel();
            loadedModel = gltf.scene;
            scene.add(loadedModel);

            // Automatically center and scale camera view to fit the custom model
            const box = new THREE.Box3().setFromObject(loadedModel);
            const size = box.getSize(new THREE.Vector3());
            const center = box.getCenter(new THREE.Vector3());

            controls.target.copy(center);
            loadedModel.position.x += (loadedModel.position.x - center.x);
            loadedModel.position.y += (loadedModel.position.y - box.min.y); // Place model on floor grid
            loadedModel.position.z += (loadedModel.position.z - center.z);

            camera.position.set(0, size.y * 1.5, size.z * 2.5);
            camera.lookAt(center);
            controls.update();

            statusText.style.display = 'none';
            downloadBtn.style.display = 'inline-block';
        }

        // --- Method 1: Load Remote URL File Path ---
        document.getElementById('load-url-btn').addEventListener('click', () => {
            const url = document.getElementById('url-input').value.trim();
            if (!url) return alert('Please enter a valid link address first.');

            statusText.style.display = 'block';
            statusText.innerHTML = '<p style="color: #4caf50;">Fetching remote model over network data...</p>';

            loader.load(url, 
                (gltf) => { handleLoadedModel(gltf); },
                undefined,
                (error) => {
                    console.error(error);
                    statusText.innerHTML = '<p style="color: #ff3333;">Error loading file link. Make sure it is a direct asset URL with valid access formatting rules.</p>';
                }
            );
        });

        // --- Method 2: Local Device Drag and Drop Interactivity ---
        const dropZone = document.getElementById('drop-zone');
        
        window.addEventListener('dragover', (e) => {
            e.preventDefault();
            dropZone.classList.add('active');
        });
        window.addEventListener('dragleave', () => {
            dropZone.classList.remove('active');
        });
        window.addEventListener('drop', (e) => {
            e.preventDefault();
            dropZone.classList.remove('active');
            
            const file = e.dataTransfer.files[0];
            if (!file) return;

            statusText.style.display = 'block';
            statusText.innerHTML = '<p style="color: #4caf50;">Processing local file payload...</p>';

            const reader = new FileReader();
            reader.onload = function (event) {
                const contents = event.target.result;
                loader.parse(contents, '', (gltf) => {
                    handleLoadedModel(gltf);
                }, (err) => {
                    statusText.innerHTML = '<p style="color: #ff3333;">Invalid file syntax data asset parsing error.</p>';
                });
            };
            reader.readAsArrayBuffer(file);
        });

        // --- Universal File Export Engine ---
        downloadBtn.addEventListener('click', () => {
            if (!loadedModel) return;
            
            exporter.parse(loadedModel, function (gltf) {
                const output = JSON.stringify(gltf, null, 2);
                const blob = new Blob([output], { type: 'application/json' });
                
                const link = document.createElement('a');
                link.href = URL.createObjectURL(blob);
                link.download = 'exported_model.gltf';
                link.click();
            }, function(error) {
                alert('Conversion pipeline packaging error occurred.');
            }, { binary: false });
        });
    </script>
