<script>
	import { onMount } from 'svelte';
	import paper from 'paper';

	let canvas;
	let firstBlackPoint;
	let path = null;
	let isDrawing = false;
	let selectedItems = [];
	let draggingPath = null;
	let pendingDeselect = null;

	function createCircle(point, radius, fillColor) {
		const circle = new paper.Path.Circle({ center: point, radius, fillColor });
		paper.project.activeLayer.addChild(circle);
	}

	function removeCircles(color, exclude = null) {
		paper.project.activeLayer.children
			.filter(item => item !== exclude && item instanceof paper.Path && item.fillColor && item.fillColor.equals(new paper.Color(color)))
			.forEach(item => item.remove());
	}

	const hitOptions = { segments: true, stroke: true, fill: true, tolerance: 5 };
	const isLeftButton = event => event.event.button === 0;

	onMount(() => {
		if (!canvas) return;
		paper.setup(canvas);
		const tool = new paper.Tool();
		canvas.addEventListener('contextmenu', e => e.preventDefault());

		tool.onMouseDown = event => {
			if (!isLeftButton(event)) return;
			const hitResult = paper.project.hitTest(event.point, hitOptions);

			if (event.modifiers.shift) {
				if (hitResult?.item) {
					if (selectedItems.includes(hitResult.item)) {
						pendingDeselect = hitResult.item;
						draggingPath = hitResult.item;
					} else {
						hitResult.item.selected = true;
						hitResult.item.fullySelected = true;
						selectedItems.push(hitResult.item);
						draggingPath = hitResult.item;
					}
				}
				return;
			} else if (selectedItems.length > 0) {
				selectedItems.forEach(item => {
					item.selected = false;
					item.fullySelected = false;
				});
				selectedItems = [];
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
				if (pendingDeselect) pendingDeselect = null;
				selectedItems.forEach(item => {
					item.position = item.position.add(event.delta);
				});
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
				if (pendingDeselect) {
					pendingDeselect.selected = false;
					pendingDeselect.fullySelected = false;
					selectedItems = selectedItems.filter(item => item !== pendingDeselect);
					pendingDeselect = null;
				}
				draggingPath = null;
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
			}

			const originalCount = path.segments.length;
			path.simplify(10);
			path.fullySelected = true;
			removeCircles('red');
			paper.view.update();
			if (path && !selectedItems.includes(path)) {
				selectedItems.push(path);
			}
			isDrawing = false;
		};
	});
</script>

<canvas bind:this={canvas} style="width: 100%; height: 100vh;"></canvas>
