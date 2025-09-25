[index.html](https://github.com/user-attachments/files/22541446/index.html)
<!doctype html>
<html lang="bg">
<head>
  <meta charset="utf-8">
  <title>Kicker 44CVX152 — 3D Box Preview (27 Hz)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    body { margin:0; font-family: Arial, sans-serif; background:#111; color:#ddd }
    #info { position:absolute; left:12px; top:12px; z-index:2; background:rgba(0,0,0,0.5); padding:10px; border-radius:6px}
    #canvas { width:100vw; height:100vh; display:block }
    label{display:block; margin-top:6px; font-size:13px}
    input[type=range]{ width:200px }
  </style>
</head>
<body>
  <div id="info">
    <strong>Kicker 44CVX152 — Vented box tuned ~27 Hz</strong>
    <div>Вътрешен полезен обем ~51.3 L (включва компенсиране за V<sub>as</sub>/V<sub>d</sub>)</div>
    <div>Примерни вътрешни размери: <strong>450 × 300 × 380 mm (W×H×D)</strong></div>
    <div>Препоръчан кръгъл порт (пример): Ø 75 mm, дължина ≈ 309 mm (акустична дължина)</div>
    <div style="margin-top:6px">Можеш да местиш модела с мишката (завъртане) и да използваш слайдъра за скала.</div>
    <label>Скала <input id="scale" type="range" min="0.4" max="1.6" step="0.01" value="1"></label>
  </div>
  <canvas id="canvas"></canvas>

  <!-- Three.js from CDN -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r152/three.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r152/examples/js/controls/OrbitControls.js"></script>
  <script>
    // Scene setup
    const canvas = document.getElementById('canvas');
    const renderer = new THREE.WebGLRenderer({ canvas, antialias:true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio || 1);

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0b0b0b);

    const camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 5000);
    camera.position.set(800, 400, 900);

    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.target.set(0,100,0);
    controls.update();

    // Lights
    const hemi = new THREE.HemisphereLight(0xffffff, 0x222222, 0.9);
    scene.add(hemi);
    const dir = new THREE.DirectionalLight(0xffffff, 0.6);
    dir.position.set(1000,1000,500);
    scene.add(dir);

    // Units: millimeters -> scale down to practical three.js units (1 unit = 1 mm)
    // Box internal dims (mm)
    const internal = { w:450, h:300, d:380 };
    const panel = 18; // panel thickness mm

    // External dims (mm)
    const ext = { w: internal.w + 2*panel, h: internal.h + 2*panel, d: internal.d + 2*panel };

    // Convert to three.js scale factor so model fits on screen
    let globalScale = 0.8; // starting scale

    // Materials
    const matWood = new THREE.MeshStandardMaterial({ color:0x2b2b2b, metalness:0.1, roughness:0.7 });
    const matPort = new THREE.MeshStandardMaterial({ color:0x111111, metalness:0.2, roughness:0.4 });
    const matDriver = new THREE.MeshStandardMaterial({ color:0x003366, metalness:0.2, roughness:0.3 });

    // Outer box (external dimensions)
    const boxGeom = new THREE.BoxGeometry(ext.w*globalScale, ext.h*globalScale, ext.d*globalScale);
    const boxMesh = new THREE.Mesh(boxGeom, matWood);
    boxMesh.position.set(0, ext.h*globalScale/2, 0);
    scene.add(boxMesh);

    // Cutout to show interior: create inner box and boolean-like visual via a hollow inner mesh (semi-transparent)
    const innerGeom = new THREE.BoxGeometry(internal.w*globalScale, internal.h*globalScale, internal.d*globalScale);
    const innerMat = new THREE.MeshStandardMaterial({ color:0x111111, opacity:0.12, transparent:true });
    const innerMesh = new THREE.Mesh(innerGeom, innerMat);
    innerMesh.position.set(0, panel*globalScale/2 + internal.h*globalScale/2, 0);
    scene.add(innerMesh);

    // Driver (approx. flange diameter 350mm, depth ~215 mm mounting)
    const driverDia = 350; // mm approx mounting cutout
    const driverDepth = 215;
    const driverGeom = new THREE.CylinderGeometry(driverDia*globalScale/2, driverDia*globalScale/2, 6, 64);
    const driver = new THREE.Mesh(driverGeom, matDriver);
    // put driver on front face center
    const frontZ = -ext.d*globalScale/2 + panel*globalScale/2;
    driver.rotation.x = Math.PI/2;
    driver.position.set(0, panel*globalScale/2 + internal.h*globalScale/2, frontZ + 3);
    scene.add(driver);

    // Port: side port example (circular) centered on right side baffle
    // Port acoustic length example (internal) ~309 mm for Ø75 mm, we'll show physical length inside box
    const portDia = 75; // mm
    const portLen = 309; // mm (acoustic length target)
    // For visualization, put port centered vertically on side and flush with outer side
    const portGeom = new THREE.CylinderGeometry((portDia*globalScale)/2, (portDia*globalScale)/2, portLen*globalScale, 40);
    const port = new THREE.Mesh(portGeom, matPort);
    port.rotation.z = Math.PI/2; // align along box depth
    // position the port so its inner end is inside the internal volume and its outer end flush with external side
    const rightX = ext.w*globalScale/2 - panel*globalScale/2; // outer side x
    // put port through right side: center vertically
    const portY = panel*globalScale/2 + internal.h*globalScale/2;
    // place so that outer end is at rightX
    port.position.set(rightX - portLen*globalScale/2, portY, 0);
    scene.add(port);

    // small label helper
    const createLabel = (text, x,y,z) => {
      const div = document.createElement('div');
      div.style.position='absolute'; div.style.color='white'; div.style.fontSize='12px';
      div.innerHTML = text;
      document.body.appendChild(div);
      return {el:div, world:new THREE.Vector3(x,y,z)};
    };

    const label = createLabel('Port Ø75 mm — acoustic length ≈ 309 mm',  rightX - portLen*globalScale/2, portY + 50*globalScale, 0);

    function updateLabels(){
      const pos = label.world.clone();
      pos.project(camera);
      const x = (pos.x * 0.5 + 0.5) * window.innerWidth;
      const y = ( -pos.y * 0.5 + 0.5) * window.innerHeight;
      label.el.style.left = x + 'px';
      label.el.style.top = y + 'px';
    }

    // Resize handling
    window.addEventListener('resize', ()=>{
      renderer.setSize(window.innerWidth, window.innerHeight);
      camera.aspect = window.innerWidth/window.innerHeight; camera.updateProjectionMatrix();
    });

    // scale slider
    document.getElementById('scale').addEventListener('input', (e)=>{
      globalScale = parseFloat(e.target.value);
      // update sizes quickly by setting scale on root nodes
      boxMesh.scale.set(globalScale, globalScale, globalScale);
      innerMesh.scale.set(globalScale, globalScale, globalScale);
      driver.scale.set(globalScale, globalScale, globalScale);
      port.scale.set(globalScale, globalScale, globalScale);
    });

    // animate
    function animate(){
      requestAnimationFrame(animate);
      controls.update();
      renderer.render(scene, camera);
      updateLabels();
    }
    animate();
  </script>
</body>
</html>
