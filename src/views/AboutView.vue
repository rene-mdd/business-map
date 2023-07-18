<script>
import { Loader } from '@googlemaps/js-api-loader'
export default {
  data() {
    return {
      listItems: [],
      map: '',
      service: '',
      infoWindow: '',
      lat: 52.524697,
      lng: 13.442576,
      dataReady: false
    }
  },
  async mounted() {},
  methods: {
    async getData() {
      const loader = new Loader({
        apiKey: 'AIzaSyCRiINOpnvk-p9m7AJIhxPTbuAQif7L4n4',
        version: '',
        libraries: ['places', 'marker']
      })

      try {
        const google = await loader.load()
        let coco = new google.maps.LatLng(this.lat, this.lng)
        this.map = new google.maps.Map(document.getElementById('map'), {
          center: coco,
          zoom: 15,
          mapTypeId: google.maps.MapTypeId.ROADMAP
        })
        this.infoWindow = new google.maps.InfoWindow({
          content: 'Click the map to get Lat/Lng!',
          position: new google.maps.LatLng(this.lat, this.lng)
        })
        let request = {
          location: coco,
          radius: '1500',
          type: ['']
        }
        let place_request = {
          placeId: '',
          fields: ['business_status', 'name', 'type', 'website', 'formatted_phone_number']
        }

        this.service = new google.maps.places.PlacesService(this.map)

        const callback = (results, status) => {
          for (let i = 0; i < results.length; i++) {
            place_request.placeId = results[i].place_id
            this.service.getDetails(place_request, callbackPlace)
          }

          if (status == google.maps.places.PlacesServiceStatus.OK) {
            for (var i = 0; i < results.length; i++) {
              console.log(results)
            }
            createMarker(results[0])
            this.map.setCenter(results[0].geometry.location)
          }
        }

        const callbackPlace = (result, status) => {
          this.listItems.push(result)
        }

        this.service.nearbySearch(request, callback)

        const createMarker = (place) => {
          if (!place.geometry || !place.geometry.location) return

          this.map.addListener('click', (mapsMouseEvent) => {
            // Close the current InfoWindow.

            const { lat, lng } = mapsMouseEvent.latLng.toJSON()
            this.lat = lat
            this.lng = lng
            console.log(lat)
            this.infoWindow.close()
            // Create a new InfoWindow.
            this.infoWindow = new google.maps.InfoWindow({
              position: mapsMouseEvent.latLng
            })
            this.infoWindow.setContent(JSON.stringify(mapsMouseEvent.latLng.toJSON(), null, 2))
            this.infoWindow.open(this.map)
          })
        }

        this.dataReady = true
      } catch (error) {
        console.error(error)
      }
    }
  }
}
</script>
<template>
  <div class="about">
    <header>
      <h1>Business at Google Maps</h1>
      <button class="business-btn" @click="getData">Get Data</button>
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
          <span style="font-weight: bold">Telefono: </span
          ><a href="tel:{{ item.formatted_phone_number }}">{{ item.formatted_phone_number }}</a>
        </li>
      </ul>
    </div>
  </div>
</template>

<style>
#map {
  height: 500px;
  max-width: 500px;
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
