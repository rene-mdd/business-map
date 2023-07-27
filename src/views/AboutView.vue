<script>
import { Loader } from '@googlemaps/js-api-loader'
// const { Marker } = await google.maps.importLibrary('marker')
export default {
  data() {
    return {
      listItems: [],
      map: '',
      service: '',
      infoWindow: '',
      dataReady: false,
      defaultLocation: {},
      lat: 52.524697,
      lng: 13.442576
    }
  },
  async mounted() {
    try {
      const loader = new Loader({
        apiKey: `${import.meta.env.VITE_MAP_API_KEY}`,
        version: '',
        libraries: ['places', 'marker']
      })
      const google = await loader.load()
      this.defaultLocation = new google.maps.LatLng(this.lat, this.lng)
      this.map = new google.maps.Map(document.getElementById('map'), {
        center: this.defaultLocation,
        zoom: 15,
        mapTypeId: google.maps.MapTypeId.ROADMAP
      })
      this.infoWindow = new google.maps.InfoWindow({
        content: 'Click the map to get Lat/Lng!',
        position: new google.maps.LatLng(this.lat, this.lng)
      })

      this.map.addListener('click', (mapsMouseEvent) => {
        // Close the current InfoWindow.

        const { lat, lng } = mapsMouseEvent.latLng.toJSON()
        this.lat = lat
        this.lng = lng
        this.infoWindow.close()
        // Create a new InfoWindow.
        this.infoWindow = new google.maps.InfoWindow({
          position: mapsMouseEvent.latLng
        })
        this.infoWindow.setContent(JSON.stringify(mapsMouseEvent.latLng.toJSON(), null, 2))
        this.infoWindow.open(this.map)
        this.findBusiness()
      })
    } catch (error) {
      console.error(error)
    }
  },
  methods: {
    createMarker(place) {
      console.log(place)
      if (!place.geometry || !place.geometry.location) return
    },
    findBusiness() {
      let request = {
        location: { lat: this.lat, lng: this.lng },
        radius: '1500',
        type: ['']
      }
      let place_request = {
        placeId: '',
        fields: ['business_status', 'name', 'type', 'website', 'formatted_phone_number']
      }
      this.service = new google.maps.places.PlacesService(this.map)
      const callback = (results, status) => {
        const callbackPlace = (result, status) => {
          this.listItems.push(result)
        }
        for (let i = 0; i < results.length; i++) {
          place_request.placeId = results[i].place_id
          this.service.getDetails(place_request, callbackPlace)
        }

        if (status == google.maps.places.PlacesServiceStatus.OK) {
          for (var i = 0; i < results.length; i++) {
            this.createMarker(results[i])
            new google.maps.Marker({
              map: this.map,
              position: results[i].geometry.location
            })
            this.map.setCenter(results[i].geometry.location)
          }
        }
      }
      this.service.nearbySearch(request, callback)
    }
  }
}
</script>
<template>
  <div class="about">
    <header>
      <h1>Business at Google Maps</h1>
    </header>
    <div id="map"></div>
    <div class="business-list-container">
      <ul class="business" v-for="item in listItems">
        <li><span style="font-weight: bold">Nombre: </span>{{ item.name }}</li>
        <li><span style="font-weight: bold">Estatus: </span>{{ item.business_status }}</li>
        <li>
          <span style="font-weight: bold">Sitio web: </span
          ><a class="website-link" href="{{ item.website }}" target="blank_">{{ item.website }}</a>
        </li>
        <li>
          <span style="font-weight: bold">Tipo: </span
          ><span v-for="item in item.types">, {{ item }}</span
          >.
        </li>
        <li>
          <p style="font-weight: bold">
            Telefono:
            <a href="tel:{{ item.formatted_phone_number }}">{{ item.formatted_phone_number }}</a>
          </p>
        </li>
      </ul>
    </div>
  </div>
</template>

<style>
#map {
  height: 500px;
  width: 100%;
  margin: auto;
}
.about {
  margin: 0 auto;
}
.about header {
  text-align: center;
}
.business-btn {
  padding: 10px 20px;
  margin: 20px 0;
  border-radius: 10px;
  background-color: #32bf84;
  cursor: pointer;
  box-shadow: rgba(50, 50, 93, 0.25) 0px 6px 12px -2px, rgba(0, 0, 0, 0.3) 0px 3px 7px -3px;
  border: none;
  font-size: large;
  color: white;
  font-weight: bold;
}

.business-list-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
}
.business {
  box-shadow: rgba(0, 0, 0, 0.35) 0px 5px 15px;
  border-radius: 5px;
  margin: 10px;
  padding: 10px;
  list-style-type: none;
  width: 400px;
  text-align: left;
  overflow: hidden;
}
.website-link {
  cursor: pointer;
}
</style>
