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
      markers: [], // Track map markers
      dataReady: false,
      defaultLocation: {},
      lat: 41.408165706775655,
      lng: 2.155339378508503,
      mobileFriendly: 'responsive',
      notMobileFriendly: 'not-responsive',
      selectedType: '',
      searchRadius: '500', // Increased default radius
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
        this.dataReady = true
        const { lat, lng } = mapsMouseEvent.latLng.toJSON()
        this.lat = lat
        this.lng = lng

        this.infoWindow = new google.maps.InfoWindow({
          position: mapsMouseEvent.latLng
        })
        this.infoWindow.setContent(JSON.stringify(mapsMouseEvent.latLng.toJSON(), null, 2))
        this.infoWindow.open(this.map)

        this.findBusiness()

        // Close info window after a delay
        setTimeout(() => {
          this.infoWindow.close()
        }, 2000)
      })
    } catch (error) {
      console.error('Error loading Google Maps:', error)
    }
  },

  methods: {
    clearMarkers() {
      // Clear existing markers
      if (this.markers && this.markers.length > 0) {
        this.markers.forEach(marker => marker.setMap(null))
        this.markers = []
      }
    },

    createMarker(place) {
      if (!place.geometry || !place.geometry.location) return

      const marker = new google.maps.Marker({
        map: this.map,
        position: place.geometry.location,
        title: place.name
      })

      this.markers.push(marker)
    },

    findBusiness() {
      // Clear previous results
      this.listItems = []
      this.clearMarkers()

      let request = {
        location: { lat: this.lat, lng: this.lng },
        radius: this.searchRadius,
        type: this.selectedType || undefined
      }

      let place_request = {
        placeId: '',
        fields: ['business_status', 'name', 'types', 'website', 'formatted_phone_number', 'geometry']
      }

      this.service = new google.maps.places.PlacesService(this.map)

      const fetchAllPages = (request, allResults = []) => {
        const callback = (results, status, pagination) => {
          if (status === google.maps.places.PlacesServiceStatus.OK) {
            allResults = allResults.concat(results)

            console.log(`Found ${results.length} results. Total so far: ${allResults.length}`)

            // Check if there are more results
            if (pagination && pagination.hasNextPage) {
              // Wait 2 seconds before next request (Google requirement)
              setTimeout(() => {
                pagination.nextPage()
              }, 2000)
            } else {
              // All results collected, now get details for each
              console.log(`Total results collected: ${allResults.length}`)
              this.processAllResults(allResults, place_request)
            }
          } else if (status === google.maps.places.PlacesServiceStatus.ZERO_RESULTS) {
            console.log('No results found for this location and criteria')
            this.dataReady = false
          } else {
            console.error('Places search failed:', status)
            this.dataReady = false
          }
        }

        this.service.nearbySearch(request, callback)
      }

      fetchAllPages(request)
    },

    processAllResults(results, place_request) {
      let processedCount = 0

      const callbackPlace = (result, status) => {
        if (status === google.maps.places.PlacesServiceStatus.OK) {
          this.listItems.push(result)

          // Create marker for this place
          if (result.geometry && result.geometry.location) {
            this.createMarker({
              geometry: result.geometry,
              name: result.name
            })
          }
        } else {
          console.warn('Failed to get details for place:', status)
        }

        processedCount += 1

        if (processedCount === results.length) {
          console.log(`Processed ${this.listItems.length} businesses with details`)
          this.fetchAsync()
        }
      }

      // Debug: Log the first result to see its structure
      if (results.length > 0) {
        console.log('Sample result structure:', results[0])
        console.log('Available keys:', Object.keys(results[0]))
      }

      // Get details for each place
      for (let i = 0; i < results.length; i++) {
        const currentPlace = results[i]

        // Try different possible place ID properties
        const placeId = currentPlace.place_id || currentPlace.placeId || currentPlace.id

        // Check if place_id exists and is valid
        if (!placeId || typeof placeId !== 'string') {
          console.warn('Invalid or missing place_id for result:', currentPlace)
          console.warn('Place ID value:', placeId, 'Type:', typeof placeId)
          processedCount += 1
          if (processedCount === results.length) {
            console.log(`Processed ${this.listItems.length} businesses with details`)
            this.fetchAsync()
          }
          continue
        }

        // Create a new request object for each place to avoid reference issues
        const individualPlaceRequest = {
          placeId: placeId,
          fields: ['business_status', 'name', 'types', 'website', 'formatted_phone_number', 'geometry']
        }

        console.log(`Making request for place ${i + 1}/${results.length} with ID: ${placeId}`)

        // Add small delay between requests to avoid rate limiting
        setTimeout(() => {
          this.service.getDetails(individualPlaceRequest, callbackPlace)
        }, i * 100) // 100ms delay between requests
      }
    },

    async fetchAsync() {
      if (this.listItems.length === 0) {
        this.dataReady = false
        return
      }

      this.websiteAsyncStatus = this.listItems.map(async (item) => {
        if (!item.website) {
          return { mobileFriendliness: 'NO_WEBSITE' }
        }

        const text = item.website
        const replacedText = text.replace(/http(?=:\/\/)/g, 'https')

        const bodyObject = {
          url: replacedText,
          requestScreenshot: false // Set to false for faster requests
        }

        try {
          let response = await fetch(
            `https://searchconsole.googleapis.com/v1/urlTestingTools/mobileFriendlyTest:run?key=${import.meta.env.VITE_MAP_API_KEY}`,
            {
              method: 'POST',
              headers: {
                'Content-Type': 'application/json'
              },
              body: JSON.stringify(bodyObject)
            }
          )

          if (response.ok) {
            const websiteData = await response.json()
            return websiteData
          } else {
            return { mobileFriendliness: 'ERROR', error: response.status }
          }
        } catch (error) {
          console.error('Error testing mobile friendliness:', error)
          return { mobileFriendliness: 'ERROR', error: error.message }
        }
      })

      try {
        const resolvedPromises = await Promise.all(this.websiteAsyncStatus)

        if (resolvedPromises) {
          this.dataReady = false
          this.websiteStatusResult = resolvedPromises

          this.websiteStatusResult.forEach((siteStatus, index) => {
            Object.assign(this.listItems[index], siteStatus)
          })

          console.log('Website analysis completed')
        }
      } catch (error) {
        console.error('Error in fetchAsync:', error)
        this.dataReady = false
      }
    },

    // Method to manually search with different parameters
    searchWithParams() {
      if (this.lat && this.lng) {
        this.dataReady = true
        this.findBusiness()
      } else {
        alert('Please click on the map first to set a location')
      }
    }
  }
}
</script>

<template>
  <div class="about">
    <header>
      <h1>Business at Google Maps</h1>
    </header>

    <div class="controls">
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
        <option value="bank">Bank</option>
        <option value="pharmacy">Pharmacy</option>
        <option value="gas_station">Gas Station</option>
        <option value="hospital">Hospital</option>
        <option value="beauty_salon">Beauty Salon</option>
      </select>

      <select v-model="searchRadius">
        <option value="500">500 m</option>
        <option value="1000">1 km</option>
        <option value="2000">2 km</option>
        <option value="10000">10 km</option>
        <option value="20000">20 km</option>
      </select>

      <button @click="searchWithParams" class="search-btn" :disabled="dataReady">
        {{ dataReady ? 'Searching...' : 'Search Current Location' }}
      </button>
    </div>

    <div id="map"></div>

    <div class="loading-wrapper" v-if="dataReady">
      <button class="buttonload">
        <i class="fa fa-spinner fa-spin"></i>
        Searching businesses...
      </button>
    </div>

    <div class="results-summary" v-if="!dataReady && listItems.length > 0">
      <p><strong>Found {{ listItems.length }} businesses</strong></p>
    </div>

    <div class="business-list-container">
      <ul class="business" v-for="(item, index) in listItems" :key="index">
        <li><span style="font-weight: bold">Nombre: </span>{{ item.name }}</li>
        <li><span style="font-weight: bold">Estatus: </span>{{ item.business_status || 'Unknown' }}</li>
        <li v-if="item.website">
          <span style="font-weight: bold">Sitio web: </span>
          <a class="website-link"
            :class="[item.mobileFriendliness === 'MOBILE_FRIENDLY' ? 'responsive' : 'not-responsive']"
            :href="item.website" target="_blank">
            {{ item.website }}
          </a>
          <span v-if="item.mobileFriendliness === 'MOBILE_FRIENDLY'" class="mobile-status responsive"> ✓ Mobile
            Friendly</span>
          <span v-else-if="item.mobileFriendliness === 'NO_WEBSITE'" class="mobile-status no-website"> No Website</span>
          <span v-else-if="item.mobileFriendliness === 'ERROR'" class="mobile-status error"> Error Testing</span>
          <span v-else class="mobile-status not-responsive"> ✗ Not Mobile Friendly</span>
        </li>
        <li v-else>
          <span style="font-weight: bold">Sitio web: </span>
          <span class="no-website">No website available</span>
        </li>
        <li>
          <span style="font-weight: bold">Tipo: </span>
          <span v-if="item.types && item.types.length > 0">
            {{ item.types.slice(0, 3).join(', ') }}
          </span>
          <span v-else>Not specified</span>
        </li>
        <li v-if="item.formatted_phone_number">
          <p style="font-weight: bold">
            Telefono:
            <a :href="`tel:${item.formatted_phone_number}`">{{ item.formatted_phone_number }}</a>
          </p>
        </li>
        <li v-else>
          <p style="font-weight: bold">Telefono: Not available</p>
        </li>
      </ul>
    </div>
  </div>
</template>

<style>
#map {
  height: 700px;
  width: 100%;
  margin: auto;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.about {
  margin: 0 auto;
  max-width: 90%;
  padding: 20px;
}

.about header {
  text-align: center;
  margin-bottom: 20px;
}

.controls {
  display: flex;
  gap: 15px;
  justify-content: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.controls select {
  padding: 8px 12px;
  border-radius: 4px;
  border: 1px solid #ddd;
  font-size: 14px;
}

.search-btn {
  padding: 8px 16px;
  background-color: #4285f4;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.search-btn:hover:not(:disabled) {
  background-color: #3367d6;
}

.search-btn:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.business-list-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 15px;
}

.business {
  box-shadow: rgba(0, 0, 0, 0.15) 0px 3px 12px;
  border-radius: 8px;
  margin: 0;
  padding: 15px;
  list-style-type: none;
  width: 350px;
  text-align: left;
  overflow: hidden;
  background: white;
  border: 1px solid #e0e0e0;
}

.business li {
  margin-bottom: 8px;
  line-height: 1.4;
}

.website-link {
  color: #1976d2;
  text-decoration: none;
}

.website-link:hover {
  text-decoration: underline;
}

.mobile-status {
  display: inline-block;
  margin-left: 10px;
  padding: 2px 6px;
  border-radius: 3px;
  font-size: 12px;
}

.responsive {
  background-color: #e8f5e8;
  color: #2e7d32;
}

.not-responsive {
  background-color: #ffebee;
  color: #c62828;
}

.no-website,
.error {
  background-color: #f5f5f5;
  color: #666;
}

.results-summary {
  text-align: center;
  margin: 15px 0;
  padding: 10px;
  background-color: #f0f7ff;
  border-radius: 4px;
}

.loading-wrapper {
  text-align: center;
  margin: 20px 0;
}

.buttonload {
  background-color: #04aa6d;
  border: none;
  color: white;
  padding: 12px 24px;
  font-size: 16px;
  border-radius: 25px;
  cursor: default;
}

.fa {
  margin-right: 8px;
}

/* Responsive design */
@media (max-width: 768px) {
  .controls {
    flex-direction: column;
    align-items: center;
  }

  .business {
    width: 100%;
    max-width: 400px;
  }

  .about {
    padding: 10px;
  }
}
</style>