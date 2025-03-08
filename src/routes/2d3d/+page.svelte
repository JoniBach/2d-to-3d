<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import paper from 'paper';
  
  let paperCanvas;
  let threeCanvas;
  let paperCircle, threeCircle;
  
  // Initialize Paper.js canvas, draw grid and add a red circle with drag functionality
  function initPaper() {
    paper.setup(paperCanvas);
    const gridSpacing = 20;
    const { left, right, top, bottom } = paper.view.bounds;
    for (let x = left; x <= right; x += gridSpacing) {
      new paper.Path.Line({ from: [x, top], to: [x, bottom], strokeColor: '#e0e0e0' });
    }
    for (let y = top; y <= bottom; y += gridSpacing) {
      new paper.Path.Line({ from: [left, y], to: [right, y], strokeColor: '#e0e0e0' });
    }
    paperCircle = new paper.Path.Circle({ center: paper.view.center, radius: 50, fillColor: 'red' });
    paperCircle.onMouseDown = function(event) {
      this.data.offset = this.position.subtract(event.point);
    };
    paperCircle.onMouseDrag = function(event) {
      this.position = event.point.add(this.data.offset);
      if (threeCircle) {
        const { x: centerX, y: centerY } = paper.view.center;
        threeCircle.position.x = this.position.x - centerX;
        threeCircle.position.y = centerY - this.position.y;
        threeCircle.position.z = 0;
      }
    };
    paper.view.draw();
  }
  
  // Initialize Three.js canvas with scene, camera, grid and controls
  function initThree() {
    const width = threeCanvas.clientWidth;
    const height = threeCanvas.clientHeight;
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0xffffff);
    const camera = new THREE.PerspectiveCamera(77, width / height, 0.1, 1000);
    camera.position.set(0, 0, 250);
    camera.lookAt(0, 0, 0);
    const renderer = new THREE.WebGLRenderer({ canvas: threeCanvas });
    renderer.setSize(width, height);
    renderer.setClearColor(0xffffff);
    const gridHelper = new THREE.GridHelper(400, 20, 0xe0e0e0, 0xe0e0e0);
    gridHelper.rotation.x = -Math.PI / 2;
    scene.add(gridHelper);
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.25;
    controls.enableZoom = true;
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
  
  onMount(() => {
    if (paperCanvas) initPaper();
    if (threeCanvas) initThree();
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