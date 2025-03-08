<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
  import paper from 'paper';
  
  let paperCanvas, threeCanvas;
  let drawingPath, threeScene, threeCamera, threeRenderer, threeControls, threePattern;
  let extrusionDepth = 10;
  
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
    // Close the path and fill it when mouse is released
    drawingPath.closed = true;
    drawingPath.fillColor = 'rgba(255, 0, 0, 0.3)';
    update3DPattern();
  };
  
  function initPaper() {
    paper.setup(paperCanvas);
    drawGrid();
    Object.assign(paper.view, {
      onMouseDown: handleMouseDown,
      onMouseDrag: handleMouseDrag,
      onMouseUp: handleMouseUp
    });
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
        const points = item.segments.map(segment =>
          new THREE.Vector2(segment.point.x - center.x, center.y - segment.point.y)
        );
        if (points.length >= 2 && !item.closed && !points[0].equals(points[points.length - 1])) {
          points.push(points[0].clone());
        }
        if (points.length >= 2) {
          const shape = new THREE.Shape(points);
          const extrudeSettings = { depth: extrusionDepth, bevelEnabled: false };
          const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
          const material = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide, transparent: true, opacity: 0.5 });
          group.add(new THREE.Mesh(geometry, material));
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

<div style="margin: 10px; text-align: center;">
  <label for="extrusionSlider">Extrusion Depth: {extrusionDepth}</label>
  <input id="extrusionSlider" type="range" min="1" max="50" bind:value={extrusionDepth} on:input={update3DPattern} style="width: 300px; margin-left: 10px;" />
</div>

<div style="display: flex; flex-direction: row; justify-content: center; align-items: center;">
  <canvas bind:this={paperCanvas} style="border: 1px solid #000; margin: 10px; width: 400px; height: 400px;"></canvas>
  <canvas bind:this={threeCanvas} style="border: 1px solid #000; margin: 10px; width: 400px; height: 400px;"></canvas>
</div>