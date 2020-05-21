<template>
  <div class="container">
    <div class="info">
      <h1 class="title">
        Nearest Bin
      </h1>
      <h2 class="subtitle">
        Find your nearest bin 🗑
      </h2>
    </div>
    <l-map
      ref="map"
      style="min-height: 100vh"
      :zoom="zoom"
      :center="currentPosition"
      :options="{ zoomControl: false }"
    >
      <l-tile-layer :url="url"></l-tile-layer>
    </l-map>
  </div>
</template>

<script>
import Papa from 'papaparse'
import { LMap, LTileLayer } from 'vue2-leaflet'

export default {
  components: {
    LMap,
    LTileLayer
  },
  data: () => ({
    allBins: [],
    url: 'http://{s}.tile.osm.org/{z}/{x}/{y}.png',
    zoom: 20,
    currentPosition: [39.4559488, -0.36372479999999996]
  }),
  created() {
    this.initData()
  },
  mounted() {
    // to access object map --> this.$refs.map.mapObject

    if ('geolocation' in navigator) {
      navigator.geolocation.getCurrentPosition((position) => {
        this.currentPosition = [
          position.coords.latitude,
          position.coords.longitude
        ]
      })
    }
  },
  methods: {
    async initData() {
      const binsData = await this.$axios.get(
        'http://mapas.valencia.es/lanzadera/opendata/Res_papeleras/CSV'
      )
      this.allBins = binsData.data
      this.formatBins(this.allBins)
    },
    formatBins(allBins) {
      // eslint-disable-next-line no-console
      console.log(Papa.parse(allBins, { header: true, delimiter: ';' }))
    }
  }
}
</script>

<style scoped>
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  flex-direction: column;
}
.info {
  position: absolute;
  top: 0;
  z-index: 1000;
}

.title {
  font-family: 'Quicksand', 'Source Sans Pro', -apple-system, BlinkMacSystemFont,
    'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}

#map {
  height: 180px;
}
</style>
