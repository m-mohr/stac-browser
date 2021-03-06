<template>
  <div>
    <b-alert v-if="errored" variant="danger" show>{{
      _entity.message
    }}</b-alert>
    <b-spinner v-else-if="!loaded" label="Loading..."></b-spinner>
    <b-container v-else :class="loaded && 'loaded'">
      <b-row>
        <b-col md="12">
          <header>
            <div>
              <b-breadcrumb :items="breadcrumbs" />
            </div>
          </header>
        </b-col>
      </b-row>
      <b-row>
        <b-col md="8">
          <h1 class="scroll">{{ title }}</h1>
          <p class="scroll">
            <small>
              <a :href="url" target="_blank"><code>{{ url }}</code></a>
            </small>
          </p>
          <b-tabs :value="tabIndex" @input="selectTab" @activate-tab="activateTab">
            <b-tab v-if="this.cogs.length" title="Preview">
              <div id="map-container">
                <div id="map"></div>
              </div>
              <MultiSelect
                v-if="cogs.length > 1"
                v-model="selectedImage"
                :options="cogs"
                placeholder="Select an image"
                track-by="href"
                label="title"
                open-direction="bottom"
              />
              <MultiSelect
                v-if="features.length > 0"
                v-model="selectedFeatures"
                :options="features"
                placeholder="Select a set of features"
                track-by="href"
                label="title"
                open-direction="bottom"
              />
            </b-tab>
            <b-tab v-if="this.thumbnail" title="Thumbnail">
              <a :href="thumbnail">
                <img id="thumbnail" align="center" :src="thumbnail" />
              </a>
            </b-tab>
            <b-tab v-if="this.assets.length" title="Assets" key="assets">
              <AssetTab :assets="assets" :bands="bands" :has-bands="hasBands" />
            </b-tab>
            <b-tab v-if="this.shownLinks.length" title="Links" key="links">
              <LinkTab :links="shownLinks" />
            </b-tab>
            <b-tab v-if="this.bands.length > 0" title="Bands">
              <b-table :items="bands" :fields="bandFields" responsive small striped />
            </b-tab>
          </b-tabs>
        </b-col>

        <b-col md="4">
          <b-card bg-variant="light">
            <div id="locator-map" />
            <MetadataSidebar
              :properties="properties"
              :keywords="keywords"
              :collection="collection"
              :collection-link="collectionLink"
              :providers="providers"
              :slugify="slugify"
            />
          </b-card>
        </b-col>
      </b-row>
    </b-container>
    <footer class="footer">
      <b-container>
        <span class="poweredby text-muted">
          Powered by <a href="https://github.com/radiantearth/stac-browser">STAC Browser</a> v{{ browserVersion }}
        </span>
      </b-container>
    </footer>
  </div>
</template>

<script>
import url from "url";

import * as d3 from "d3-scale-chromatic";
import isEqual from "lodash.isequal";
import Leaflet from "leaflet";
import "leaflet-easybutton";
import { mapActions, mapGetters } from "vuex";

import common from "./common";
import { getTileSource } from "../util";
import Migrate from '@radiantearth/stac-migrate';

// ToDo 3.0: Import tabs lazily
import LinkTab from "./LinkTab.vue";
import AssetTab from "./AssetTab.vue";

const COG_TYPES = [
  "image/vnd.stac.geotiff; cloud-optimized=true",
  "image/x.geotiff", // generated by sat-api
  "image/tiff; application=geotiff; profile=cloud-optimized"
];

const FEATURES_TYPES = ["application/geo+json"];

export default {
  ...common,
  name: "ItemDetail",
  components: {
    LinkTab,
    AssetTab,
    MetadataSidebar: () => import(/* webpackChunkName: "metadata-sidebar" */ "./MetadataSidebar.vue"),
    MultiSelect: () => import(/* webpackChunkName: "multiselect" */ 'vue-multiselect')
  },
  props: {
    ancestors: {
      type: Array,
      required: true
    },
    center: {
      type: Array,
      default: null
    },
    path: {
      type: String,
      required: true
    },
    resolve: {
      type: Function,
      required: true
    },
    slugify: {
      type: Function,
      required: true
    },
    url: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      stacVersion: null,
      fullscreen: false,
      locatorMap: null,
      map: null,
      featuresLayer: null,
      tileLayer: null,
      overlayLayer: null,
      locatorOverlayLayer: null,
      geojsonOptions: {
        style: function() {
          return {
            weight: 2,
            color: "#333",
            opacity: 1,
            fillOpacity: 0
          };
        }
      },
      selectedFeatures: null,
      selectedImage: null,
      tabIndex: 0,
      tabsChanged: false
    };
  },
  computed: {
    ...common.computed,
    ...mapGetters(["getEntity"]),
    _entity() {
      let object = this.getEntity(this.url);
      if (object instanceof Error) {
        return object;
      }
      else if (object.type === "FeatureCollection") {
        const { hash } = url.parse(this.url);
        const idx = hash.slice(1);
        object = object.features[idx];
      }
      this.stacVersion = object.stac_version; // Store the original stac_version as it gets replaced by the migration
      let cloned = JSON.parse(JSON.stringify(object)); // Clone to avoid changing the vuex store, remove once migration is done directly in vuex
      return Migrate.item(cloned);
    },
    _collectionLinks() {
      return this.links.filter(x => x.rel === "collection");
    },
    _description() {
      return this.properties.description;
    },
    keywords() {
      if (this.collection && Array.isArray(this.collection.keywords)) {
        return this.collection.keywords;
      }
      else {
        return common.computed.keywords.apply(this)
      }
    },
    _license() {
      return (
        this.properties.license ||
        (this.collection && this.collection.license) ||
        (this.rootCatalog && this.rootCatalog.license)
      );
    },
    providers() {
      return (
        this.properties.providers ||
        (this.collection && this.collection.providers) ||
        common.computed.providers.apply(this)
      );
    },
    _temporalCoverage() {
      if (this.properties.start_datetime != null) {
        return [
          this.properties.start_datetime,
          this.properties.end_datetime
        ]
          .map(x => x || "..")
          .join("/");
      }

      return this.properties.datetime;
    },
    _title() {
      return this.properties.title;
    },
    attribution() {
      if (this.license != null || this.licensor != null) {
        return `Imagery ${this.license || ""} ${this.licensor || ""}`;
      }

      return null;
    },
    bands() {
      // ToDo: Merge all bands from assets
      return (
        this.properties["eo:bands"] ||
        (this.collection &&
          this.collection.properties &&
          this.collection.properties["eo:bands"]) ||
        []
      );
    },
    cog() {
      return (
        this.cogs
          .slice()
          // prefer COGs with "visual" key
          .sort((a, b) => b.key.indexOf("visual") - a.key.indexOf("visual"))
          .shift()
      );
    },
    cogs() {
      return this.assets
        .filter(x => COG_TYPES.includes(x.type))
        .map(cog => ({
          ...cog,
          title:
            cog.bandNames.length > 0
              ? `${cog.title} (${cog.bandNames})`
              : cog.title
        }))
        .map(cog => ({
          ...cog,
          title: `Image: ${cog.title}`
        }));
    },
    collection() {
      if (this.collectionLink != null) {
        this.load(this.collectionLink.href);

        return this.getEntity(this.collectionLink.href);
      }

      return null;
    },
    collectionLink() {
      return this._collectionLinks
        .map(x => ({
          ...x,
          href: this.resolve(x.href, this.url),
          slug: this.slugify(this.resolve(x.href, this.url))
        }))
        .pop();
    },
    entity() {
      return this.item;
    },
    features() {
      return this.assets
        .filter(x => FEATURES_TYPES.includes(x.type))
        .map(features => ({
          ...features,
          title: `Features: ${features.title}`
        }));
    },
    featuresSource() {
      if (this.selectedFeatures == null) {
        return "";
      }

      return this.selectedFeatures.href;
    },
    item() {
      return this._entity;
    },
    jsonLD() {
      const dataset = {
        "@context": "https://schema.org/",
        "@type": "Dataset",
        // required
        name: this.title,
        description: this.description || `${this.title} STAC Item`,
        // recommended
        citation: this.properties["sci:citation"],
        identifier: this.properties["sci:doi"] || this.item.id,
        keywords: this.keywords,
        license: this.licenseUrl,
        isBasedOn: this.url,
        url: this.path,
        workExample: (this.properties["sci:publications"] || []).map(p => ({
          identifier: p.doi,
          citation: p.citation
        })),
        includedInDataCatalog: [this.collectionLink, this.parentLink]
          .filter(x => !!x)
          .map(l => ({
            isBasedOn: l.href,
            url: l.slug
          })),
        spatialCoverage: {
          "@type": "Place",
          geo: {
            "@type": "GeoShape",
            box: (this.item.bbox || []).join(" ")
          }
        },
        temporalCoverage: this._temporalCoverage,
        distribution: this.assets.map(a => ({
          contentUrl: a.href,
          fileFormat: a.type,
          name: a.title
        })),
        image: this.thumbnail
      };

      return dataset;
    },
    licensor() {
      return this.providers
        .filter(x => (x.roles || []).includes("licensor"))
        .map(x => {
          if (x.url != null) {
            return `<a href=${x.url}>${x.name}</a>`;
          }

          return x.name;
        })
        .pop();
    },
    parentLink() {
      return this.links
        .filter(x => x.rel === "parent")
        .map(x => ({
          ...x,
          href: this.resolve(x.href, this.url),
          slug: this.slugify(this.resolve(x.href, this.url))
        }))
        .pop();
    },
    properties() {
      return this.entity.properties || {};
    },
    tileSource() {
      if (this.selectedImage == null) {
        return "";
      }

      return getTileSource(this.selectedImage.href);
    }
  },
  watch: {
    ...common.watch,
    featuresSource(to, from) {
      if (to !== from) {
        if (this.map != null) {
          this.initializeFeaturesLayer();
        }
      }
    },
    fullscreen(to, from) {
      if (to !== from) {
        if (this.map != null) {
          if (to) {
            this.map.getContainer().classList.add("leaflet-pseudo-fullscreen");
          } else {
            this.map
              .getContainer()
              .classList.remove("leaflet-pseudo-fullscreen");
          }
        }

        this.updateState({
          fullscreen: to
        });

        if (this.map != null) {
          this.map.invalidateSize();
        }
      }
    },
    selectedFeatures(to, from) {
      if (!isEqual(to, from)) {
        const idx = this.features.indexOf(to);

        if (idx >= 0) {
          this.updateState({
            sf: idx
          });
        } else {
          this.updateState({
            sf: null
          });
        }
      }
    },
    selectedImage(to, from) {
      if (!isEqual(to, from)) {
        const idx = this.cogs.indexOf(to);

        if (idx >= 0) {
          this.updateState({
            si: idx
          });
        } else {
          this.updateState({
            si: null
          });
        }
      }
    },
    tileSource(to, from) {
      if (to !== from) {
        if (this.map != null) {
          if (this.tileLayer != null) {
            this.tileLayer.removeFrom(this.map);
          }

          this.tileLayer = Leaflet.tileLayer(to, {
            attribution: this.attribution
          }).addTo(this.map);
        }
      }
    }
  },
  methods: {
    ...common.methods,
    ...mapActions(["load"]),
    initialize() {
      this.syncWithQueryState();

      this.$nextTick(() => {
        if (this.cog != null) {
          this.selectedImage = this.selectedImage || this.cog;

          this.initializePreviewMap();
        }
        this.initializeLocatorMap();
      });
    },
    initializeFeaturesLayer() {
      if (this.featuresLayer != null) {
        this.featuresLayer.removeFrom(this.map);
      }

      this.featuresLayer = Leaflet.geoJSON(null, {
        onEachFeature: (feature, layer) =>
          layer.bindPopup(() => {
            const el = document.createElement("table");

            const labelProperties = this.properties["label:properties"] || [];

            el.innerHTML = Object.entries(feature.properties)
              .filter(([k]) =>
                labelProperties.length > 0 ? labelProperties.includes(k) : true
              )
              .map(
                ([k, v]) =>
                  `<tr><td><strong>${k}</strong></td><td><code>${v}</code></td></tr>`
              )
              .join("");

            return el;
          }),
        style: feature => {
          const labelClasses = this.properties["label:classes"];
          const classes = (labelClasses || [])
            .map(x => x.classes.map(c => `${x.name}-${c}`))
            .flat();
          const classification = (
            labelClasses
              .map(x => [x.name, feature.properties[x.name]])
              .find(x => x[1] != null) || []
          ).join("-");
          const color =
            d3.schemeTableau10[
              classes.indexOf(classification) % d3.schemeTableau10.length
            ] || "#fc0";

          return {
            color,
            weight: 1,
            opacity: 1,
            fillOpacity: 0.25
          };
        }
      }).addTo(this.map);

      this.$nextTick(async () => {
        try {
          const rsp = await fetch(this.featuresSource);
          const data = await rsp.json();
          this.featuresLayer.addData(data);
        } catch (err) {
          console.warn(err);
        }
      });
    },
    initializeLocatorMap() {
      if (this.locatorMap == null) {
        this.locatorMap = Leaflet.map("locator-map", {
          attributionControl: false,
          zoomControl: false,
          boxZoom: false,
          doubleClickZoom: false,
          dragging: false,
          scrollWheelZoom: false,
          touchZoom: false
        });

        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_nolabels/{z}/{x}/{y}@2x.png",
          {
            attribution: `Map data <a href="https://www.openstreetmap.org/copyright">&copy; OpenStreetMap contributors</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>`
          }
        ).addTo(this.locatorMap);
        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_only_labels/{z}/{x}/{y}@2x.png",
          {
            opacity: 0.6,
            zIndex: 1000
          }
        ).addTo(this.locatorMap);
      } else {
        this.locatorOverlayLayer.removeFrom(this.locatorMap);
      }

      this.locatorOverlayLayer = Leaflet.geoJSON(this.item, {
        pane: "tilePane",
        style: {
          weight: 1,
          color: "#ffd65d",
          opacity: 1,
          fillOpacity: 1
        }
      }).addTo(this.locatorMap);

      this.locatorMap.fitBounds(this.locatorOverlayLayer.getBounds(), {
        padding: [95, 95]
      });
    },
    initializePreviewMap() {
      if (this.map == null) {
        this.map = Leaflet.map("map", {
          scrollWheelZoom: false
        });

        this.map.on("moveend", this.updateHash);
        this.map.on("zoomend", this.updateHash);

        this.map.attributionControl.setPrefix("");

        this.button = Leaflet.easyButton(
          "fas fa-expand fa-2x",
          () => (this.fullscreen = !this.fullscreen),
          {
            position: "topright"
          }
        ).addTo(this.map);

        if (this.fullscreen) {
          this.map.getContainer().classList.add("leaflet-pseudo-fullscreen");
        }

        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_nolabels/{z}/{x}/{y}@2x.png",
          {
            attribution: `Map data <a href="https://www.openstreetmap.org/copyright">&copy; OpenStreetMap contributors</a>, &copy; <a href="https://carto.com/attributions">CARTO</a>`
          }
        ).addTo(this.map);
        Leaflet.tileLayer(
          "https://{s}.basemaps.cartocdn.com/dark_only_labels/{z}/{x}/{y}@2x.png",
          {
            zIndex: 1000
          }
        ).addTo(this.map);
      } else {
        if (this.tileLayer != null) {
          this.tileLayer.removeFrom(this.map);
        }

        if (this.featuresLayer != null) {
          this.featuresLayer.removeFrom(this.map);
        }

        this.overlayLayer.removeFrom(this.map);
      }

      if (this.tileSource) {
        this.tileLayer = Leaflet.tileLayer(this.tileSource, {
          attribution: this.attribution,
          maxZoom: 22
        }).addTo(this.map);
      }

      if (this.featuresSource) {
        this.initializeFeaturesLayer();
      }

      this.overlayLayer = Leaflet.geoJSON(this.item, {
        ...this.geojsonOptions,
        pane: "tilePane"
      }).addTo(this.map);

      if (this.center != null) {
        const [zoom, lat, lng] = this.center;

        this.map.setView([lat, lng], zoom);
      } else {
        this.map.fitBounds(this.overlayLayer.getBounds());
      }
    },
    selectTab(tabIndex) {
      if (typeof tabIndex !== 'number') {
        tabIndex = parseInt(tabIndex, 10);
        if (isNaN(tabIndex)) {
          return;
        }
      }
      if (this.tabsChanged && this.tabIndex !== tabIndex) {
        this.updateState({
          t: tabIndex
        });
        if (tabIndex === 0) {
          this.map && this.map.invalidateSize();
        }
      }
      this.$nextTick(() => {
        this.tabIndex = tabIndex;
      });
    },
    activateTab() {
      this.tabsChanged = true;
    },
    syncWithQueryState() {
      this.selectTab(this.$route.query.t);

      if (this.$route.query.si != null) {
        this.selectedImage = this.cogs[this.$route.query.si];
      } else {
        this.selectedImage = this.cog;
      }

      if (this.$route.query.sf != null) {
        this.selectedFeatures = this.features[this.$route.query.sf];
      }

      this.fullscreen = [true, "true"].includes(this.$route.query.fullscreen);
    },
    async updateHash() {
      const center = this.map.getCenter();
      const zoom = this.map.getZoom();
      const hash = `${zoom}/${center.lat.toFixed(6)}/${center.lng.toFixed(6)}`;

      if (isEqual(this.$route.hash, `#${hash}`)) {
        return;
      }

      try {
        await this.$router.replace({
          ...this.$route,
          hash
        });
      } catch (err) {
        console.warn(err);
      }
    }
  }
};
</script>

<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
<style src="./base.css"></style>

<style scoped lang="css">
.leaflet-pseudo-fullscreen {
  position: fixed !important;
  width: 100% !important;
  height: 100% !important;
  top: 0 !important;
  left: 0 !important;
  z-index: 99999;
}

.leaflet-container {
  background-color: #262626;
}

#locator-map {
  height: 200px;
  width: 100%;
  margin-bottom: 10px;
}

#map-container {
  height: 500px;
}

#map {
  height: 100%;
  width: 100%;
}

#header_logo {
  max-width: 100px;
}

.table-responsive.assets {
  padding: 15px;
}

.multiselect {
  margin-top: 5px;
}
</style>
