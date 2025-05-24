# Danger
Open world game
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Block City Runner</title>
  <style>
    body { margin: 0; overflow: hidden; }
    canvas { display: block; }
    #hud {
      position: absolute;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      color: white;
      font-size: 2em;
      z-index: 10;
      text-shadow: 2px 2px 8px #000;
      font-family: Arial, sans-serif;
      pointer-events: none;
    }
    #win-message {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      color: yellow;
      font-size: 3em;
      z-index: 20;
      text-shadow: 2px 2px 8px #000;
      font-family: Arial, sans-serif;
      display: none;
    }
    #game-title {
      position: absolute;
      top: 20px;
      width: 100%;
      text-align: center;
      color: white;
      text-shadow: 2px 2px 8px #000;
      font-size: 3em;
      z-index: 10;
      font-family: Arial, sans-serif;
      pointer-events: none;
    }
  </style>
</head>
<body>
  <div id="game-title">Block City Runner</div>
  <div id="hud">Score: 0</div>
  <div id="win-message">You Win!</div>
  <script src="https://cdn.jsdelivr.net/npm/three@0.150.1/build/three.min.js"></script>
  <script>
    // Setup scene, camera, renderer
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.7);
    scene.add(ambientLight);
    const dirLight = new THREE.DirectionalLight(0xffffff, 1);
    dirLight.position.set(10, 20, 10);
    scene.add(dirLight);

    // Player cube
    const playerMaterial = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
    const player = new THREE.Mesh(new THREE.BoxGeometry(1, 1, 1), playerMaterial);
    player.position.y = 0.5;
    scene.add(player);

    // Ground
    const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(100, 100),
      new THREE.MeshStandardMaterial({ color: 0x444444, side: THREE.DoubleSide })
    );
    ground.receiveShadow = true;
    ground.rotation.x = -Math.PI / 2;
    scene.add(ground);

    // Buildings
    const buildings = [];
    function createBuilding(x, z, width, height, depth, color) {
      const geometry = new THREE.BoxGeometry(width, height, depth);
      const material = new THREE.MeshStandardMaterial({ color });
      const building = new THREE.Mesh(geometry, material);
      building.position.set(x, height / 2, z);
      building.castShadow = true;
      scene.add(building);
      buildings.push(building);
    }
    for (let i = -20; i <= 20; i += 10) {
      for (let j = -20; j <= 20; j += 10) {
        const width = 4 + Math.random() * 3;
        const height = 5 + Math.random() * 15;
        const depth = 4 + Math.random() * 3;
        createBuilding(i, j, width, height, depth, 0x888888);
      }
    }

    // Collectibles
    const collectibles = [];
    function createCollectible(x, z) {
      const geometry = new THREE.SphereGeometry(0.4, 16, 16);
      const material = new THREE.MeshStandardMaterial({ color: 0xffee00, emissive: 0x333300 });
      const collectible = new THREE.Mesh(geometry, material);
      collectible.position.set(x, 0.4, z);
      collectible.castShadow = true;
      scene.add(collectible);
      collectibles.push(collectible);
    }
    // Place 10 collectibles randomly, not too close to buildings
    function randomPos() {
      return -40 + Math.random() * 80;
    }
    let placed = 0;
    while (placed < 10) {
      let x = randomPos();
      let z = randomPos();
      let safe = true;
      for (const b of buildings) {
        const dx = b.position.x - x;
        const dz = b.position.z - z;
        if (Math.abs(dx) < 3 && Math.abs(dz) < 3) {
          safe = false;
          break;
        }
      }
      if (safe) {
        createCollectible(x, z);
        placed++;
      }
    }

    // HUD
    const hud = document.getElementById('hud');
    let score = 0;

    // Camera initial position
    camera.position.set(0, 5, 10);
    camera.lookAt(player.position);

    // Movement controls
    const keys = {};
    document.addEventListener('keydown', e => keys[e.key.toLowerCase()] = true);
    document.addEventListener('keyup', e => keys[e.key.toLowerCase()] = false);

    // Game state
    let gameWon = false;

    function updatePlayer() {
      if (gameWon) return;
      const speed = 0.17;
      const rotSpeed = 0.07;

      // Rotate player
      if (keys['a']) player.rotation.y += rotSpeed;
      if (keys['d']) player.rotation.y -= rotSpeed;

      // Move forward/backward in facing direction
      let dir = new THREE.Vector3(
        Math.sin(player.rotation.y),
        0,
        Math.cos(player.rotation.y)
      );
      let move = new THREE.Vector3();
      if (keys['w']) move.add(dir.clone().multiplyScalar(speed));
      if (keys['s']) move.add(dir.clone().multiplyScalar(-speed));

      // Simple collision with buildings
      let nextPos = player.position.clone().add(move);
      let collides = false;
      for (const b of buildings) {
        const box = new THREE.Box3().setFromObject(b);
        let playerBox = new THREE.Box3().setFromCenterAndSize(
          nextPos.clone().setY(0.5),
          new THREE.Vector3(1, 1, 1)
        );
        if (box.intersectsBox(playerBox)) {
          collides = true;
          break;
        }
      }
      // Clamp to ground
      nextPos.x = Math Math.min(49, nextPos.x));
      nextPos.z = Math.max(-49, Math.min(49, nextPos.z));
      if (!collides) player.position.copy(nextPos);
    }

    function updateCamera() {
      // Camera stays behind the player
      const offset = new THREE.Vector3(
        Math.sin(player.rotation.y) * 8,
        5,
        Math.cos(player.rotation.y) * 8
      );
      camera.position.lerp(player.position.clone().add(offset), 0.18);
      camera.lookAt(player.position);
    }

    function checkCollectibles() {
      for (let i = collectibles {
        const c = collectibles[i];
        if (player.position.distanceTo(c.position) < 1.0) {
          scene.remove(c);
          collectibles.splice(i, 1);
          score++;
          hud.textContent = `Score: ${score}`;
          if (score === 10) {
            document.getElementById('win-message').style.display = 'block';
            gameWon = true;
          }
        }
      }
    }

    // Resize handler
    window.addEventListener('resize', () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window camera);
    }
    animate();
  </script>
</body>
</html>
