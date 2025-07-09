<script>
import { Loader } from '@googlemaps/js-api-loader'
export default {
  data() {
    return {
      listItems: [],
      websiteStatusResult: [],
      websiteAsyncStatus: [],
      promiseResult: [],
      map: '',
      service: '',
      infoWindow: '',
      dataReady: false,
      defaultLocation: {},
      lat: 41.408165706775655,
      lng: 2.155339378508503,
      mobileFriendly: 'responsive',
      notMobileFriendly: 'not-responsive',
      selectedType: 'hotel',
    }
  },
  async mounted() {
    try {
      const loader = new Loader({
        apiKey: `${import.meta.env.VITE_MAP_API_KEY}`,
        version: 'weekly',
        libraries: ['places']
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
        this.dataReady = true
        const { lat, lng } = mapsMouseEvent.latLng.toJSON()
        this.lat = lat
        this.lng = lng
        // Create a new InfoWindow.
        this.infoWindow = new google.maps.InfoWindow({
          position: mapsMouseEvent.latLng
        })
        this.infoWindow.setContent(JSON.stringify(mapsMouseEvent.latLng.toJSON(), null, 2))
        this.infoWindow.open(this.map)
        this.findBusiness()
        this.infoWindow.close()
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
    radius: '1000',
    type: this.selectedType || undefined
  }
  let place_request = {
    placeId: '',
    fields: ['business_status', 'name', 'type', 'website', 'formatted_phone_number', 'geometry']
  }
  this.service = new google.maps.places.PlacesService(this.map)
  const callback = (results, status) => {
    let totalAmount = 0
    const callbackPlace = (result, status) => {
      totalAmount += 1
      if (status === google.maps.places.PlacesServiceStatus.OK) {
        if (!result.website) {
          this.listItems.push(result)
          // Optional: Place marker only for businesses without websites
          if (result.geometry && result.geometry.location) {
            new google.maps.Marker({
              map: this.map,
              position: result.geometry.location
            })
          }
        }
      }
      if (totalAmount === results.length) {
        this.fetchAsync()
      }
    }
    for (let i = 0; i < results.length; i++) {
      place_request.placeId = results[i].place_id
      this.service.getDetails(place_request, callbackPlace)
    }
  }
  this.service.nearbySearch(request, callback)
},
    async fetchAsync() {
      this.websiteAsyncStatus = this.listItems.map(async (item) => {
        const text = item.website ? item.website : 'https//www. .net'
        const replacedText = text.replace(/http(?=:\/\/)/g, 'https')
        const bodyObject = {
          url: replacedText,
          requestScreenshot: true
        }
        let response = await fetch(
          `https://searchconsole.googleapis.com/v1/urlTestingTools/mobileFriendlyTest:run?key=${import.meta.env.VITE_MAP_API_KEY
          }`,
          {
            method: 'POST', // *GET, POST, PUT, DELETE, etc.
            headers: {
              'Content-Type': 'application/json'
              // 'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: JSON.stringify(bodyObject)
          }
        )
        const websiteData = await response.json()
        return websiteData

        // return data
      })
      // if (this.listItems.length === this.websiteStatus.length) {
      const resolvedPromises = await Promise.all(this.websiteAsyncStatus)
      if (resolvedPromises) {
        this.dataReady = false
        this.websiteStatusResult = resolvedPromises
        this.websiteStatusResult.forEach((siteStatus, index) => {
          Object.assign(this.listItems[index], siteStatus)
        })
      }

      // }
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
    <div class="loading-wrapper" v-if="dataReady">
      <button class="buttonload"><i class="fa fa-spinner fa-spin"></i></button>
    </div>
    <select v-model="selectedType">
        <option value="">All Types</option>
        <option value="restaurant">Restaurant</option>
        <option value="cafe">Cafe</option>
        <option value="store">Store</option>
        <option value="gym">Gym</option>
        <option value="lawyer">Lawyer</option>
        <option value="school">School</option>
        <option value="hotel">Hotel</option>
        <option value="dentist">Dentist</option>
        <option value="car_repair">Car Repair</option>
        <!-- Add more as needed -->
      </select>
    <div class="business-list-container">
      <ul class="business" v-for="item in listItems">
        <li><span style="font-weight: bold">Nombre: </span>{{ item.name }}</li>
        <li><span style="font-weight: bold">Estatus: </span>{{ item.business_status }}</li>
        <li>
          <span style="font-weight: bold">Sitio web: </span><a class="website-link"
            :class="[item.mobileFriendliness === 'MOBILE_FRIENDLY' ? 'responsive' : '']" href="{{ item.website }}"
            style="{{ color:  }}" target="blank_">{{ item.website }}</a>
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

.responsive {
  color: rgb(236, 91, 83);
}

.loading-wrapper {
  text-align: center;
}

.buttonload {
  background-color: #04aa6d;
  border: none;
  color: white;
  padding: 24px;
  font-size: 28px;
  border-radius: 35px;
  margin-top: 10px;
}

.fa {
  margin-left: 8px;
  margin-right: 8px;
}
</style>
