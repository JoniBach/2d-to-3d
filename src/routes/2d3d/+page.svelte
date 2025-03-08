<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import paper from 'paper';
  
  let paperCanvas;
  let threeCanvas;
  let paperCircle, threeCircle;
  onMount(() => {
    // Initialize Paper.js canvas
    if (paperCanvas) {
      paper.setup(paperCanvas);
      // Draw a simple background grid for Paper.js canvas
      const gridSpacing = 20;
      const bounds = paper.view.bounds;
      for (let x = bounds.left; x <= bounds.right; x += gridSpacing) {
        new paper.Path.Line({
          from: [x, bounds.top],
          to: [x, bounds.bottom],
          strokeColor: '#e0e0e0'
        });
      }
      for (let y = bounds.top; y <= bounds.bottom; y += gridSpacing) {
        new paper.Path.Line({
          from: [bounds.left, y],
          to: [bounds.right, y],
          strokeColor: '#e0e0e0'
        });
      }
      // Create a red circle at the center
      paperCircle = new paper.Path.Circle({
        center: paper.view.center,
        radius: 50,
        fillColor: 'red'
      });
      paperCircle.onMouseDown = function(event) {
        this.data.offset = this.position.subtract(event.point);
      };

      paperCircle.onMouseDrag = function(event) {
        this.position = event.point.add(this.data.offset);
        // Update the 3D circle position to replicate the 2D circle movement
        if (threeCircle) {
          const center = paper.view.center;
          threeCircle.position.x = this.position.x - center.x;
          threeCircle.position.y = center.y - this.position.y; // Invert Y movement
          threeCircle.position.z = 0;
        }
      };
      paper.view.draw();
    }
    // Initialize Three.js canvas
    if (threeCanvas) {
      const width = threeCanvas.clientWidth;
      const height = threeCanvas.clientHeight;
      const scene = new THREE.Scene();
      scene.background = new THREE.Color(0xffffff);
      const camera = new THREE.PerspectiveCamera(
        77, // Field of view
        width / height, // Aspect ratio
        0.1,
        1000
      );
      camera.position.set(0, 0, 250); // Adjusted Z position for closer zoom
      camera.lookAt(0, 0, 0);

      const renderer = new THREE.WebGLRenderer({ canvas: threeCanvas });
      renderer.setSize(width, height);
      renderer.setClearColor(0xffffff);

      // Add a background grid to the Three.js scene
      const gridHelper = new THREE.GridHelper(400, 20, 0xe0e0e0, 0xe0e0e0);
      gridHelper.rotation.x = -Math.PI / 2; // Rotate to lie in the XY plane
      scene.add(gridHelper);

      const controls = new OrbitControls(camera, renderer.domElement);
      controls.enableDamping = true; // Enable damping (inertia)
      controls.dampingFactor = 0.25; // Damping factor
      controls.enableZoom = true; // Allow zooming
      controls.update();

      // Create a 2D circle and add to scene
      const geometry = new THREE.CircleGeometry(50, 32);
      const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
      threeCircle = new THREE.Mesh(geometry, material);
      scene.add(threeCircle);

      function animate() {
        requestAnimationFrame(animate);
        controls.update();
        renderer.render(scene, camera);
      }

      animate();
    }
  });
</script>

<style>
  .container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
  }
  canvas {
    border: 1px solid #000;
    margin: 10px;
  }
  .paper-canvas, .three-canvas {
    width: 400px;
    height: 400px;
  }
</style>

<div class="container">
  <canvas bind:this={paperCanvas} class="paper-canvas"></canvas>
  <canvas bind:this={threeCanvas} class="three-canvas"></canvas>
</div>