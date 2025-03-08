<script>
	import { onMount } from 'svelte';
	import paper from 'paper';

	let canvas;
	let firstBlackPoint;
	let path;

	function createCircle(point, radius, fillColor) {
		const circle = new paper.Path.Circle({
			center: point,
			radius,
			fillColor
		});
		paper.project.activeLayer.addChild(circle);
		console.log(`${fillColor} circle created at:`, point);
	}

	function removeCircles(color, exclude = null) {
		paper.project.activeLayer.children
			.filter(
				(item) =>
					item !== exclude &&
					item instanceof paper.Path &&
					item.fillColor &&
					item.fillColor.equals(new paper.Color(color))
			)
			.forEach((item) => item.remove());
	}

	onMount(() => {
		if (!canvas) return;
		paper.setup(canvas);
		const tool = new paper.Tool();
		canvas.addEventListener('contextmenu', (e) => e.preventDefault());

		tool.onMouseDown = (event) => {
			if (event.event.button === 0) {
				if (path) path.selected = false;
				path = new paper.Path({
					segments: [event.point],
					strokeColor: 'black'
				});
				firstBlackPoint = event.point;
				createCircle(event.point, 4, 'black');
				console.log('Layer children count after start:', paper.project.activeLayer.children.length);
			}
		};

		tool.onMouseDrag = (event) => {
			if (event.event.button === 0) {
				if (path) {
					path.add(event.point);
					createCircle(event.point, 1, 'red');
					console.log(
						'Layer children count after drag:',
						paper.project.activeLayer.children.length
					);
				}
			}
		};

		tool.onMouseUp = (event) => {
			if (event.event.button === 0) {
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

				if (path) {
					const originalCount = path.segments.length;
					path.simplify(10);
					path.fullySelected = true;
					const newCount = path.segments.length;
					const removed = originalCount - newCount;
					const percentage = 100 - Math.round((newCount / originalCount) * 100);
					console.log(
						`${removed} of the ${originalCount} segments were removed. Saving ${percentage}%`
					);
					removeCircles('red');
					console.log('Red circles removed.');
				}

				paper.view.update();
			}
		};
	});
</script>

<canvas bind:this={canvas} style="width: 100%; height: 100vh;"></canvas>
