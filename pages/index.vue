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
    <div v-cloak id="app" style="height: 100vh; width: 100%">
      <l-map
        v-if="showMap"
        :zoom="zoom"
        :center="center"
        :options="mapOptions"
        style="height: 100%"
      >
        <l-tile-layer :url="url" :attribution="attribution"></l-tile-layer>
        <l-control-scale
          position="topright"
          :imperial="false"
          :metric="true"
        ></l-control-scale>
        <l-geo-json
          :geojson="geojson"
          :options-style="styleFunction"
        ></l-geo-json>
        <!-- <l-circle
          :lat-lng="center"
          :radius="circle.radius"
          :stroke="circle.stroke"
          :color="circle.color"
          :weight="circle.weight"
          :fill-color="circle.fillColor"
        ></l-circle> -->
        <!-- <l-circle-marker
          :lat-lng="updatedCenter"
          :radius="circlemarker.radius"
          :stroke="circle.stroke"
          :fill-color="circlemarker.fillColor"
          :fill-opacity="circlemarker.fillOpacity"
        >
        </l-circle-marker> -->
        <l-marker
          :lat-lng.sync="center"
          :radius="circlemarker.radius"
          :stroke="circle.stroke"
          :draggable="draggable"
          :fill-color="circlemarker.fillColor"
          :fill-opacity="circlemarker.fillOpacity"
        >
          <l-icon
            :icon-size="dynamicSize"
            :icon-anchor="dynamicAnchor"
            icon-url="/alfiler.png"
          >
          </l-icon>
        </l-marker>
        <l-marker
          v-if="nearestBin && nearestBin.lat"
          :lat-lng.sync="nearestBin"
          :radius="circlemarker.radius"
          :stroke="circle.stroke"
          :draggable="draggable"
          :fill-color="circlemarker.fillColor"
          :fill-opacity="circlemarker.fillOpacity"
        >
          <l-icon
            :icon-size="dynamicSize"
            :icon-anchor="dynamicAnchor"
            :tooltip-anchor="tooltipAnchor"
            icon-url="/basura.png"
          />
          <l-tooltip :options="{ permanent: true, direction: 'top' }">
            {{ parseInt(nearestBin.distance) }} m.
          </l-tooltip>
        </l-marker>
      </l-map>
    </div>
  </div>
</template>

<script>
import Papa from 'papaparse'
import { LMap, LTileLayer, LMarker, LControlScale, LIcon } from 'vue2-leaflet'
import { latLng } from 'leaflet'

export default {
  components: {
    LMap,
    LTileLayer,
    LMarker,
    LControlScale,
    LIcon
  },
  data: () => ({
    allBins: [],
    zoom: 17,
    center: latLng(39.4559488, -0.36372479999999996),
    updatedCenter: latLng(39.4559488, -0.36372479999999996),
    draggable: false,
    geojson: null,
    url:
      'https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png',
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
    circle: {
      center: latLng(39.4559488, -0.36372479999999996),
      radius: 1000,
      color: '#1CD89A',
      weight: 2,
      fillColor: '#1CD89A'
    },
    circlemarker: {
      center: latLng(39.4559488, -0.36372479999999996),
      radius: 8,
      stroke: false,
      fillColor: '#006DDC',
      fillOpacity: 1
    },
    mapOptions: {
      zoomSnap: 0.5
    },
    iconSize: 34,
    showMap: true,
    nearestBin: { lat: '', lng: '' }
  }),
  computed: {
    dynamicSize() {
      return [this.iconSize, this.iconSize * 1.15]
    },
    dynamicAnchor() {
      return [this.iconSize / 2, this.iconSize * 1.3]
    },
    tooltipAnchor() {
      return [0, -40]
    },
    styleFunction() {
      const fillColor = this.fillColor
      return () => {
        return {
          weight: 2,
          color: '#1CD89A',
          opacity: 1,
          fillColor,
          fillOpacity: 0.3
        }
      }
    }
  },
  created() {
    this.initData()
  },
  mounted() {
    this.geolocate()
    this.watchposition()
  },
  methods: {
    geolocate() {
      navigator.geolocation.getCurrentPosition((position) => {
        this.center = {
          lat: position.coords.latitude,
          lng: position.coords.longitude
        }
      })
    },
    watchposition() {
      navigator.geolocation.watchPosition((position) => {
        this.updatedCenter = {
          lat: position.coords.latitude,
          lng: position.coords.longitude
        }
      })
    },
    async initData() {
      const binsData = await this.$axios.get(
        'http://mapas.valencia.es/lanzadera/opendata/Res_papeleras/CSV'
      )
      const bins = binsData.data
      this.formatBins(bins)
    },
    formatBins(allBins) {
      const bins = Papa.parse(allBins, { header: true, delimiter: ';' })
      this.allBins = bins.data
      this.getNearestBin()
    },
    async getNearestBin() {
      const { data } = await this.convertToUTM(this.center.lat, this.center.lng)
      this.nearestBin = this.findShortestDistance(
        {
          X: data[0].easting,
          Y: data[0].northing
        },
        this.allBins
      )
      const binWithLatLng = await this.convertToLatLng(
        this.nearestBin.X,
        this.nearestBin.Y
      )
      this.nearestBin = {
        ...this.nearestBin,
        ...latLng(binWithLatLng.data.srcLat, binWithLatLng.data.srcLon)
      }
    },
    findShortestDistance(myPosition, positions) {
      let shortestDistance = this.calculateDistance(myPosition, positions[0])
      let nearestBinPosition = positions[0]
      positions.forEach((position) => {
        const distance = this.calculateDistance(myPosition, position)
        if (distance < shortestDistance) {
          shortestDistance = distance
          nearestBinPosition = position
        }
      })
      return { ...nearestBinPosition, distance: shortestDistance }
    },
    calculateDistance(pointOne, pointTwo) {
      const diffX = parseFloat(pointOne.X) - parseFloat(pointTwo.X)
      const diffY = parseFloat(pointOne.Y) - parseFloat(pointTwo.Y)

      return Math.sqrt(diffX * diffX + diffY * diffY)
    },
    async convertToUTM(lat, lng) {
      try {
        return await this.$axios.get(
          `https://www.latlong.net/dec2utm.php?lat=${lat}&long=${lng}`
        )
      } catch (error) {
        console.error(error)
      }
    },
    async convertToLatLng(pointX, pointY) {
      try {
        return await this.$axios.get(
          `https://geodesy.noaa.gov/api/ncat/utm?utmZone=30&northing=${pointY}&easting=${pointX}&hemi=N&a=6378160.0&invf=298.25`
        )
      } catch (error) {
        console.error(error)
      }
    }
  }
}
</script>

<style>
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
  font-size: 60px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 22px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.leaflet-tooltip {
  margin-top: -20px;
}
</style>
