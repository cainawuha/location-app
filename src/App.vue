
<template>
  <div>
    <button @click="getCurrentLocation">Get Current Location</button>
    <input v-model="searchQuery" @keyup.enter="searchLocation">
    <button @click="searchLocation">Search</button>

    <div id="map" style="height: 500px;"></div>

    <table>
      <thead>
        <tr>
          <th><button @click="deleteSelected">Delete</button></th>
          <th>Location</th>
          <th>Local Time</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="record in displayedRecords" :key="record.id">
          <td><input type="checkbox" v-model="selectedRecords" :value="record.id"></td>
          <td>{{ record.name }}</td>
          <td>{{ record.time }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { Loader } from "@googlemaps/js-api-loader";
import axios from 'axios';
export default {
  data() {
    return {
      searchQuery: '',
      map: null,
      markers: [],
      records: [],
      selectedRecords: [],
      currentPage: 1,
      totalPages: 1,
      recordsPerPage: 10,
      allSelected: false,
    };
  },
  computed: {
    displayedRecords() {
      const start = (this.currentPage - 1) * 10;
      const end = this.currentPage * 10;
      return this.records.slice(start, end);
    },
    maxPage() {
      return Math.ceil(this.records.length / this.recordsPerPage);
    },
  },
  methods: {
    async getCurrentLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(async position => {
          const pos = {
            lat: position.coords.latitude,
            lng: position.coords.longitude
          };

          const placeName = await this.getPlaceNameFromLocation(pos.lat, pos.lng);

          const tzResponse = await axios.get('https://maps.googleapis.com/maps/api/timezone/json', {
            params: {
              location: `${pos.lat},${pos.lng}`,
              timestamp: Math.floor(Date.now() / 1000),
              key: 'AIzaSyDRIxglsfllrMEItudCbeyPY4NDvbIUIAQ'
            }
          });
          const offset = tzResponse.data.dstOffset + tzResponse.data.rawOffset;

          // const localTime = new Date(Date.now() + offset * 1000).toLocaleString();
          const localTimeInMilliseconds = Date.now() + offset * 1000;
          const localDate = new Date(localTimeInMilliseconds);
          const localTime = localDate.getUTCFullYear() + "-" +
            (localDate.getUTCMonth() + 1).toString().padStart(2, '0') + "-" +
            localDate.getUTCDate().toString().padStart(2, '0') + ", " +
            localDate.getUTCHours().toString().padStart(2, '0') + ":" +
            localDate.getUTCMinutes().toString().padStart(2, '0') + ":" +
            localDate.getUTCSeconds().toString().padStart(2, '0') + " " +
            (localDate.getUTCHours() >= 12 ? "p.m." : "a.m.");
          this.addRecordAndMarker(placeName, pos, localTime);
          this.map.setCenter(pos);
          console.log("Final Local Time before adding to records:", localTime);

        });
      } else {
        console.error("Geolocation is not supported by this browser.");
      }
    },

    async searchLocation() {
      try {
        // Step 1: Get Geocoding Info
        const geoResponse = await axios.get('https://maps.googleapis.com/maps/api/geocode/json', {
          params: {
            address: this.searchQuery,
            key: 'AIzaSyDRIxglsfllrMEItudCbeyPY4NDvbIUIAQ'
          }
        });

        if (geoResponse.data.results && geoResponse.data.results.length > 0) {
          const location = geoResponse.data.results[0].geometry.location;


          // Step 2: Get Time Zone Info
          const tzResponse = await axios.get('https://maps.googleapis.com/maps/api/timezone/json', {
            params: {
              location: `${location.lat},${location.lng}`,
              timestamp: Math.floor(Date.now() / 1000),
              key: 'AIzaSyDRIxglsfllrMEItudCbeyPY4NDvbIUIAQ'
            }
          });

          const offset = tzResponse.data.dstOffset + tzResponse.data.rawOffset;
          // Step 3: Calculate Local Time
          const localTimeInMilliseconds = Date.now() + offset * 1000;
          const localDate = new Date(localTimeInMilliseconds);
          const localTime = localDate.getUTCFullYear() + "-" +
            (localDate.getUTCMonth() + 1).toString().padStart(2, '0') + "-" +
            localDate.getUTCDate().toString().padStart(2, '0') + ", " +
            localDate.getUTCHours().toString().padStart(2, '0') + ":" +
            localDate.getUTCMinutes().toString().padStart(2, '0') + ":" +
            localDate.getUTCSeconds().toString().padStart(2, '0') + " " +
            (localDate.getUTCHours() >= 12 ? "p.m." : "a.m.");


          this.addRecordAndMarker(this.searchQuery, location, localTime);
          this.map.setCenter(location);
          console.log("Final Local Time before adding to records:", localTime);
        } else {
          console.error("No results found for this address.");
        }
      } catch (error) {
        console.error("Error fetching location or time zone information:", error);
      }
    },


    addRecordAndMarker(name, location, localTime) {

      this.records.push({
        id: Date.now(),
        name: name,
        location: location,
        time: localTime
      });
      // eslint-disable-next-line
      const marker = new google.maps.Marker({
        position: location,
        map: this.map
      });

      this.markers.push(marker);
    },

    deleteSelected() {
      this.records = this.records.filter(record => !this.selectedRecords.includes(record.id));
      this.markers = this.markers.filter((marker, index) => {
        if (this.selectedRecords.includes(this.records[index].id)) {
          marker.setMap(null);
          return false;
        }
        return true;
      });
      this.selectedRecords = [];
      this.allSelected = false;
    },
    nextPage() {
      if (this.currentPage < this.maxPage) {
        this.currentPage += 1;
      }
    },
    prevPage() {
      if (this.currentPage > 1) {
        this.currentPage -= 1;
      }
    },

    selectAll() {
      if (this.allSelected) {
        this.selectedRecords = this.displayedRecords.map(record => record.id);
      } else {
        this.selectedRecords = [];
      }
    },

    async getPlaceNameFromLocation(lat, lng) {
      const response = await axios.get('https://maps.googleapis.com/maps/api/geocode/json', {
        params: {
          latlng: `${lat},${lng}`,
          key: 'AIzaSyDRIxglsfllrMEItudCbeyPY4NDvbIUIAQ'
        }
      });


      if (response.data.results && response.data.results.length > 0) {
        return response.data.results[0].formatted_address;
      } else {
        return 'Unknown Location';
      }
    }

  },
  mounted() {
    const loader = new Loader({
      apiKey: "AIzaSyDRIxglsfllrMEItudCbeyPY4NDvbIUIAQ",
      version: "weekly"
    });

    loader.load().then(() => {
      // eslint-disable-next-line
      this.map = new google.maps.Map(document.getElementById("map"), {
        center: { lat: -34.397, lng: 150.644 },
        zoom: 8
      });
    });
  }
};
</script>

