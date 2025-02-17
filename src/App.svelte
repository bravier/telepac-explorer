<script lang="ts">
  import type { Action } from "svelte/action";

  import shp from "shpjs";

  import proj4 from "proj4";

  import { Map, View } from "ol";
  import { fromLonLat } from "ol/proj";
  import { register } from "ol/proj/proj4";
  import WMTS from "ol/source/WMTS";
  import WMTSTileGrid from "ol/tilegrid/WMTS";
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
          source: new WMTS({
            url: "https://data.geopf.fr/wmts",
            layer: "ORTHOIMAGERY.ORTHOPHOTOS",
            matrixSet: "PM_0_21",
            format: "image/jpeg",
            style: "normal",
            tileGrid: new WMTSTileGrid({
              tileSize: 256,
              // https://geoservices.ign.fr/documentation/services/services-deprecies/images-tuilees-wmts-ogc#résolutions-et-échelles-en-webmercator-sphérique
              // https://github.com/IGNF/geopf-extensions-openlayers/blob/a264423/src/packages/Utils/LayerUtils.js#L142-L165
              resolutions: [
                156543.033928041, 78271.51696402048, 39135.758482010235,
                19567.87924100512, 9783.93962050256, 4891.96981025128,
                2445.98490512564, 1222.99245256282, 611.49622628141,
                305.7481131407048, 152.8740565703525, 76.43702828517624,
                38.21851414258813, 19.10925707129406, 9.554628535647032,
                4.777314267823516, 2.388657133911758, 1.194328566955879,
                0.5971642834779395, 0.2985821417389697, 0.1492910708694849,
                0.0746455354347424,
              ],
              matrixIds: Array.from({ length: 22 }, (_, i) => i.toString()),
              origin: [-20037508.3427891992032528, 20037508.3427891992032528],
            }),
          }),
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
