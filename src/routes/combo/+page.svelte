<script>
	import { onMount } from 'svelte';
	import paper from 'paper';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
	import { SVGLoader } from 'three/examples/jsm/loaders/SVGLoader';

	let canvas;
	let threeCanvas;
	let firstBlackPoint;
	let path = null;
	let isDrawing = false;
	let selectedItems = [];
	let draggingPath = null;
	let pendingDeselect = null;

	let threeScene, threeCamera, threeRenderer, threeControls, threePattern;
	let extrusionDepth = 10;
	const GRID_SPACING = 20;
	const GRID_COLOR = '#e0e0e0';

	function createCircle(point, radius, fillColor) {
		const circle = new paper.Path.Circle({ center: point, radius, fillColor });
		paper.project.activeLayer.addChild(circle);
	}

	function removeCircles(color, exclude = null) {
		paper.project.activeLayer.children
			.filter(item => item !== exclude && item instanceof paper.Path && item.fillColor && item.fillColor.equals(new paper.Color(color)))
			.forEach(item => item.remove());
	}

	function deselectAllItems() {
		selectedItems.forEach(item => {
			item.selected = false;
			item.fullySelected = false;
		});
		selectedItems = [];
	}

	function toggleSelection(item) {
		if (selectedItems.includes(item)) {
			pendingDeselect = item;
		} else {
			item.selected = true;
			item.fullySelected = true;
			selectedItems.push(item);
		}
		draggingPath = item;
	}

	function handleShiftMouseDrag(event) {
		if (pendingDeselect) pendingDeselect = null;
		selectedItems.forEach(item => {
			item.position = item.position.add(event.delta);
		});
		update3DPattern();
	}

	function handleShiftMouseUp() {
		if (pendingDeselect) {
			pendingDeselect.selected = false;
			pendingDeselect.fullySelected = false;
			selectedItems = selectedItems.filter(item => item !== pendingDeselect);
			pendingDeselect = null;
		}
		draggingPath = null;
	}

	function drawGrid() {
		const { left, right, top, bottom } = paper.view.bounds;
		for (let x = left; x <= right; x += GRID_SPACING) {
			new paper.Path.Line({ from: [x, top], to: [x, bottom], strokeColor: GRID_COLOR, data: { grid: true } });
		}
		for (let y = top; y <= bottom; y += GRID_SPACING) {
			new paper.Path.Line({ from: [left, y], to: [right, y], strokeColor: GRID_COLOR, data: { grid: true } });
		}
	}

	const hitOptions = { segments: true, stroke: true, fill: true, tolerance: 5 };
	const isLeftButton = event => event.event.button === 0;

	function smoothPoints(points) {
		if (points.length < 3) return points;
		const curve = new THREE.CatmullRomCurve3(points.map(p => new THREE.Vector3(p.x, p.y, 0)), true);
		const segments = Math.max(points.length * 4, 20);
		const smoothed3D = curve.getPoints(segments);
		return smoothed3D.map(v => new THREE.Vector2(v.x, v.y));
	}

	function update3DPattern() {
		if (threePattern) {
			threeScene.remove(threePattern);
		}
		const group = new THREE.Group();
		const serializer = new XMLSerializer();
		const loader = new SVGLoader();
		paper.project.activeLayer.children.forEach(item => {
			if (item instanceof paper.Path && item.strokeColor && !item.data?.grid) {
				const svgNode = item.exportSVG({ asString: false });
				const svgString = serializer.serializeToString(svgNode);
				const data = loader.parse(svgString);
				data.paths.forEach(path => {
					const shapes = path.toShapes(true);
					shapes.forEach(shape => {
						const extrudeSettings = { depth: extrusionDepth, bevelEnabled: false };
						const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
						const material = new THREE.MeshBasicMaterial({ color: 0xff0000, side: THREE.DoubleSide, transparent: true, opacity: 0.5 });
						group.add(new THREE.Mesh(geometry, material));
					});
				});
			}
		});
		group.scale.y = -1;
		const center = paper.view.center;
		group.position.set(-center.x, center.y, 0);
		threePattern = group;
		threeScene.add(threePattern);
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

	onMount(() => {
		if (!canvas) return;
		paper.setup(canvas);
		drawGrid()
		const tool = new paper.Tool();
		canvas.addEventListener('contextmenu', e => e.preventDefault());

		tool.onMouseDown = event => {
			if (!isLeftButton(event)) return;
			const hitResult = paper.project.hitTest(event.point, hitOptions);
			if (event.modifiers.shift) {
				if (hitResult?.item) {
					toggleSelection(hitResult.item);
				}
				return;
			} else {
				deselectAllItems();
				draggingPath = null;
			}
			isDrawing = true;
			if (path) path.selected = false;
			path = new paper.Path({ segments: [event.point], strokeColor: 'black' });
			firstBlackPoint = event.point;
			createCircle(event.point, 4, 'black');
		};

		tool.onMouseDrag = event => {
			if (!isLeftButton(event)) return;
			if (event.modifiers.shift && draggingPath?.selected) {
				handleShiftMouseDrag(event);
				return;
			}
			if (isDrawing && path) {
				path.add(event.point);
				createCircle(event.point, 1, 'red');
			}
		};

		tool.onMouseUp = event => {
			if (!isLeftButton(event)) return;
			if (event.modifiers.shift) {
				handleShiftMouseUp();
				return;
			}
			if (!isDrawing) return;
			createCircle(event.point, 4, 'black');
			if (firstBlackPoint && firstBlackPoint.getDistance(event.point) <= 8) {
				const mid = firstBlackPoint.add(event.point).divide(2);
				path.firstSegment.point = mid;
				path.lastSegment.point = mid;
				path.closed = true;
				removeCircles('black', path);
				path.fillColor = new paper.Color('blue');
				path.fillColor.alpha = 0.05;
				update3DPattern();
			}
			path.simplify(10);
			path.fullySelected = true;
			removeCircles('red');
			paper.view.update();
			if (path && !selectedItems.includes(path)) {
				selectedItems.push(path);
			}
			isDrawing = false;
		};

		// initialize three canvas if available
		if (threeCanvas) initThree();
	});
</script>

<div style="display: flex; flex-direction: row; justify-content: center; align-items: center;">
	<canvas bind:this={canvas} style="border: 1px solid #000; margin: 10px; width: 400px; height: 400px;"></canvas>
	<canvas bind:this={threeCanvas} style="border: 1px solid #000; margin: 10px; width: 400px; height: 400px;"></canvas>
</div>
