<template>
  <div class="logo"><img src="../assets/logo.svg" alt="Μάντεψε τον Νομό"/></div>
  <div class="menu">
    <button @click="startGame" :disabled="gameState.started" v-if="!gameState.started">Ξεκίνα Νέο</button>
  </div>

  <div class="currentMunicipality" v-if="gameState.started">{{ gameState.currentMunicipality }}</div>
  <div id="map" style="height: 450px; width: 100%">
    <l-map :zoom="zoom" :center="center" :use-global-leaflet="false" :options="options" ref="mapRef">
      <l-geo-json :geojson="municipalities" :options="munOptions" :options-style="municipalitiesStyle"></l-geo-json>
    </l-map>

    <div class="overlay" v-if="gameState.started">
      <p>{{ 55 - municipalitiesNames.length }}/54</p>
      <p>Σκορ: {{ gameState.score }}</p>
      <p>Προσπάθειες: {{ numOfTries }}/3</p>
    </div>
  </div>

  <div v-if="gameState.finished" class="popup" ref="popupRef">
    <h2>{{ title }}</h2>
    <p>{{ message }}</p>
    <button @click="gameState.finished = false">Κλείσιμο</button>
  </div>
  <div class="wrong-answer" ref="answerRef">
    {{ answer }}
  </div>
</template>

<script setup>
import {ref} from 'vue';
import 'leaflet/dist/leaflet.css';

const options = ref({
  zoomControl: false,
  scrollWheelZoom: true,
  doubleClickZoom: false,
  dragging: true,
  attributionControl: false,
});
const zoom = 6;
const center = [38.286, 24.000]; // Centered on Greece
const url = 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png';
const voices = speechSynthesis.getVoices();
const greekVoice = voices.find(voice => voice.lang === 'el-GR')
console.log(greekVoice)
</script>

<script>
import {LMap, LTileLayer, LGeoJson} from '@vue-leaflet/vue-leaflet';
import municipalities from '../data/municipalities_simple.json';
import wrongSound from '@/assets/sounds/wrong.mp3';
import rightSound from '@/assets/sounds/right.mp3';
import finishSound from '@/assets/sounds/finish2.mp3';
const wrongAudio = new Audio(wrongSound);
const rightAudio = new Audio(rightSound);
const finishAudio = new Audio(finishSound);

const fillColors = ['#f00', '#f0f', '#ff0', '#0f0'];

export default {
  components: {
    LMap,
    LTileLayer,
    LGeoJson,
  },
  data() {
    return {
      numOfTries: 3,
      gameState: {
        started: false,
        finished: false,
        score: 0,
        currentMunicipality: null,
      },
      municipalitiesNames: [],
      answer: '',
      title: '',
      message: '',
    };
  },
  mounted() {
    this.$refs.answerRef.addEventListener('animationend', this.handleAnimationEnd);
  },
  computed: {
    munOptions() {
      return {
        onEachFeature: this.onEachFeatureFunction
      };
    },
    onEachFeatureFunction() {
      return (feature, layer) => {
        layer.on('click', (e) => {
          this.answer = feature.properties.NAME_GR;
          this.checkAnswer(feature.properties.NAME_GR, e);
        });
      };
    },
    municipalitiesStyle() {
      return () => {
        return {
          fillColor: "#fff",
          weight: 1,
          opacity: 1,
          color: "#000",
          fillOpacity: 0.7
        };
      };
    },
  },
  methods: {
    getRandomMunicipalityName() {
      return this.municipalitiesNames[Math.floor(Math.random() * this.municipalitiesNames.length)];
    },
    speakMunicipalityName(name) {
      const synth = window.speechSynthesis;
      const utterThis = new SpeechSynthesisUtterance(name);
      if (greekVoice) {
        utterThis.voice = greekVoice;
      }
      utterThis.lang = "el-GR";
      synth.speak(utterThis);
    },
    startGame() {
      this.municipalitiesNames = municipalities.features.map(feature => {
        return feature.properties.NAME_GR;
      });
      this.gameState.started = true;
      this.gameState.finished = false;
      this.gameState.score = 0;
      this.clearMap();
      this.nextMunicipality();
    },
    checkAnswer(answer, e) {
      if (this.gameState.finished || !this.gameState.started || !this.municipalitiesNames.includes(answer)) {
        return;
      }
      if (answer === this.gameState.currentMunicipality) {
        this.playSound(rightAudio)
        this.gameState.score += this.numOfTries;
        // remove the municipality from the list
        this.municipalitiesNames = this.municipalitiesNames.filter(name => name !== this.gameState.currentMunicipality);
        if (this.municipalitiesNames.length === 0) {
          this.setColor();
          this.finishGame();
        } else {
          this.setColor();
          this.nextMunicipality();
        }
      } else {
        this.playSound(wrongAudio)
        this.$refs.answerRef.style.top = `${e.originalEvent.clientY - 5}px`;
        this.$refs.answerRef.style.left = `${e.originalEvent.clientX + 5}px`;
        this.$refs.answerRef.style.display = 'block';
        this.numOfTries -= 1;
        if (this.numOfTries === 0) {
          // remove the municipality from the list
          this.municipalitiesNames = this.municipalitiesNames.filter(name => name !== this.gameState.currentMunicipality);
          this.setColor();
          this.nextMunicipality();
        }
      }
    },
    playSound(audio) {
      if (!audio.paused) {
        audio.pause();
        audio.currentTime = 0;
      }
      audio.play();
    },
    setColor() {
      this.$refs.mapRef.leafletObject.eachLayer(layer => {
        if (layer.feature && layer.feature.properties.NAME_GR === this.gameState.currentMunicipality) {
          layer.setStyle({fillColor: fillColors[this.numOfTries]});
        }
      });
    },
    clearMap() {
      this.$refs.mapRef.leafletObject.eachLayer(layer => {
        if (layer.feature) {
          layer.setStyle({fillColor: "#fff"});
        }
      });
    },
    nextMunicipality() {
      this.numOfTries = 3;
      this.gameState.currentMunicipality = this.getRandomMunicipalityName();
      // this.speakMunicipalityName(this.gameState.currentMunicipality);
    },
    finishGame() {
      this.playSound(finishAudio)
      this.gameState.finished = true;
      this.gameState.started = false;
      this.showScore('Τέλος παιχνιδιού', `Το σκορ σας είναι ${this.gameState.score}`);
    },
    showScore(title, message) {
      this.title = title;
      this.message = message;
    },
    handleAnimationEnd(e) {
      if (e.animationName === 'fadeOut') {
        this.$refs.answerRef.style.display = 'none';
      }
    },
  },
};
</script>

<style>
.logo {
  text-align: center;
  margin: 10px 0;

  img {
    width: 300px;
    max-width: 60%;
  }
}

h1 {
  text-align: center;
  font-size: 2em;
  margin-top: 10px;
  color: #fff;
  text-shadow: 0 0 2px #8f6641;
}

#map {
  position: relative;
}

.menu {
  display: block;
  text-align: center;
  margin-bottom: 10px;

  button {
    padding: 10px 20px;
    font-size: 1.2em;
    background: #3a89ff;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
}

.leaflet-container {
  background-color: #e2d1a0;
}

.currentMunicipality {
  display: flex;
  justify-content: center;
  background: #fcb040;
  color: #fff;
  padding: 5px;
  font-weight: 600;
  font-size: 1.2em;
  border-radius: 5px 5px 0 0;
  text-shadow: -2px 2px 1px #b39261;
}

.popup {
  position: absolute;
  top: 150px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 20px;
  border-radius: 10px;
  font-size: 1.5em;
  z-index: 999;
  text-align: center;

  button {
    padding: 10px 20px;
    font-size: 1.2em;
    background: #3a89ff;
    color: #fff;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
}

.overlay {
  display: flex;
  justify-content: space-between;
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  font-size: 0.8em;
  z-index: 999;
  box-sizing: border-box;
  padding: 0 5px;

  p {
    padding: 2px;
    margin: 0;
  }
}

.wrong-answer {
  position: absolute;
  display: none;
  background: #f00;
  color: #fff;
  padding: 2px;
  border-radius: 5px;
  animation: show .3s, fadeOut 1s 2s;
  z-index: 999;
}

@keyframes show {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}

@keyframes fadeOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}
</style>