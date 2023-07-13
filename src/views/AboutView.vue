<script>
import { Loader } from '@googlemaps/js-api-loader'
export default {
  data() {
    return {
      listItems: []
    }
  },
  methods: {
    async getData() {
      const loader = new Loader({
        apiKey: 'AIzaSyCRiINOpnvk-p9m7AJIhxPTbuAQif7L4n4',
        version: '',
        libraries: ['places']
      })

      try {
        const google = await loader.load()
        const mapOptions = {
          center: {
            lat: 0,
            lng: 0
          },
          zoom: 4
        }

        var coco = new google.maps.LatLng(41.40807945494184, 2.1554292820665517)
        // const map = new google.maps.Map(document.getElementById('map'), {
        //   center: coco,
        //   zoom: 15
        // })
        var request = {
          location: coco,
          radius: '1500',
          type: [''],
        }
        let place_request = {
          placeId: "", 
          fields: ['business_status', 'name', 'type', 'website', 'formatted_phone_number']
        }

        var service = new google.maps.places.PlacesService(map)
        service.nearbySearch(request, callback);
        function callback(results, status) {
          console.log(results);
          for (let i = 0; i < results.length; i++) {
            place_request.placeId = results[i].place_id;
            console.log(place_request);
            service.getDetails(place_request, callbackPlace);

          }

          // if (status == google.maps.places.PlacesServiceStatus.OK) {
          //   for (var i = 0; i < results.length; i++) {
          //     createMarker(results[i])
          //   }
          // }
        }
         const callbackPlace = (result, status) => {
          console.log(result);
          this.listItems.push(result);
        }
      } catch (error) {
        console.log(error)
      }
      // function createMarker(place) {
      //   console.log(place)
      //   google.maps.Marker({
      //     position: place.geometry.location,
      //     map: map
      //   })
      // }
    }
  }
}
</script>
<template>
  <div class="about">
    <h1>Get Business</h1>
    <button class="business-btn" @click="getData">Get Data</button>
    <div>
      <div>
        <div v-for="item in listItems">
        <ul class="business">
          <li>Nombre{{ item.name }}</li>
          <li>Estatus {{ item.business_status }}</li>
          <li>Sitio web: <a src='{{item.website}}'>{{ item.website }}</a></li>
          <li>Tipo: <span v-for="item in item.types">, {{ item }}</span>.</li>
          <li>Telefono: {{ item.formatted_phone_number }}</li>
        </ul>
        </div>
      </div>
      <div id="map"></div>
      <!-- Rest of your template -->
    </div>
  </div>
</template>

<style>
@media (min-width: 1024px) {
  .about {
    min-height: 100vh;
    display: flex;
    align-items: center;
    flex-direction: column;
  }
  .business-btn {
    padding: 10px 20px;
    border-radius: 10px;
    background-color: rgb(65, 218, 218);
  }
  .business {
    border: 2px solid rgb(255, 166, 0);
    border-radius: 5px;
    margin: 10px 0;
  }
}
</style>
