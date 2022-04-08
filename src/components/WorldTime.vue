<template>
  <div>
    <div>
      <WorldMap @map-clicked="what" />
    </div>
    <input type="text" :value="geoPosition">
    <button @click="searchCity">Add</button>
    <button @click="saveCities">Save</button><br>
    <span id="msgbox" v-show="message.length > 0">{{message}}</span>
    <div id="clocks">
      <Clock v-for="(c,pos) in selectedCities" :key="pos"
        :time-zone="c.timeZone" :label="c.name">
      </Clock><br>
      <p>Enter name of place you wish to remove:</p>
      <input type="text" v-model="remove">
      <button @click="removeTime">Remove Time</button>
    </div>
  </div>
</template>

<script lang="ts">
// import ChildComponent from "./60-props-child.vue";
import axios, { AxiosResponse } from "axios";
import Clock from "./Clock.vue";
import WorldMap from "./WorldMap.vue";
import { Vue, Component } from "vue-property-decorator";
import { getAuth } from "firebase/auth";
import { FirebaseApp, initializeApp } from "firebase/app";
import { DocumentReference, setDoc, doc, getDoc, updateDoc, deleteField, Firestore, getFirestore, DocumentSnapshot } from "firebase/firestore";
import firebaseConfig from "@/myconfig";

const TimeApp:FirebaseApp = initializeApp(firebaseConfig);
const TimeDB:Firestore = getFirestore(TimeApp);

const timezoneDBUrl = "http://api.timezonedb.com/v2.1";
type City = {
  name: string;
  timeZone: string;
};

type TimeZoneData = {
  countryName: string;
  gmtOffset: number;
  regionName: string;
  zoneName: string;
};
@Component({ components: { Clock, WorldMap } })
export default class Sample extends Vue {
  remove = "";
  geoPos: { lat?: number; lng?: number } = {};
  selectedCities: Array<City> = [];
  apiKey = "";
  message = "";

  saveCities(): void {
    const auth = getAuth();
    const uid = auth.currentUser?.uid;
    const doc1: DocumentReference = doc(TimeDB, `users/${uid}`)

    for (let i = 0; i < this.selectedCities.length; i++) {
    setDoc(doc1, { name: `${this.selectedCities[i].name}`, timeZone: `${this.selectedCities[i].timeZone}`})
    .then(() => {
    console.log("New doc added");
    })
    .catch((err:any) => {"error"});
    }

    /* const proms = this.selectedCities.map((st:any) => {
    const cityTimes = collection(TimeDB, `users/${uid}/${st.name}`);
    return addDoc(cityTimes, {name: st.name, timeZone: st.timeZone});
    })
    Promise.all(proms).then(() => {
    console.log("Batch inserts done");
    })
     .catch((err:any) => { /* your code here  }); 
    */
  }

  removeTime(): void {
    this.selectedCities = this.selectedCities.filter((z) => z.name != this.remove);
    
    const auth = getAuth();
    const uid = auth.currentUser?.uid;
    const timeDoc = doc(TimeDB, `users/${uid}`);
    updateDoc(timeDoc,
    { name: deleteField(), timeZone: deleteField() })
    .then(() => { console.debug("Fields deleted") });
  }

  showMessage(txt: string) {
    this.message = txt;

    // The message will automatically disappear after 5 seconds
    setTimeout(() => {
      this.message = "";
    }, 5000);
  }
  
  mounted(): void {
    this.apiKey = process.env.VUE_APP_TIMEZONE_API_KEY;

    const auth = getAuth();
    const uid = auth.currentUser?.uid;

    const mich : DocumentReference = doc(TimeDB, `users/${uid}`);
    getDoc(mich).then((myDoc: DocumentSnapshot) => {
    if (myDoc.exists()) {
      const details = myDoc.data();
      this.selectedCities.push({name: details.name, timeZone: details.timeZone})
    }
    });
  }

  get geoPosition(): string {
    if (this.geoPos.lat && this.geoPos.lng)
      return `${this.geoPos.lat.toPrecision(5)},${this.geoPos.lng.toPrecision(
        5
      )}`;
    else return "N/A";
  }
  searchCity(): void {
    const param = new URLSearchParams();
    param.append("key", this.apiKey);
    param.append("format", "json");
    param.append("by", "position");
    param.append("lat", this.geoPos.lat!.toString());
    param.append("lng", this.geoPos.lng!.toString());
    const tzUrl = `${timezoneDBUrl}/get-time-zone?` + param.toString();
    // Use a Web Proxy Server to get around CORS issue
    // since timezonedb.com does not allow CORS
    axios
      .request({
        method: "GET",
        url: "https://api.allorigins.win/get",
        params: {
          url: tzUrl,
        },
      })
      .then((r: AxiosResponse) => {
        return r.data;
      })
      .then((r: any) => JSON.parse(r.contents))
      .then((r: TimeZoneData) => {
        // Add the selected location to our array
        if (this.selectedCities.some((c) => c.name === r.regionName)){
          this.showMessage(`${r.regionName} has already been selected`)
        } else {
        this.selectedCities.push({ name: r.regionName, timeZone: r.zoneName });
        }
      });
  }

  what(geoPos: { lat: number; lng: number }): void {
    // When the user pans the map left/right the longitude
    // angle can be out of the [-180,+180] range
    while (geoPos.lng > 180) geoPos.lng -= 360;
    while (geoPos.lng < -180) geoPos.lng += 360;
    this.geoPos = { ...geoPos };
  }


}
</script>

<style scoped>
#clocks {
  margin-top: 1em;
}
pre {
  white-space: normal;
  padding: 0.5em;
  border: 2px solid gray;
}
#msgbox {
  font-size: 80%;
  font-style: italic;
  border-radius: 0.5em;
  background-color: hsl(0, 0%, 75%);
  padding: 0.5em;
}
</style>