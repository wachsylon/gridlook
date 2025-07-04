<script lang="ts" setup>
import * as THREE from "three";
import * as zarr from "zarrita";
import {
  makeColormapMaterial,
  availableColormaps,
  calculateColorMapProperties,
} from "./utils/colormapShaders.ts";
import { decodeTime } from "./utils/timeHandling.ts";

import { datashaderExample } from "./utils/exampleFormatters.ts";
import {
  computed,
  onBeforeMount,
  ref,
  shallowRef,
  onMounted,
  watch,
  type Ref,
  type ShallowRef,
} from "vue";

import { useGlobeControlStore } from "./store/store.js";
import { storeToRefs } from "pinia";
import type {
  TBounds,
  TColorMap,
  TSources,
  TVarInfo,
} from "../types/GlobeTypes.ts";
import { useToast } from "primevue/usetoast";
import { getErrorMessage } from "./utils/errorHandling.ts";
import { useSharedGlobeLogic } from "./sharedGlobe.ts";

const props = defineProps<{
  datasources?: TSources;
  varbounds?: TBounds;
  colormap?: TColorMap;
  invertColormap?: boolean;
}>();

const emit = defineEmits<{ varinfo: [TVarInfo] }>();
const store = useGlobeControlStore();
const toast = useToast();
const { timeIndexSlider, timeIndex, varnameSelector, varname } =
  storeToRefs(store);

const datavars: ShallowRef<
  Record<string, zarr.Array<zarr.DataType, zarr.FetchStore>>
> = shallowRef({});
const updateCount = ref(0);
const updatingData = ref(false);

const estimatedSpacing = ref(0);

let points: THREE.Points | undefined = undefined;

let canvas: Ref<HTMLCanvasElement | undefined> = ref();
let box: Ref<HTMLDivElement | undefined> = ref();

const {
  getScene,
  getCamera,
  redraw,
  makeSnapshot,
  toggleRotate,
  getDataVar,
  getTimeVar,
  registerUpdateLOD,
} = useSharedGlobeLogic(canvas, box);

watch(
  () => varnameSelector.value,
  () => {
    getData();
  }
);

watch(
  () => timeIndexSlider.value,
  () => {
    getData();
  }
);

watch(
  () => props.datasources,
  () => {
    datasourceUpdate();
  }
);

watch(
  () => props.varbounds,
  () => {
    updateColormap();
  }
);

watch(
  () => props.invertColormap,
  () => {
    updateColormap();
  }
);

watch(
  () => props.colormap,
  () => {
    updateColormap();
  }
);

const colormapMaterial = computed(() => {
  if (props.invertColormap) {
    return makeColormapMaterial(props.colormap, 1.0, -1.0);
  } else {
    return makeColormapMaterial(props.colormap, 0.0, 1.0);
  }
});

const gridsource = computed(() => {
  if (props.datasources) {
    return props.datasources.levels[0].grid;
  } else {
    return undefined;
  }
});

const datasource = computed(() => {
  if (props.datasources) {
    return props.datasources.levels[0].datasources[varnameSelector.value];
  } else {
    return undefined;
  }
});

function publishVarinfo(info: TVarInfo) {
  emit("varinfo", info);
}

async function datasourceUpdate() {
  datavars.value = {};
  if (props.datasources !== undefined) {
    await Promise.all([getData()]);
    updateColormap();
  }
}

function updateColormap() {
  const low = props.varbounds?.low as number;
  const high = props.varbounds?.high as number;
  const { addOffset, scaleFactor } = calculateColorMapProperties(
    low,
    high,
    props.invertColormap
  );

  const material = points!.material as THREE.ShaderMaterial;
  material.uniforms.colormap.value = availableColormaps[props.colormap!];
  material.uniforms.addOffset.value = addOffset;
  material.uniforms.scaleFactor.value = scaleFactor;
  material.needsUpdate = true;
  redraw();
}

function latLonToCartesianFlat(
  lat: number,
  lon: number,
  out: Float32Array,
  offset: number,
  radius = 1
) {
  const latRad = (lat * Math.PI) / 180;
  const lonRad = (lon * Math.PI) / 180;
  out[offset] = radius * Math.cos(latRad) * Math.cos(lonRad);
  out[offset + 1] = radius * Math.cos(latRad) * Math.sin(lonRad);
  out[offset + 2] = radius * Math.sin(latRad);
}

function estimateAverageSpacing(positions: Float32Array): number {
  /*
    Estimate average spacing between points based on first 10 points
  */
  let totalDist = 0;
  let count = 0;
  for (let i = 0; i < positions.length - 6 && count < 10; i += 3) {
    const dx = positions[i] - positions[i + 3];
    const dy = positions[i + 1] - positions[i + 4];
    const dz = positions[i + 2] - positions[i + 5];
    const dist = Math.sqrt(dx * dx + dy * dy + dz * dz);
    totalDist += dist;
    count++;
  }
  return totalDist / count;
}

async function getGrid(grid: zarr.Group<zarr.Readable>, data: Float64Array) {
  // Load latitudes and longitudes arrays (1D)
  const latitudes = (
    await zarr.open(grid.resolve("lat"), { kind: "array" }).then(zarr.get)
  ).data as Float64Array;

  const longitudes = (
    await zarr.open(grid.resolve("lon"), { kind: "array" }).then(zarr.get)
  ).data as Float64Array;

  const N = latitudes.length;

  if (longitudes.length !== N || data.length !== N) {
    throw new Error(
      "Latitudes, longitudes, and data must have the same length"
    );
  }

  // Allocate typed arrays for positions and values
  const positions = new Float32Array(N * 3);
  const dataValues = new Float32Array(N);

  // Convert lat/lon to Cartesian positions
  for (let i = 0; i < N; i++) {
    latLonToCartesianFlat(latitudes[i], longitudes[i], positions, i * 3);
    dataValues[i] = data[i];
  }

  estimatedSpacing.value = estimateAverageSpacing(positions);

  points!.geometry.setAttribute(
    "position",
    new THREE.BufferAttribute(positions, 3)
  );
  points!.geometry.setAttribute(
    "data_value",
    new THREE.BufferAttribute(dataValues, 1)
  );
  points!.geometry.computeBoundingSphere();
  const material = points!.material as THREE.ShaderMaterial;
  material.needsUpdate = true;
  updateLOD();
}

function updateLOD() {
  /* FIXME: Points do not scale automatically when the camera zooms in.
  This is a hack to make the points bigger when the camera is close by
  taking into acount some the distance of some arbitrary points (estimatedSpacing)
  the distance of the camera (cameraDistance) and some scaling factor (k).

  The size will vary between 0.5 and 30, which is probably not the best way to do it.
  It would be better to have the size also depend on the screen size aswell.
   */
  const cameraDistance = getCamera()!.position.length();
  const k = 70000; // tweak this value to taste
  const size = THREE.MathUtils.clamp(
    (k * estimatedSpacing.value) / cameraDistance,
    0.5,
    30 // maybe something dynamic, depending on the screen size?
  );
  const material: THREE.ShaderMaterial = points!
    .material as THREE.ShaderMaterial;
  material.uniforms.pointSize.value = size;
}

async function getData() {
  store.startLoading();
  try {
    updateCount.value += 1;
    const myUpdatecount = updateCount.value;
    if (updatingData.value) {
      return;
    }
    updatingData.value = true;
    const localVarname = varnameSelector.value;
    const currentTimeIndexSliderValue = timeIndexSlider.value;
    const [timevar, datavar] = await Promise.all([
      getTimeVar(props.datasources!),
      getDataVar(localVarname, props.datasources!),
    ]);
    let timeinfo = {};
    if (timevar !== undefined) {
      const timeattrs = timevar.attrs;
      const timevalues = (await zarr.get(timevar, [null])).data;
      timeinfo = {
        attrs: timeattrs,
        values: timevalues,
        current: decodeTime(
          (timevalues as number[])[currentTimeIndexSliderValue],
          timeattrs
        ),
      };
    }
    if (datavar !== undefined) {
      const root = zarr.root(new zarr.FetchStore(gridsource.value!.store));
      const grid = await zarr.open(root.resolve(gridsource.value!.dataset), {
        kind: "group",
      });
      const rawData = await zarr.get(datavar, [
        currentTimeIndexSliderValue,
        ...Array(datavar.shape.length - 1).fill(null),
      ]);
      let min = Number.POSITIVE_INFINITY;
      let max = Number.NEGATIVE_INFINITY;
      for (let i of rawData.data as Float64Array) {
        if (Number.isNaN(i)) continue;
        min = Math.min(min, i);
        max = Math.max(max, i);
      }
      await getGrid(grid, rawData.data as Float64Array);
      publishVarinfo({
        attrs: datavar.attrs,
        timeinfo,
        timeRange: { start: 0, end: datavar.shape[0] - 1 },
        bounds: { low: min, high: max },
      });
      timeIndex.value = currentTimeIndexSliderValue;
      varname.value = localVarname;
    }
    updatingData.value = false;
    if (updateCount.value !== myUpdatecount) {
      await getData();
    }
  } catch (error) {
    toast.add({
      detail: `Couldn't fetch data: ${getErrorMessage(error)}`,
      life: 3000,
    });
    updatingData.value = false;
  } finally {
    store.stopLoading();
  }
}

function copyPythonExample() {
  const example = datashaderExample({
    cameraPosition: getCamera()!.position,
    datasrc: datasource.value!.store + datasource.value!.dataset,
    gridsrc: gridsource.value!.store + gridsource.value!.dataset,
    varname: varname.value,
    timeIndex: timeIndex.value,
    varbounds: props.varbounds!,
    colormap: props.colormap!,
    invertColormap: props.invertColormap,
  });
  navigator.clipboard.writeText(example);
}

onMounted(() => {
  let sphereGeometry = new THREE.SphereGeometry(0.999, 64, 64);
  const earthMat = new THREE.MeshBasicMaterial({ color: 0x000000 }); // black color

  // it is quite likely that the data points do not cover the whole globe
  // in order to avoid some ugly transparency issues, we add an opaque black
  // sphere underneath
  const globeMesh = new THREE.Mesh(sphereGeometry, earthMat);
  globeMesh.geometry.attributes.position.needsUpdate = true;
  globeMesh.rotation.x = Math.PI / 2;
  globeMesh.geometry.computeBoundingBox();
  globeMesh.geometry.computeBoundingSphere();

  getScene()?.add(points as THREE.Points);
  getScene()?.add(globeMesh);
});

onBeforeMount(async () => {
  const geometry = new THREE.BufferGeometry();
  const material = colormapMaterial.value;
  points = new THREE.Points(geometry, material);
  await datasourceUpdate();
  registerUpdateLOD(updateLOD);
});

defineExpose({ makeSnapshot, copyPythonExample, toggleRotate });
</script>

<template>
  <div ref="box" class="globe_box" tabindex="0" autofocus>
    <canvas ref="canvas" class="globe_canvas"> </canvas>
  </div>
</template>

<style>
div.globe_box {
  height: 100%;
  width: 100%;
  padding: 0;
  margin: 0;
  overflow: hidden;
  display: flex;
  z-index: 0;
}

div.globe_canvas {
  padding: 0;
  margin: 0;
}
</style>
