<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>3D Pulley System</title>
  <style>
    body { margin: 0; overflow: hidden; background: #eef3f8; }
    #label {
      position: fixed; top: 16px; left: 16px;
      font: 16px Arial, sans-serif; color: #233044;
    }
  </style>

  <script type="importmap">
    {
      "imports": {
        "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
        "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"
      }
    }
  </script>
</head>
<body>
  <div id="label">3D Pulley System — drag to rotate, scroll to zoom</div>

  <script type="module">
    import * as THREE from "three";
    import { OrbitControls } from "three/addons/controls/OrbitControls.js";

    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xeef3f8);

    const camera = new THREE.PerspectiveCamera(
      45, window.innerWidth / window.innerHeight, 0.1, 100
    );
    camera.position.set(5, 4, 8);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    document.body.appendChild(renderer.domElement);

    const controls = new OrbitControls(camera, renderer.domElement);
    controls.target.set(0, 0.6, 0);
    controls.enableDamping = true;

    scene.add(new THREE.HemisphereLight(0xffffff, 0x7d8999, 2));
    const light = new THREE.DirectionalLight(0xffffff, 2);
    light.position.set(4, 7, 5);
    scene.add(light);

    const metal = new THREE.MeshStandardMaterial({
      color: 0x526579, metalness: 0.65, roughness: 0.3
    });
    const wheelMat = new THREE.MeshStandardMaterial({
      color: 0x2584c4, metalness: 0.35, roughness: 0.35
    });
    const ropeMat = new THREE.MeshStandardMaterial({
      color: 0xd89538, roughness: 0.8
    });
    const blockMat = new THREE.MeshStandardMaterial({
      color: 0xe45b4f, roughness: 0.55
    });

    // A beam between two points.
    function beam(a, b, radius, material) {
      const start = new THREE.Vector3(...a);
      const end = new THREE.Vector3(...b);
      const direction = new THREE.Vector3().subVectors(end, start);
      const mesh = new THREE.Mesh(
        new THREE.CylinderGeometry(radius, radius, direction.length(), 16),
        material
      );
      mesh.position.copy(start).add(end).multiplyScalar(0.5);
      mesh.quaternion.setFromUnitVectors(
        new THREE.Vector3(0, 1, 0), direction.normalize()
      );
      scene.add(mesh);
      return mesh;
    }

    // Frame
    beam([-1.5, 3.2, 0], [1.5, 3.2, 0], 0.09, metal);
    beam([-1.3, 3.2, 0], [-1.3, 2.6, 0], 0.06, metal);
    beam([1.3, 3.2, 0], [1.3, 2.6, 0], 0.06, metal);

    // Make a pulley wheel in the XY plane.
    function makePulley(x, y, radius) {
      const group = new THREE.Group();
      group.position.set(x, y, 0);

      const rim = new THREE.Mesh(
        new THREE.TorusGeometry(radius, 0.09, 12, 48),
        wheelMat
      );
      group.add(rim);

      const hub = new THREE.Mesh(
        new THREE.CylinderGeometry(0.16, 0.16, 0.22, 24),
        metal
      );
      hub.rotation.x = Math.PI / 2;
      group.add(hub);

      // Spokes
      for (let i = 0; i < 6; i++) {
        const angle = (i / 6) * Math.PI * 2;
        const spoke = new THREE.Mesh(
          new THREE.BoxGeometry(radius * 1.65, 0.055, 0.08),
          metal
        );
        spoke.rotation.z = angle;
        group.add(spoke);
      }

      scene.add(group);
      return group;
    }

    const upperWheel = makePulley(0, 1.65, 0.62);
    const lowerWheel = makePulley(0, -0.35, 0.62);

    // Axles and supports
    beam([0, 1.65, -0.2], [0, 1.65, 0.2], 0.11, metal);
    beam([0, 3.2, 0], [0, 1.65, 0], 0.045, metal);

    // Ropes: two vertical strands and curved sections around the wheels.
    beam([-0.62, 1.65, 0], [-0.62, -0.35, 0], 0.035, ropeMat);
    beam([0.62, -0.35, 0], [0.62, 1.65, 0], 0.035, ropeMat);

    function ropeArc(centerY, radius, startAngle, endAngle) {
      const points = [];
      for (let i = 0; i <= 32; i++) {
        const t = i / 32;
        const angle = startAngle + (endAngle - startAngle) * t;
        points.push(new THREE.Vector3(
          radius * Math.cos(angle),
          centerY + radius * Math.sin(angle),
          0
        ));
      }
      const curve = new THREE.CatmullRomCurve3(points);
      scene.add(new THREE.Mesh(
        new THREE.TubeGeometry(curve, 64, 0.035, 8, false),
        ropeMat
      ));
    }

    // Rope around the bottom half of the lower pulley and top half of the upper one.
    ropeArc(-0.35, 0.62, Math.PI, Math.PI * 2);
    ropeArc(1.65, 0.62, 0, Math.PI);

    // Load block hanging from the lower pulley.
    beam([0, -0.98, 0], [0, -1.3, 0], 0.045, metal);
    const load = new THREE.Mesh(
      new THREE.BoxGeometry(1.0, 0.65, 0.75),
      blockMat
    );
    load.position.set(0, -1.65, 0);
    scene.add(load);

    // Ground grid for visual reference.
    const grid = new THREE.GridHelper(12, 12, 0x9aa8b8, 0xcbd3dc);
    grid.position.y = -2.05;
    scene.add(grid);

    function animate() {
      requestAnimationFrame(animate);
      upperWheel.rotation.z -= 0.015;
      lowerWheel.rotation.z += 0.015;
      controls.update();
      renderer.render(scene, camera);
    }
    animate();

    window.addEventListener("resize", () => {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    });
  </script>
</body>
</html>
