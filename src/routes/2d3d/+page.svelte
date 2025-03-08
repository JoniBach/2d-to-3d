<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import paper from 'paper';
  
  let paperCanvas, threeCanvas;
  let drawingPath, threeScene, threeCamera, threeRenderer, threeControls, threePattern;
  
  const GRID_SPACING = 20;
  const GRID_COLOR = '#e0e0e0';
  
  function drawGrid() {
    const { left, right, top, bottom } = paper.view.bounds;
    for (let x = left; x <= right; x += GRID_SPACING) {
      new paper.Path.Line({ from: [x, top], to: [x, bottom], strokeColor: GRID_COLOR, data: { grid: true } });
    }
    for (let y = top; y <= bottom; y += GRID_SPACING) {
      new paper.Path.Line({ from: [left, y], to: [right, y], strokeColor: GRID_COLOR, data: { grid: true } });
    }
  }
  
  const handleMouseDown = (event) => {
    drawingPath = new paper.Path({ strokeColor: 'red', strokeWidth: 2 });
    drawingPath.add(event.point);
  };

  const handleMouseDrag = (event) => {
    drawingPath.add(event.point);
    paper.view.draw();
  };

  const handleMouseUp = (event) => {
    drawingPath.add(event.point);
    update3DPattern();
  };
  
  function initPaper() {
    paper.setup(paperCanvas);
    drawGrid();
    paper.view.onMouseDown = handleMouseDown;
    paper.view.onMouseDrag = handleMouseDrag;
    paper.view.onMouseUp = handleMouseUp;
    paper.view.draw();
  }
  
  function initThree() {
    const { clientWidth: width, clientHeight: height } = threeCanvas;
    threeScene = new THREE.Scene();
    threeScene.background = new THREE.Color(0xffffff);
    threeCamera = new THREE.PerspectiveCamera(77, width / height, 0.1, 1000);
    threeCamera.position.set(0, 0, 250);
    threeCamera.lookAt(0, 0, 0);
    threeRenderer = new THREE.WebGLRenderer({ canvas: threeCanvas });
    threeRenderer.setSize(width, height);
    threeRenderer.setClearColor(0xffffff);
    const gridHelper = new THREE.GridHelper(400, 20, GRID_COLOR, GRID_COLOR);
    gridHelper.rotation.x = -Math.PI / 2;
    threeScene.add(gridHelper);
    threeControls = new OrbitControls(threeCamera, threeRenderer.domElement);
    threeControls.enableDamping = true;
    threeControls.dampingFactor = 0.25;
    threeControls.enableZoom = true;
    const animate = () => {
      requestAnimationFrame(animate);
      threeControls.update();
      threeRenderer.render(threeScene, threeCamera);
    };
    animate();
  }
  
  function update3DPattern() {
    if (threePattern) {
      threeScene.remove(threePattern);
    }
    const center = paper.view.center;
    const group = new THREE.Group();
    paper.project.activeLayer.children.forEach(item => {
      if (item instanceof paper.Path && item.strokeColor && !item.data?.grid) {
        const vertices = item.segments.flatMap(segment => [segment.point.x - center.x, center.y - segment.point.y, 0]);
        if (vertices.length >= 6) {
          const geometry = new THREE.BufferGeometry();
          geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
          const material = new THREE.LineBasicMaterial({ color: 0xff0000 });
          group.add(new THREE.Line(geometry, material));
        }
      }
    });
    threePattern = group;
    threeScene.add(threePattern);
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