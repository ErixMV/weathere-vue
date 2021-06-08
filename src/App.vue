<template>
  <div class="main" :class="mode === 'dark' ? 'dark' : ''">
    <div class="left">
      <Header />
      <input class="city-selector" type="text" ref="input" />
    </div>
    <div v-show="forecastCard">
      <div class="forecast-wrapper">
        <div v-for="(day, i) in forecastCard" :key="i" class="day-card">
          <div class="today" v-if="i === 0">
            <div>
              <span class="current-temp">{{ day.temperature }}</span>
              <span class="degree-symbol">º</span>
            </div>
            <div>{{ day.tempRange[0] }}º / {{ day.tempRange[1] }}º</div>
            <div>{{ day.text }}</div>
            <div>{{ day.place }}</div>
          </div>
          <div class="collapsed" v-else>
            <div>{{ i === 1 ? "Tomorrow" : day.dayWeek }}, {{ day.dayMonth }}</div>
            <div class="icon-pop">
              <img class="icon" :src="day.icon" />
              <span>{{ day.pop }} %</span>
            </div>
            <div>{{ day.tempRange[0] }}º / {{ day.tempRange[1] }}º</div>
          </div>
        </div>
      </div>
      <section class="wind-info" v-if="today">
        <div class="title-section">Wind</div>
        <div class="wind-icon">
          <img src="/images/basmilius/wind.svg" />
        </div>
        <div>
          <div>Direction: {{ today.windDir }}</div>
          <div>Speed: {{ today.windSpeed }} Km/h</div>
        </div>
      </section>
      <section class="wind-info" v-if="today">
        <div class="title-section">Clouds</div>
        <div class="wind-icon">
          <img src="/images/basmilius/cloud.svg" />
        </div>
        <div>
          <div>Clouds coverage: {{ today.cloudsCov }} %</div>
        </div>
      </section>
      <section class="wind-info" v-if="today">
        <div class="title-section">Rain</div>
        <div class="wind-icon">
          <img src="/images/basmilius/rain.svg" />
        </div>
        <div>
          <div>Prob. of precipitation: {{ today.pop }} %</div>
          <div>Relative Humidity: {{ today.relHumidity }} %</div>
        </div>
      </section>
    </div>
  </div>
  <Toggle :mode="mode" @toggle="toggle" />
</template>

<script>
import * as places from "places.js";
import axios from "axios";
import { placesConf, configureConf } from "./config/places";
import { dayWeek } from "./assets/static/dayWeek";
import { monthName } from "./assets/static/monthName";
import Header from "./components/Header";
import Toggle from "./components/Toggle";

export default {
  name: "App",
  data() {
    return {
      city: "",
      error: "",
      forecast: [],
      mode: "light",
    };
  },
  computed: {
    today() {
      if (this.forecast.length !== 0) {
        const today = this.forecast[0];
        return {
          windDir: today.wind_cdir_full,
          windSpeed: Math.round(today.wind_spd * 3.6),
          pop: today.pop,
          relHumidity: today.rh,
          cloudsCov: today.clouds,
        };
      }
    },
    forecastCard() {
      return this.forecast.map((day) => {
        // const dayWeek = [
        //   "Sunday",
        //   "Monday",
        //   "Tuesday",
        //   "Wednesday",
        //   "Thursday",
        //   "Friday",
        //   "Saturday",
        // ];
        // const monthName = [
        //   "jan",
        //   "fre",
        //   "mar",
        //   "apr",
        //   "may",
        //   "jun",
        //   "jul",
        //   "aug",
        //   "sep",
        //   "oct",
        //   "nov",
        //   "dec",
        // ];
        const d = new Date(day.datetime);
        return {
          place: this.city.split(",")[0],
          dayMonth: `${d.getDate()} ${monthName[d.getMonth()]}.`,
          dayWeek: dayWeek[d.getDay()],
          temperature: Math.floor(day.temp),
          text: day.weather.description,
          icon: `/images/algoliaIcons/${day.weather.icon}.png`,
          tempRange: [Math.trunc(day.max_temp), Math.trunc(day.min_temp)],
          pop: day.pop,
        };
      });
    },
  },
  watch: {
    forecast() {
      console.log(this.forecast);
    },
  },
  components: {
    Header,
    Toggle,
  },
  methods: {
    toggle() {
      this.mode = this.mode === "dark" ? "light" : "dark";
    },
    submitHandler(instance, latlng) {
      instance.getVal() !== ""
        ? this.getForecast(latlng)
        : (this.error = "Select a city.");
    },
    async getForecast({ lat, lng }) {
      try {
        const { data } = await axios.post("http://localhost:5000/rest/forecast", {
          lat,
          lng,
        });
        this.forecast = data.data;
      } catch (err) {
        console.log(err.message);
        this.error = "The forecast of the selected city is not available.";
      }
    },
  },
  async mounted() {
    const instancePlacesAutocomplete = await places({
      // appId: data.id,
      // apiKey: data.apikey,
      ...placesConf,
      container: this.$refs.input,
    }).configure(configureConf);

    // Places events
    //Change
    await instancePlacesAutocomplete.on("change", ({ suggestion }) => {
      this.city = suggestion.value;
      instancePlacesAutocomplete.close();

      this.submitHandler(instancePlacesAutocomplete, suggestion.latlng);
    });

    // //Locate
    // await instancePlacesAutocomplete.on("locate", () => {
    //   if ("geolocation" in navigator) {
    //     navigator.geolocation.getCurrentPosition(
    //       ({ coords }) => {
    //         setCity("Current location");
    //         instancePlacesAutocomplete.setVal("Current location");
    //         getForecast({ lat: coords.latitude, lng: coords.longitude });
    //         instancePlacesAutocomplete.close();
    //       },
    //       (error) => {
    //         console.log(error);
    //         setError(`GeoLocation failed: ${error.message}`);
    //         setTimeout(() => {
    //           setError("");
    //         }, 2500);
    //       },
    //       { maximumAge: Infinity, timeout: 2000 }
    //     );
    //   } else {
    //     setError("Geolocation is not available");
    //     setTimeout(() => {
    //       setError("");
    //     }, 2500);
    //   }
    // });

    // Clear
    await instancePlacesAutocomplete.on("clear", () => {
      instancePlacesAutocomplete.setVal("");
      this.city = "";
      this.forecast = [];
    });

    // Error
    await instancePlacesAutocomplete.on("error", () => {
      console.log("error");

      instancePlacesAutocomplete.setVal("");
      instancePlacesAutocomplete.close();
    });
  },
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  font-family: sans-serif;
  box-sizing: border-box;
}

body {
  background: url("/images/bg2.jpg");
  -webkit-background-size: cover;
  -moz-background-size: cover;
  -o-background-size: cover;
  background-size: cover;
  background-attachment: fixed;
}

#app {
  /* color: #ddd; */
  /* width: 100vw;
  height: 100vh; */
}

.main {
  /* Center */
  /* position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%); */

  background: rgba(255, 255, 255, 0.8);
  /* background: #fff; */
  /* height: 90vh; */
  width: 90vw;
  padding: 10px;
  margin: auto;
}

.left {
  width: 100%;
}

.left .city-selector {
}

.title-section {
  display: block;
  width: 100%;
  border-bottom: 1px solid #ccc;
  padding: 7px 0px;
  margin-left: 5px;
}

.forecast-wrapper {
  display: flex;
  justify-content: center;
  flex-direction: column;
}

.day-card {
  font-size: 12px;
  padding: 3px 5px;
  border-bottom: 1px solid #aaa;
}

.day-card:first-child,
.day-card:last-child {
  border: 0;
}

.day-card .today {
  text-align: center;
}

.day-card .today div:first-child {
  display: flex;
  justify-content: center;
}

.day-card .icon-pop {
  display: flex;
  justify-content: center;
  align-items: center;
}

.day-card .icon-pop span {
  margin-left: 10px;
}

.day-card .today .current-temp {
  font-size: 3rem;
}

.day-card .today .degree-symbol {
  position: relative;
  top: 7px;
}

.day-card .collapsed {
  font-size: 25px;
  text-align: center;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: inherit;
  height: 40px;
}

.day-card .collapsed div {
  width: 33%;
}

.day-card .collapsed .icon {
  height: 40px;
}

.day-card .collapsed div:first-child {
  text-align: left;
}

.day-card .collapsed div:last-child {
  text-align: right;
}

/* Wind */
.wind-info {
  display: flex;
  justify-content: space-evenly;
  align-items: center;
  flex-wrap: wrap;
}

.wind-info div:last-child {
  width: 50%;
  white-space: nowrap;
}

.wind-icon {
  /* width: 50%; */
}

.wind-icon img {
  width: 100%;
}

.dark {
  background: rgba(0, 0, 0, 0.8);
  color: #f3f3f3;
}
</style>
