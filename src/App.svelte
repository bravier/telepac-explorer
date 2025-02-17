<script lang="ts">
  import type { Action } from "svelte/action";

  import shp from "shpjs";

  import proj4 from "proj4";

  import { Map, View } from "ol";
  import { fromLonLat } from "ol/proj";
  import { register } from "ol/proj/proj4";
  import OSM from "ol/source/OSM";
  import VectorSource from "ol/source/Vector";
  import VectorLayer from "ol/layer/Vector";
  import TileLayer from "ol/layer/Tile";
  import GeoJSON from "ol/format/GeoJSON";

  // Very useful link to understand how to reproject GeoJSON from Lambert 93 to WGS84:
  // https://benjamin.hervy.fr/blog/2022/openlayers-proj/

  proj4.defs(
    "EPSG:2154",
    "+proj=lcc +lat_0=46.5 +lon_0=3 +lat_1=49 +lat_2=44 +x_0=700000 +y_0=6600000 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=m +no_defs +type=crs"
  );
  register(proj4);

  let fileInput: HTMLInputElement;

  let map: Map | null = null;

  async function handleFileInput() {
    const files = fileInput?.files;
    if (!files?.length) {
      alert("Veuillez sélectionner au moins un fichier.");
      return;
    }
    if (Array.from(files).some((file) => file.type !== "application/zip")) {
      alert("Veuillez ne sélectionner que des fichiers .zip.");
      return;
    }

    const addShapefileToMap = async (file: File) => {
      const zip = await file.arrayBuffer();
      const geoJSON = await shp(zip);
      const vectorSource = new VectorSource({
        features: new GeoJSON().readFeatures(geoJSON, {
          dataProjection: "EPSG:2154",
          featureProjection: "EPSG:3857",
        }),
      });
      const vectorLayer = new VectorLayer({
        properties: { title: file.name },
        source: vectorSource,
      });
      map?.addLayer(vectorLayer);
      map?.getView().fit(vectorSource.getExtent());
    };

    await Promise.all([...files].map(addShapefileToMap));
  }

  const setup: Action<HTMLDivElement> = (node) => {
    map = new Map({
      target: node.id,
      layers: [
        new TileLayer({
          source: new OSM(),
        }),
      ],
      view: new View({
        center: fromLonLat([3.6, 46.9]),
        zoom: 6.5,
      }),
    });

    return {
      destroy() {
        if (map) {
          map.setTarget(undefined);
          map = null;
        }
      },
    };
  };
</script>

<div class="pico">
  <header>
    <hgroup>
      <h3>
        TelePAC Explorer
        <span style="font-size: 0.8rem"
          >par <a href="https://github.com/bravier">bravier</a></span
        >
      </h3>
      <p>Visualisez facilement vos données TelePAC dans votre navigateur.</p>
    </hgroup>
    <button id="file-input-button" onclick={() => fileInput.click()}>
      Choisir un ou plusieurs fichiers .zip
    </button>
    <input
      hidden
      type="file"
      id="file-input"
      accept="application/zip"
      multiple
      bind:this={fileInput}
      onchange={handleFileInput}
    />
  </header>
</div>

<main>
  <div id="map" use:setup></div>
</main>

<style>
  main {
    flex: 1;
    min-height: 0;
    position: relative;

    #map {
      height: 100%;
      width: 100%;
    }
  }
</style>
