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
          if (status === google.maps.places.PlacesServiceStatus.OK && result) {
            // Only push if not already in listItems
            const exists = this.listItems.some(item => item.place_id === result.place_id)
            if (!exists) {
              this.listItems.push(result)

              if (result.website) {
                this.fetchAsync(result.website)
              }
            }
          }
        }
        for (let i = 0; i < 2; i++) {
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
    },
    async fetchAsync(webUrl) {
      console.log(webUrl)
      const bodyObject = {
        url: webUrl,
        requestScreenshot: true
      }
      let response = await fetch(
        'https://searchconsole.googleapis.com/v1/urlTestingTools/mobileFriendlyTest:run?key=AIzaSyBMqGteA4DXEDJuWBWUa0YZtxx3xn0k_hk',
        {
          method: 'POST', // *GET, POST, PUT, DELETE, etc.
          mode: 'no-cors', // no-cors, *cors, same-origin
          cache: 'no-cache', // *default, no-cache, reload, force-cache, only-if-cached
          credentials: 'same-origin', // include, *same-origin, omit
          headers: {
            'Content-Type': 'application/json'
            // 'Content-Type': 'application/x-www-form-urlencoded',
          },
          redirect: 'follow', // manual, *follow, error
          referrerPolicy: 'no-referrer',
          body: JSON.stringify(bodyObject)
        }
      )
      // let data = await response.json()
      console.log(response)
      // return data
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
          <span style="font-weight: bold">Sitio web: </span><a class="website-link" rel="noopener noreferrer" href="{{ item.website }}"
            target="_blank">{{ item.website }}</a>
        </li>
        <li>
          <span style="font-weight: bold">Tipo: </span><span v-for="item in item.types">, {{ item }}</span>.
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
