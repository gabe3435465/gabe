<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Realistic 3D Webcam (HTML + JS)</title>
  <style>
    html,body{height:100%;margin:0;background:#000;overflow:hidden;font-family:system-ui,Segoe UI,Roboto,Arial}
    #ui { position: fixed; left: 12px; top: 12px; z-index: 20; color: #fff; background: rgba(0,0,0,0.4); padding: 8px 10px; border-radius: 8px; }
    #err { position: fixed; left: 12px; bottom: 12px; z-index: 20; color: #fff; background: rgba(160,0,0,0.9); padding: 8px 10px; border-radius: 8px; display:none }
  </style>
</head>
<body>
  <div id="ui">Allow camera → realistic 3D webcam</div>
  <div id="err"></div>
  <video id="cam" autoplay playsinline muted style="display:none"></video>

  <!-- Three.js + examples from CDN -->
  <script src="https://unpkg.com/three@0.160.0/build/three.min.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/controls/OrbitControls.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/loaders/RGBELoader.js"></script>

  <script src="https://unpkg.com/three@0.160.0/examples/js/postprocessing/EffectComposer.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/postprocessing/RenderPass.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/postprocessing/UnrealBloomPass.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/postprocessing/SSAOPass.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/postprocessing/ShaderPass.js"></script>
  <script src="https://unpkg.com/three@0.160.0/examples/js/shaders/FXAAShader.js"></script>

  <script>
  (async function(){
    const ui = id => document.getElementById(id);
    function showErr(m){ const e = ui('err'); e.style.display='block'; e.textContent = m; console.error(m); }

    // 1) Camera
    const video = ui('cam');
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ video: { width: 1280, height: 720 }, audio: false });
      video.srcObject = stream;
      await video.play();
    } catch(e) {
      showErr('Camera access failed: ' + (e && e.message ? e.message : e));
      return;
    }

    // 2) Renderer, scene, camera
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(innerWidth, innerHeight);
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.outputEncoding = THREE.sRGBEncoding;
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;
    document.body.appendChild(renderer.domElement);

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x070707);

    const camera = new THREE.PerspectiveCamera(45, innerWidth/innerHeight, 0.1, 100);
    camera.position.set(0, 1.6, 4);

    const controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.target.set(0,1,0);
    controls.enableDamping = true;
    controls.dampingFactor = 0.08;

    // 3) HDR environment for reflections (fallback if load fails)
    const rgbe = new THREE.RGBELoader();
    let envMap = null;
    try {
      const hdr = await new Promise((res, rej) => {
        rgbe.setDataType(THREE.UnsignedByteType)
            .load('https://raw.githubusercontent.com/mrdoob/three.js/dev/examples/textures/equirectangular/royal_esplanade_1k.hdr', res, undefined, rej);
      });
      const pmrem = new THREE.PMREMGenerator(renderer);
      pmrem.compileEquirectangularShader();
      envMap = pmrem.fromEquirectangular(hdr).texture;
      hdr.dispose(); pmrem.dispose();
      scene.environment = envMap;
    } catch(err) {
      console.warn('HDR load failed — continuing without env map', err);
    }

    // 4) Lights
    const hemi = new THREE.HemisphereLight(0xffffff, 0x222222, 0.5);
    scene.add(hemi);

    const dir = new THREE.DirectionalLight(0xffffff, 1.2);
    dir.position.set(4, 8, 4);
    dir.castShadow = true;
    dir.shadow.mapSize.set(2048, 2048);
    dir.shadow.camera.left = -6; dir.shadow.camera.right = 6;
    dir.shadow.camera.top = 6; dir.shadow.camera.bottom = -6;
    dir.shadow.radius = 3;
    scene.add(dir);

    // 5) Floor (PBR)
    const floorGeo = new THREE.PlaneGeometry(40, 40);
    const floorMat = new THREE.MeshStandardMaterial({
      color: 0x0f0f10,
      roughness: 0.45,
      metalness: 0.18,
      envMap: envMap,
      envMapIntensity: 0.9
    });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI/2;
    floor.receiveShadow = true;
    floor.position.y = 0;
    scene.add(floor);

    // 6) Contact shadow (soft ellipse texture)
    function contactShadowTexture(size=1024){
      const c = document.createElement('canvas');
      c.width = c.height = size;
      const ctx = c.getContext('2d');
      ctx.clearRect(0,0,size,size);
      const g = ctx.createRadialGradient(size/2, size/2, size*0.05, size/2, size/2, size*0.6);
      g.addColorStop(0, 'rgba(0,0,0,0.65)');
      g.addColorStop(1, 'rgba(0,0,0,0)');
      ctx.fillStyle = g;
      ctx.beginPath();
      ctx.ellipse(size/2, size/2, size*0.45, size*0.25, 0, 0, Math.PI*2);
      ctx.fill();
      const t = new THREE.CanvasTexture(c);
      t.encoding = THREE.sRGBEncoding;
      return t;
    }
    const shadowPlane = new THREE.Mesh(new THREE.PlaneGeometry(3.6, 2.4), new THREE.MeshBasicMaterial({ map: contactShadowTexture(), transparent: true, depthWrite: false }));
    shadowPlane.rotation.x = -Math.PI/2;
    shadowPlane.position.y = 0.01;
    scene.add(shadowPlane);

    // 7) Webcam screen object: bezel + glass + screen
    // video texture
    const videoTex = new THREE.VideoTexture(video);
    videoTex.minFilter = THREE.LinearFilter;
    videoTex.magFilter = THREE.LinearFilter;
    videoTex.format = THREE.RGBAFormat;
    videoTex.encoding = THREE.sRGBEncoding;
    videoTex.repeat.x = -1; videoTex.offset.x = 1; // mirror horizontally

    // bezel
    const bezelMat = new THREE.MeshStandardMaterial({ color: 0x0b0b0b, roughness: 0.6, metalness: 0.1 });
    const bezel = new THREE.Mesh(new THREE.BoxGeometry(1.9, 1.2, 0.12), bezelMat);
    bezel.position.set(0, 1.05, 0);
    bezel.castShadow = true;
    bezel.receiveShadow = true;
    scene.add(bezel);

    // screen plane
    const screenMat = new THREE.MeshPhysicalMaterial({
      map: videoTex,
      emissive: new THREE.Color(0x111111),
      emissiveIntensity: 0.9,
      metalness: 0.02,
      roughness: 0.08,
      reflectivity: 0.3,
      clearcoat: 0.6,
      clearcoatRoughness: 0.03
    });
    const screen = new THREE.Mesh(new THREE.PlaneGeometry(1.6, 0.9), screenMat);
    screen.position.set(0, 1.05, 0.065);
    screen.castShadow = false;
    screen.receiveShadow = false;
    scene.add(screen);

    // small stand under the monitor
    const standMat = new THREE.MeshStandardMaterial({ color: 0x121212, roughness: 0.6, metalness: 0.2 });
    const standBase = new THREE.Mesh(new THREE.BoxGeometry(1.1, 0.06, 0.5), standMat);
    standBase.position.set(0, 0.18, 0.2);
    standBase.receiveShadow = true;
    scene.add(standBase);
    const standNeck = new THREE.Mesh(new THREE.BoxGeometry(0.08, 0.5, 0.08), standMat);
    standNeck.position.set(0, 0.46, 0.2);
    standNeck.castShadow = true;
    scene.add(standNeck);

    // 8) postprocessing: composer + passes (Render, SSAO, Bloom, FXAA)
    const composer = new THREE.EffectComposer(renderer);
    const renderPass = new THREE.RenderPass(scene, camera);
    composer.addPass(renderPass);

    const ssaoPass = new THREE.SSAOPass(scene, camera, innerWidth, innerHeight);
    ssaoPass.kernelRadius = 16;
    ssaoPass.output = THREE.SSAOPass.OUTPUT.Default;
    ssaoPass.minDistance = 0.001;
    ssaoPass.maxDistance = 0.3;
    composer.addPass(ssaoPass);

    const bloomPass = new THREE.UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), 0.35, 0.5, 0.1);
    composer.addPass(bloomPass);

    const fxaaPass = new THREE.ShaderPass(THREE.FXAAShader);
    fxaaPass.uniforms['resolution'].value.set(1 / innerWidth, 1 / innerHeight);
    composer.addPass(fxaaPass);

    // small helper: exposure + optional vignette via shader could be added later

    // 9) Animate / render
    const clock = new THREE.Clock();
    let t = 0;
    function animate(){
      requestAnimationFrame(animate);
      const dt = clock.getDelta();
      t += dt * 0.25;

      // subtle model motion for life
      bezel.rotation.y = Math.sin(t*0.3) * 0.02;
      bezel.rotation.x = Math.sin(t*0.15) * 0.01;
      screen.rotation.copy(bezel.rotation);
      standBase.rotation.copy(bezel.rotation);
      standNeck.rotation.copy(bezel.rotation);
      shadowPlane.position.x = bezel.position.x;
      shadowPlane.position.z = bezel.position.z;

      controls.update();
      composer.render(dt);
    }
    animate();

    // 10) handle resize
    window.addEventListener('resize', ()=>{
      renderer.setSize(innerWidth, innerHeight);
      composer.setSize(innerWidth, innerHeight);
      camera.aspect = innerWidth / innerHeight;
      camera.updateProjectionMatrix();
      ssaoPass.setSize(innerWidth, innerHeight);
      fxaaPass.uniforms['resolution'].value.set(1 / innerWidth, 1 / innerHeight);
    });

    // 11) sanity checks: WebGL support
    if(!renderer.getContext()){
      showErr('WebGL not available in this browser.');
    }

  })();
  </script>
</body>
</html>
