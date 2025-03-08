<script>
  import { onMount } from 'svelte';
  import paper from 'paper';
  
  let canvas;
  let firstBlackPoint;


  function createCircle(point, radius, fillColor) {
    const circle = new paper.Path.Circle({
      center: point,
      radius,
      fillColor
    });
    paper.project.activeLayer.addChild(circle);
    console.log(`${fillColor} circle created at:`, point);
  }
  onMount(() => {
    if (!canvas) return;
    paper.setup(canvas);
    const tool = new paper.Tool();
    let path;
  
    tool.onMouseDown = (event) => {
      if (path) path.selected = false;
      path = new paper.Path({
        segments: [event.point],
        strokeColor: 'black',
        fullySelected: false
      });
      firstBlackPoint = event.point;
      createCircle(event.point, 4, 'black');
      console.log('Current layer children count after start circle:', paper.project.activeLayer.children.length);
    };
  
    tool.onMouseDrag = (event) => {
      if (path) {
        path.add(event.point);
        createCircle(event.point, 1, 'red');
        console.log('Current layer children count after drag circle:', paper.project.activeLayer.children.length);
      }
    };
  
    tool.onMouseUp = (event) => {
      createCircle(event.point, 4, 'black');
      console.log('Current layer children count after end circle:', paper.project.activeLayer.children.length);
  
      // Check if the two black circles touch
      if (firstBlackPoint && firstBlackPoint.getDistance(event.point) <= 8) {
        const mid = firstBlackPoint.add(event.point).divide(2);
        path.firstSegment.point = mid;
        path.lastSegment.point = mid;
        path.closed = true;
        console.log('Paths joined at center:', mid);

        // Remove black circles after joining
        paper.project.activeLayer.children
          .filter(item => item !== path && item instanceof paper.Path && item.fillColor && item.fillColor.equals(new paper.Color('black')))
          .forEach(circle => circle.remove());
        console.log('Black circles removed.');
        // Fill the closed shape with a light blue color
        path.fillColor = new paper.Color('lightblue');
        console.log('Shape filled with lightblue.');
      }
  
      if (path) {
        const originalSegmentCount = path.segments.length;
        path.simplify(10);
        path.fullySelected = true;
        const newSegmentCount = path.segments.length;
        const difference = originalSegmentCount - newSegmentCount;
        const percentage = 100 - Math.round(newSegmentCount / originalSegmentCount * 100);
        console.log(`${difference} of the ${originalSegmentCount} segments were removed. Saving ${percentage}%`);
  
        paper.project.activeLayer.children
          .filter(item => item instanceof paper.Path && item.fillColor && item.fillColor.equals(new paper.Color('red')))
          .forEach(circle => circle.remove());
        console.log('Red circles removed.');
      }
  
      paper.view.update();
    };
  });
</script>

<canvas bind:this={canvas} style="width: 100%; height: 100vh;"></canvas>