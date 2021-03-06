<template>
  <div
    id="canvas-container"
    @mouseup="dragEnd"
    @touchend="dragEnd"
    @mousemove="drag"
    @touchmove="drag"
  >
    <canvas ref="canvas" />
    <div
      class="control-point"
      v-for="(point, i) in points"
      :key="i"
      :style="percentify(point)"
      @mousedown="dragStart(i, $event)"
      @touchstart="dragStart(i, $event)"
    />
  </div>
  <div id="controls-row">
    <label for="text-for-curve">
      Enter text here:
      <input type="text" v-model="text" id="text-for-curve" />
    </label>
    <label for="text-color">
      <input type="color" id="text-color" v-model="textColor" />
      Text Color
    </label>
    <label for="lines-on">
      <input type="checkbox" id="lines-on" v-model="linesOn" />
      Guidelines visible
    </label>
    <label for="rotate-letters">
      <input type="checkbox" id="rotate-letters" v-model="rotateLetters" />
      Rotate letters
    </label>
    <button @click="downloadPNG">Download PNG</button>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, reactive, ref, watch } from "vue";
import { Bezier } from "bezier-js";

// innovative, i know
interface Point {
  x: number;
  y: number;
  selected?: boolean;
}
interface Dimensions {
  width: number;
  height: number;
}
class Hull {
  readonly points: Point[];
  readonly length: number;
  static distance(p1: Point, p2: Point) {
    return Math.sqrt((p2.x - p1.x) ** 2 + (p2.y - p1.y) ** 2);
  }
  static computeLength(points: Point[]) {
    let acc = 0;
    for (let i = 1; i < points.length; i++) {
      acc += Hull.distance(points[i - 1], points[i]);
    }
    return acc;
  }
  static computeNormal(pointA: Point, pointB: Point) {
    const tangent: Point = {
      x: pointB.x - pointA.x,
      y: pointB.y - pointA.y,
    };
    const rotated: Point = {
      x: -tangent.y,
      y: tangent.x,
    };
    const magnitude = Hull.distance({ x: 0, y: 0 }, rotated);
    return { x: rotated.x / magnitude, y: rotated.y / magnitude };
  }
  static lerp(a: Point, b: Point, t: number): Point {
    return {
      x: b.x * t + a.x * (1 - t),
      y: b.y * t + a.y * (1 - t),
    };
  }
  constructor(points: Point[]) {
    this.points = points;
    this.length = Hull.computeLength(points);
  }
}
class HullWalker {
  hull: Hull;
  distanceWalked: number = 0;
  pointAIndex: number = 0;
  /** just used for logging/debugging */
  history: number[] = [];
  get pointA() {
    return this.hull.points[this.pointAIndex];
  }
  pointBIndex: number = 1;
  get pointB() {
    return this.hull.points[this.pointBIndex];
  }
  metaT: number = 0;
  get currentPoint() {
    return Hull.lerp(this.pointA, this.pointB, this.metaT);
  }
  get currentSegmentLength() {
    return Hull.distance(this.pointA, this.pointB);
  }
  get currentNormal(): Point {
    const baseNormal = Hull.computeNormal(this.pointA, this.pointB);
    if (this.metaT == 0.5) {
      return baseNormal;
    } else if (this.metaT < 0.5) {
      if (this.pointAIndex == 0) {
        return baseNormal;
      } else {
        const adjacentNormal = Hull.computeNormal(
          this.hull.points[this.pointAIndex - 1],
          this.pointA
        );
        const normalT = 0.5 + this.metaT;
        return Hull.lerp(adjacentNormal, baseNormal, normalT);
      }
    } else {
      if (this.pointBIndex == this.hull.points.length - 1) {
        return baseNormal;
      } else {
        const adjacentNormal = Hull.computeNormal(
          this.pointB,
          this.hull.points[this.pointBIndex + 1]
        );
        const normalT = this.metaT - 0.5;
        return Hull.lerp(baseNormal, adjacentNormal, normalT);
      }
    }
  }
  constructor(h: Hull) {
    this.hull = h;
  }
  advanceBy(distance: number) {
    let distToGo = distance;
    while (distToGo > 0) {
      const leftInSegment = Hull.distance(this.currentPoint, this.pointB);
      if (distToGo < leftInSegment) {
        this.metaT += distToGo / this.currentSegmentLength;
        distToGo = 0;
      } else {
        distToGo -= leftInSegment;
        if (this.pointBIndex + 1 >= this.hull.points.length) {
          // failsafe for if we've advanced too far
          this.metaT = 1;
          break;
        } else {
          this.pointAIndex += 1;
          this.pointBIndex += 1;
          this.metaT = 0;
        }
      }
    }
    this.distanceWalked += distance;
    this.history.push(this.distanceWalked);
  }
}

const rectSide = 20;
function renderControlledArc(
  canvas: HTMLCanvasElement | null,
  linesOn: boolean,
  p1: Point,
  p2: Point,
  p3: Point,
  text: string,
  textColor: string,
  rotateLetters: boolean
) {
  if (!canvas) return;
  const ctx = canvas.getContext("2d");
  if (!ctx) return;
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.setLineDash([]);
  ctx.strokeStyle = "black";
  ctx.lineWidth = 2;
  if (linesOn) {
    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.quadraticCurveTo(p2.x, p2.y, p3.x, p3.y);
    ctx.stroke();
    const points = [p1, p2, p3];
    const rects: [number, number, number, number][] = points.map((p) => [
      p.x - rectSide / 2,
      p.y - rectSide / 2,
      rectSide,
      rectSide,
    ]);
    ctx.fillStyle = "lightblue";
    for (let i = 0; i < 3; i++) {
      ctx.strokeRect(...rects[i]);
      if (points[i].selected) {
        ctx.fillRect(...rects[i]);
      }
    }
    ctx.fillStyle = "#000";
    ctx.setLineDash([5, 5]);
    ctx.beginPath();
    ctx.moveTo(p1.x, p1.y);
    ctx.lineTo(p2.x, p2.y);
    ctx.lineTo(p3.x, p3.y);
    ctx.stroke();
  }
  if (text) {
    const testFontSize = 50;
    ctx.font = `${testFontSize}px LivvicBold`;
    ctx.textAlign = "center";
    const chars = Array.from(text);
    let charWidths = chars.map((c) => ctx.measureText(c).width);
    let totalWidth = charWidths.reduce((acc, v) => acc + v);
    const curve = new Bezier(p1, p2, p3);
    const hull = new Hull(curve.getLUT(25));
    // for (const point of hull.points) {
    //   ctx.fillStyle = "red";
    //   ctx.beginPath();
    //   ctx.arc(point.x, point.y, 10, 0, 2 * Math.PI);
    //   ctx.fill();
    // }
    const textTakesUp = totalWidth / hull.length;
    ctx.font = `${Math.min(500, testFontSize / textTakesUp)}px LivvicBold`;
    charWidths = chars.map((c) => ctx.measureText(c).width);
    const walker = new HullWalker(hull);
    totalWidth = charWidths.reduce((acc, v) => acc + v);
    console.log("final text width:", totalWidth);
    console.log("final hull length:", hull.length);
    let pos = hull.length / 2 - totalWidth / 2 + charWidths[0] / 2;
    console.log("starting at position", pos, "along the hull");
    ctx.fillStyle = textColor;
    for (let i = 0; i < chars.length; i++) {
      walker.advanceBy(pos - walker.distanceWalked);
      const coords = walker.currentPoint;
      // if (true) {
      //   ctx.fillStyle = "red";
      //   ctx.beginPath();
      //   ctx.arc(coords.x, coords.y, 10, 0, 2 * Math.PI);
      //   ctx.fill();
      //   ctx.fillStyle = "black";
      // }
      // const coords = curve.get(Math.min(t, 1));
      if (rotateLetters) {
        const normal = walker.currentNormal;
        ctx.translate(coords.x, coords.y);
        ctx.rotate((normal.x > 0 ? -1 : 1) * Math.acos(normal.y));
        ctx.translate(-coords.x, -coords.y);
      }
      ctx.fillText(chars[i], coords.x, coords.y);
      ctx.resetTransform();
      if (i < chars.length - 1) {
        pos += charWidths[i] / 2 + charWidths[i + 1] / 2;
      }
    }
  }
}

export default defineComponent({
  name: "App",
  setup(props) {
    const canvas = ref<null | HTMLCanvasElement>(null);
    const text = ref("");
    const textColor = ref("#000000");
    const linesOn = ref(true);
    const rotateLetters = ref(true);
    // these are all placeholders:
    const points = reactive<[Point, Point, Point]>([
      { x: 0, y: 0 },
      { x: 0, y: 0 },
      { x: 0, y: 0 },
    ]);
    const nativeRes = reactive<Dimensions>({ width: 0, height: 0 });
    const resScaleFactor = Math.ceil(devicePixelRatio);

    const renderCanvas = () =>
      renderControlledArc(
        canvas.value,
        linesOn.value,
        ...points,
        text.value,
        textColor.value,
        rotateLetters.value
      );
    watch([points, text, linesOn, rotateLetters, textColor], renderCanvas);

    onMounted(() => {
      if (!canvas.value) return;
      const width = Math.round(window.innerWidth * 0.9);
      const height = Math.round(window.innerHeight * 0.8);
      nativeRes.height = canvas.value.height = height * resScaleFactor;
      nativeRes.width = canvas.value.width = width * resScaleFactor;
      canvas.value.style.width = width + "px";
      canvas.value.style.height = height + "px";
      points[0] = { x: nativeRes.width * 0.25, y: nativeRes.height * 0.75 };
      points[1] = { x: nativeRes.width * 0.5, y: nativeRes.height * 0.25 };
      points[2] = { x: nativeRes.width * 0.75, y: nativeRes.height * 0.75 };
    });

    const percentify = (p: Point) => ({
      left: (p.x / nativeRes.width) * 100 + "%",
      top: (p.y / nativeRes.height) * 100 + "%",
    });

    /** groundbreaking, i know */
    function clamp(value: number, min: number, max: number) {
      return Math.min(Math.max(value, min), max);
    }

    let pointsMoving = false;
    let pointsMoved = false;
    let lastSelectedPoint = 0;
    let lastDragPos: Point = { x: 0, y: 0 };

    function getDragPos(event: MouseEvent | TouchEvent) {
      if ("touches" in event) {
        const touch = event.touches[0];
        return { x: touch.clientX, y: touch.clientY };
      } else {
        return { x: event.clientX, y: event.clientY };
      }
    }

    function dragStart(index: number, event: MouseEvent | TouchEvent) {
      event.preventDefault();
      pointsMoving = true;
      lastDragPos = getDragPos(event);
      lastSelectedPoint = index;
      if (!event.shiftKey) {
        if (!points[index].selected) {
          deselectPoints();
          points[index].selected = true;
        }
      } else {
        points[index].selected = !points[index].selected;
      }
    }

    function drag(event: MouseEvent | TouchEvent) {
      if (!pointsMoving) return;
      if (!canvas.value) return;
      event.preventDefault();
      pointsMoved = true;
      const pointer = getDragPos(event);
      const dx = (pointer.x - lastDragPos.x) * resScaleFactor;
      const dy = (pointer.y - lastDragPos.y) * resScaleFactor;
      for (const point of points) {
        if (point.selected) {
          const newPoint = {
            x: point.x + dx,
            y: point.y + dy,
          };
          point.x = clamp(newPoint.x, 5, nativeRes.width - 5);
          point.y = clamp(newPoint.y, 5, nativeRes.height - 5);
        }
      }
      lastDragPos = pointer;
    }

    function dragEnd(event: MouseEvent) {
      const onPoint = (event.target as HTMLElement).classList.contains(
        "control-point"
      );
      if (!pointsMoved && !event.shiftKey) {
        deselectPoints();
        if (onPoint) {
          points[lastSelectedPoint].selected = true;
        }
      }
      pointsMoved = false;
      pointsMoving = false;
    }

    function deselectPoints() {
      for (const point of points) point.selected = false;
    }

    function downloadPNG() {
      if (!canvas.value) return;
      const oldLinesOn = linesOn.value;
      linesOn.value = false;
      renderCanvas();
      canvas.value.toBlob((blob) => {
        linesOn.value = oldLinesOn;
        if (!blob) return;
        const url = URL.createObjectURL(blob);
        const anchor = document.createElement("a");
        anchor.download = "curve.png";
        anchor.href = url;
        anchor.click();
      }, "png");
    }

    new FontFace(
      "LivvicBold",
      `url("fonts/livvic-bold-webfont.woff2") format("woff2"),
        url("fonts/livvic-bold-webfont.woff") format("woff")`
    )
      .load()
      .then((font) => {
        document.fonts.add(font);
        renderCanvas();
      });

    return {
      canvas,
      text,
      points,
      percentify,
      nativeRes,
      rect: rectSide / devicePixelRatio,
      dragStart,
      drag,
      dragEnd,
      deselectPoints,
      linesOn,
      rotateLetters,
      downloadPNG,
      textColor,
    };
  },
});
</script>

<style>
/* @font-face {
  font-family: "LivvicBold";
  src: url("./assets/livvic-bold-webfont.woff2") format("woff2"),
    url("./assets/livvic-bold-webfont.woff") format("woff");
  font-weight: normal;
  font-style: normal;
} */
@font-face {
  font-family: "LivvicRegular";
  src: url("./assets/livvic-regular-webfont.woff2") format("woff2"),
    url("./assets/livvic-regular-webfont.woff") format("woff");
  font-weight: normal;
  font-style: normal;
}
* {
  box-sizing: border-box;
}
#app {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: LivvicRegular, sans-serif;
}
#canvas-container {
  position: relative;
  border: 1px black dashed;
  margin-bottom: 5px;
  display: flex;
  overflow: hidden;
}
.control-point {
  position: absolute;
  transform: translate(-50%, -50%);
  cursor: pointer;
  height: v-bind('rect+"px"');
  width: v-bind('rect+"px"');
}
#controls-row > *:not(:last-child) {
  margin-right: 5px;
}
@media (max-width: 700px) {
  #controls-row {
    display: flex;
    flex-direction: column;
  }
}
</style>
