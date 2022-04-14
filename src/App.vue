<template>
  <div
    id="canvas-container"
    @mouseup="dropAllPoints"
    @mouseleave="dropAllPoints"
    @touchend="dropAllPoints"
    @mousemove="movePoint($event)"
    @touchmove="movePoint($event)"
  >
    <canvas ref="canvas" />
    <div
      class="control-point"
      v-for="(point, i) in points"
      :key="i"
      :style="percentify(point)"
      @mousedown="
        dropAllPoints();
        point.moving = true;
      "
      @touchstart="
        dropAllPoints();
        point.moving = true;
      "
    />
  </div>
  <div id="controls-row">
    <label for="text-for-curve">
      Enter text here:
      <input type="text" v-model="text" id="text-for-curve" />
    </label>
    <label for="lines-on">
      <input type="checkbox" id="lines-on" v-model="linesOn" />
      Guidelines visible
    </label>
    <label for="rotate-letters">
      <input type="checkbox" id="rotate-letters" v-model="rotateLetters" />
      Rotate letters
    </label>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, reactive, ref, watch } from "vue";
import { Bezier } from "bezier-js";

// innovative, i know
interface Point {
  x: number;
  y: number;
  moving?: boolean;
}
interface Dimensions {
  width: number;
  height: number;
}

const rectSide = 20;
function renderControlledArc(
  canvas: HTMLCanvasElement | null,
  linesOn: boolean,
  p1: Point,
  p2: Point,
  p3: Point,
  text: string,
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
    ctx.strokeRect(
      p1.x - rectSide / 2,
      p1.y - rectSide / 2,
      rectSide,
      rectSide
    );
    ctx.strokeRect(
      p2.x - rectSide / 2,
      p2.y - rectSide / 2,
      rectSide,
      rectSide
    );
    ctx.strokeRect(
      p3.x - rectSide / 2,
      p3.y - rectSide / 2,
      rectSide,
      rectSide
    );
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
    const curveLength = curve.length();
    const textTakesUp = totalWidth / curveLength;
    ctx.font = `${Math.min(500, testFontSize / textTakesUp)}px LivvicBold`;
    charWidths = chars.map((c) => ctx.measureText(c).width);
    totalWidth = charWidths.reduce((acc, v) => acc + v);
    let t =
      (curveLength / 2 - totalWidth / 2 + charWidths[0] / 2) / curveLength;
    for (let i = 0; i < chars.length; i++) {
      const coords = curve.get(Math.min(t, 1));
      if (rotateLetters) {
        const normal = curve.normal(Math.min(t, 1));
        ctx.translate(coords.x, coords.y);
        ctx.rotate((normal.x > 0 ? -1 : 1) * Math.acos(normal.y));
        ctx.translate(-coords.x, -coords.y);
      }
      ctx.fillText(chars[i], coords.x, coords.y);
      ctx.resetTransform();
      if (i < chars.length - 1) {
        t += (charWidths[i] / 2 + charWidths[i + 1] / 2) / curveLength;
      }
    }
  }
}

export default defineComponent({
  name: "App",
  setup(props) {
    const canvas = ref<null | HTMLCanvasElement>(null);
    // these are all placeholders:
    const text = ref("");
    const linesOn = ref(true);
    const rotateLetters = ref(true);
    const points = reactive<[Point, Point, Point]>([
      { x: 0, y: 0 },
      { x: 0, y: 0 },
      { x: 0, y: 0 },
    ]);
    const nativeRes = reactive<Dimensions>({ width: 0, height: 0 });
    const resScaleFactor = Math.ceil(devicePixelRatio);

    watch([points, text, linesOn, rotateLetters], () =>
      renderControlledArc(
        canvas.value,
        linesOn.value,
        ...points,
        text.value,
        rotateLetters.value
      )
    );

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

    function movePoint(event: MouseEvent | TouchEvent) {
      const index = points.findIndex((p) => p.moving);
      if (index == -1) return;
      if (!canvas.value) return;
      const bbox = canvas.value.getBoundingClientRect();
      let pointer: Point;
      if (event instanceof MouseEvent) {
        pointer = { x: event.clientX, y: event.clientY };
      } else {
        event.preventDefault();
        pointer = { x: event.touches[0].clientX, y: event.touches[0].clientY };
      }
      const newPoint = {
        x: (pointer.x - bbox.left) * resScaleFactor,
        y: (pointer.y - bbox.top) * resScaleFactor,
        moving: points[index].moving,
      };
      points[index] = newPoint;
    }

    function dropAllPoints() {
      for (const point of points) {
        point.moving = false;
      }
    }

    return {
      canvas,
      text,
      points,
      percentify,
      nativeRes,
      rect: rectSide / devicePixelRatio,
      movePoint,
      dropAllPoints,
      linesOn,
      rotateLetters,
    };
  },
});
</script>

<style>
@font-face {
  font-family: "LivvicBold";
  src: url("./assets/livvic-bold-webfont.woff2") format("woff2"),
    url("./assets/livvic-bold-webfont.woff") format("woff");
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
