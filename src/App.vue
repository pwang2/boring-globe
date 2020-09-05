<template>
  <div ref="globeWrap" style="width: 100%; height: 100vh">
    <Globe :region="region" :width="width" :height="height"></Globe>
  </div>
</template>

<script>
import { feature } from "topojson-client";
import { sample } from "lodash-es";
import { onMounted, ref } from "vue";
import Globe from "./components/Globe.vue";
import world from "world-atlas/countries-50m.json";

export default {
  name: "App",
  components: { Globe },
  setup() {
    const globeWrap = ref(null);
    const region = ref("156");
    const width = ref(300);
    const height = ref(300);

    onMounted(() => {
      const countryCodes = feature(world, world.objects.countries)
        .features.map((d) => d?.id)
        .filter((d) => !!d);

      setInterval(async () => {
        region.value = sample(countryCodes);
      }, 5000);

      function resize() {
        const size = Math.min(
          globeWrap.value.getBoundingClientRect().width,
          globeWrap.value.getBoundingClientRect().height
        );
        width.value = size;
        height.value = size;
      }
      resize();
      window.addEventListener("resize", resize);
    });

    return { globeWrap, region, width, height };
  },
};
</script>
