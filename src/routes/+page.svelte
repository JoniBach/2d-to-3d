<script>
	import { onMount } from 'svelte';
	import paper from 'paper';

	let canvas, firstBlackPoint, path, selectedItem = null, draggingPath = null, isDrawing = false;

	function createCircle(point, radius, fillColor) {
		const circle = new paper.Path.Circle({ center: point, radius, fillColor });
		paper.project.activeLayer.addChild(circle);
		console.log(`${fillColor} circle created at:`, point);
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
			
			if (selectedItem) {
				if (hitResult?.item === selectedItem) {
					if (event.modifiers.shift) {
						draggingPath = selectedItem;
					}
					return; // Clicking on the selected shape does nothing
				} else {
					selectedItem.selected = false;
					selectedItem.fullySelected = false;
					console.log('Deselected shape');
					selectedItem = null;
				}
			}
			
			if (event.modifiers.shift) {
				if (hitResult?.item) {
					hitResult.item.selected = true;
					hitResult.item.fullySelected = true;
					selectedItem = hitResult.item;
					draggingPath = hitResult.item;
					console.log('Shape selected with shift at:', event.point);
				}
				return;
			} else {
				isDrawing = true;
				if (path) path.selected = false;
				path = new paper.Path({ segments: [event.point], strokeColor: 'black' });
				firstBlackPoint = event.point;
				createCircle(event.point, 4, 'black');
				console.log('Layer children count after start:', paper.project.activeLayer.children.length);
			}
		};

		tool.onMouseDrag = event => {
			if (!isLeftButton(event)) return;
			if (event.modifiers.shift && draggingPath && draggingPath.selected) {
				draggingPath.position = draggingPath.position.add(event.delta);
			} else if (isDrawing && path) {
				path.add(event.point);
				createCircle(event.point, 1, 'red');
				console.log('Layer children count after drag:', paper.project.activeLayer.children.length);
			}
		};

		tool.onMouseUp = event => {
			if (!isLeftButton(event)) return;
			if (event.modifiers.shift) {
			    draggingPath = null;
			    return;
			} else {
			    if (!isDrawing) return;
			    createCircle(event.point, 4, 'black');
			    console.log('Layer children count after end:', paper.project.activeLayer.children.length);
			    if (firstBlackPoint && firstBlackPoint.getDistance(event.point) <= 8) {
				    const mid = firstBlackPoint.add(event.point).divide(2);
				    path.firstSegment.point = mid;
				    path.lastSegment.point = mid;
				    path.closed = true;
				    console.log('Paths joined at center:', mid);
				    removeCircles('black', path);
				    console.log('Black circles removed.');
				    path.fillColor = new paper.Color('blue');
				    path.fillColor.alpha = 0.05;
				    console.log('Shape filled with blue.');
			    }
			    const originalCount = path.segments.length;
			    path.simplify(10);
			    path.fullySelected = true;
			    const newCount = path.segments.length;
			    const removed = originalCount - newCount;
			    const percentage = 100 - Math.round((newCount / originalCount) * 100);
			    console.log(`${removed} of the ${originalCount} segments were removed. Saving ${percentage}%`);
			    removeCircles('red');
			    console.log('Red circles removed.');
			    paper.view.update();
			    isDrawing = false;
			}
		};
	});
</script>

<canvas bind:this={canvas} style="width: 100%; height: 100vh;"></canvas>
