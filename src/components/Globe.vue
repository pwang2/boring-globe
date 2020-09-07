<template>
  <canvas ref="globe" />
</template>

<script>
import { watch, watchEffect, ref, toRefs, onMounted } from "vue";
import { feature } from "topojson-client";
import { presimplify, simplify } from "topojson-simplify";
import versor from "versor/src/index.js";
import worldRaw from "world-atlas/countries-50m.json";
import d3 from "../utils/d3.js";

function drag(projection) {
  let v0, q0, r0, a0;

  function dragstarted(event) {
    v0 = versor.cartesian(projection.invert(d3.pointer(event, this)));
    q0 = versor((r0 = projection.rotate()));
  }

  function dragged(event) {
    const pt = d3.pointer(event, this);
    const v1 = versor.cartesian(projection.rotate(r0).invert(pt));
    const delta = versor.delta(v0, v1);
    let q1 = versor.multiply(q0, delta);

    if (pt[2]) {
      const d = (pt[2] - a0) / 2;
      const s = -Math.sin(d);
      const c = Math.sign(Math.cos(d));
      q1 = versor.multiply([Math.sqrt(1 - s * s), 0, 0, c * s], q1);
    }

    projection.rotate(versor.rotation(q1));
    if (delta[0] < 0.7) dragstarted.apply(this, [event, this]);
  }

  return d3.drag().on("start", dragstarted).on("drag", dragged);
}

function getInterpolates(countries) {
  let r1,
    p1,
    r2 = [0, 0, 0],
    p2 = [0, 0];

  return (code) => {
    const country = countries.find((d) => String(d.id) === code);
    r1 = r2;
    p1 = p2;
    p2 = d3.geoCentroid(country);
    r2 = [-p2[0], 0 - p2[1], 0];
    const ip = d3.geoInterpolate(p1, p2);
    const iv = versor.interpolate(r1, r2);
    return [country, ip, iv, p1, p2];
  };
}

function init(canvas, width, height, autospin) {
  if (!canvas) return;

  const sphere = { type: "Sphere" };
  const extend = [
    [20, 20],
    [width - 20, height - 20],
  ];
  const ratio = window.devicePixelRatio;
  canvas.style.width = width + "px";
  canvas.style.height = height + "px";
  canvas.width = ratio * width;
  canvas.height = ratio * height;

  const ctx = canvas.getContext("2d");
  ctx.scale(ratio, ratio);

  const projection = d3.geoOrthographic().fitExtent(extend, sphere);
  const path = d3.geoPath(projection, ctx);
  const graticule = d3.geoGraticule().step([20, 20])();

  const presimplifiedWorld = presimplify(worldRaw);
  const world = simplify(presimplifiedWorld, 0.1);
  const land = feature(world, world.objects.land);
  const countries = feature(world, world.objects.countries).features;

  const glowFn = getEarthGlow(width, height);

  function getEarthGlow(width, height) {
    let canvas;

    return () => {
      if (canvas) return canvas;

      const glowCanvas = document.createElement("canvas");
      glowCanvas.width = width;
      glowCanvas.height = height;

      const ctx = glowCanvas.getContext("2d");
      const path = d3.geoPath(projection, ctx);

      ctx.beginPath(),
        path(sphere),
        (ctx.strokeStyle = "#333"),
        (ctx.fillStyle = "#a2c0f4"),
        (ctx.lineWidth = 0.5),
        (ctx.shadowColor = "#333"),
        (ctx.shadowBlur = 20),
        ctx.stroke(),
        ctx.fill();

      return ctx.canvas;
    };
  }

  function render(country, arc) {
    ctx.clearRect(0, 0, width, height);
    ctx.drawImage(glowFn(), 0, 0);

    ctx.beginPath(), path(land), (ctx.fillStyle = "#fbf8f4"), ctx.fill();

    if (country) {
      ctx.beginPath(), path(country), (ctx.fillStyle = "#f008"), ctx.fill();
    }
    if (arc) {
      ctx.beginPath(), path(arc), (ctx.lineWidth = 1), ctx.stroke();
    }

    ctx.beginPath(),
      path(graticule),
      ((ctx.strokeStyle = "#666"), (ctx.lineWidth = 0.3)),
      ctx.stroke();

    return ctx.canvas;
  }

  function drawLabel(ctx, name, [x, y]) {
    if (!name) return;

    const curStyle = ctx.fillStyle;
    //draw circle to make small countries visible on the map
    ctx.moveTo(x, y);
    ctx.fillStyle = "rgba(51, 102, 51, 1)";
    ctx.beginPath();
    ctx.arc(x, y, 2, 0, 2 * Math.PI);
    ctx.fill();

    ctx.fillStyle = "#333";
    ctx.font = "12px Arial";
    ctx.textAlign = "left";
    ctx.textBaseline = "middle";

    ctx.fillText(name, x + 3, y);
    ctx.fillStyle = curStyle;
  }

  function createRotateTimer() {
    let lastTime = d3.now();
    return (elapsed) => {
      const now = d3.now();
      const diff = now - lastTime;
      if (diff < elapsed) {
        const rotation = projection.rotate();
        rotation[0] += diff * 0.01;
        projection.rotate(rotation);
        render();
      }
      lastTime = now;
    };
  }

  const interpolates = getInterpolates(countries);
  const rotateCb = createRotateTimer();
  const autorotate = d3.timer(rotateCb);
  if (!autospin) autorotate.stop();

  d3.select(canvas)
    .call(drag(projection).on("drag.render", render).on("end.render", render))
    .on("mousemove", function (event) {
      autorotate.stop();
      const pos = projection.invert(d3.pointer(event, this));
      const c = countries.find((c) => d3.geoContains(c, pos));
      if (c) {
        render(c);
        drawLabel(ctx, c?.properties?.name, projection(d3.geoCentroid(c)));
      }
    })
    .on("mouseout", () => autorotate.restart(rotateCb, 2000));

  return function animate(codeTo, codeFrom) {
    autorotate && autorotate.stop();
    if (!codeTo) return;
    const [countryTo, ip, iv, p1, p2] = interpolates(codeTo);
    const countryToName = countryTo?.properties?.name;
    const countryFrom = countries.find((c) => String(c.id) === codeFrom);
    const countryFromName = countryFrom?.properties?.name;

    d3.select(canvas)
      .transition()
      .duration(1500)
      .tween("rotate", () => (t) => {
        // var startTime = performance.now();
        projection.rotate(iv(t));
        render(countryTo, { type: "LineString", coordinates: [p1, ip(t)] });

        drawLabel(ctx, countryFromName, projection(p1));
        // var endTime = performance.now();
        // console.log(endTime - startTime);
      })
      .transition()
      .duration(500)
      .tween("render", () => (t) => {
        render(countryTo, { type: "LineString", coordinates: [ip(t), p2] });
        drawLabel(ctx, countryToName, [width / 2, height / 2]);
      })
      .on("end", () => autorotate.restart(rotateCb, 3000));
  };
}

export default {
  props: {
    region: { type: String },
    width: { type: Number, default: 500 },
    height: { type: Number, default: 500 },
    autospin: { type: Boolean, default: true },
  },

  setup(props) {
    const globe = ref(null);
    const { width, height, region, autospin } = toRefs(props);

    onMounted(() =>
      watchEffect(() => {
        const animate = init(
          globe.value,
          width.value,
          height.value,
          autospin.value
        );
        watch(region, animate, { immediate: true });
      })
    );

    return { globe };
  },
};
</script>
