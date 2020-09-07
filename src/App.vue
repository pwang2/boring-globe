<template>
  <div ref="globeWrap" style="width: 100%; height: 100vh">
    <Globe
      :region="region"
      :width="width"
      :height="height"
      :autospin="autospin"
    ></Globe>
  </div>
</template>

<script>
import { feature } from "topojson-client";
import { sample } from "lodash-es";
import { ref, onMounted, onUnmounted } from "vue";
import Globe from "./components/Globe.vue";
import world, { objects } from "world-atlas/countries-50m.json";

const countryCodes = feature(world, objects.countries)
  .features.map((d) => d?.id)
  .filter((d) => !!d);

export default {
  name: "App",
  components: { Globe },
  setup() {
    const globeWrap = ref(null);
    const region = ref("156");
    const width = ref(300);
    const height = ref(300);
    const autospin = ref(true);

    function resize() {
      const bcr = this.getBoundingClientRect();
      const size = Math.min(bcr.width, bcr.height);
      width.value = size;
      height.value = size;
    }

    onMounted(() => {
      resize.call(globeWrap.value);
      window.addEventListener("resize", resize.bind(globeWrap.value));
    });

    onUnmounted(() => {
      window.removeEventListener("resize", resize.bind(globeWrap.value));
    });

    setInterval(() => (region.value = sample(countryCodes)), 50000);
    return { globeWrap, region, width, height, autospin };
  },
};
</script>
