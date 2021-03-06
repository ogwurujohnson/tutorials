<template>
  <div id="app">
    <GmapMap
      :center="{ lat: initialPosition.lat, lng: initialPosition.lng }"
      :zoom="10"
      map-type-id="terrain"
      style="width: 100%; height: 90%"
    >
      <!-- check if icon link in makers payload then display -->
      <GmapMarker
        :key="index"
        v-for="(m, index) in markers"
        :position="m.position"
        :clickable="true"
        :draggable="false"
        @click="center = m.position"
        :icon="m.icon"
        :title="m.userName"
      />
    </GmapMap>

    <div class="notification">
      <p>Online Users: {{ markers.length }}</p>
      <ul>
        <li v-for="(user, i) in onlineUsers" :key="i">
          <pre v-text="user.data.userName"></pre>
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
import axios from "axios";
import * as Ably from "ably";
import Loading from "vue-loading-overlay";
import "vue-loading-overlay/dist/vue-loading.css";

var ably_keys = process.env.VUE_APP_ABLY_KEY
var ably = new Ably.Realtime({
  key: ably_keys,
  clientId: `${Math.random() * 1000000}`
});

export default {
  name: "Application",
  mounted() {
    const name = prompt(
      "To get started, input your name in the field below and locate your friends around based on your location, please turn on your location setting \n What is your name?"
    );
    this.usersName = name;
    const channel = prompt("Enter name of channel you are interested in");
    this.channelName = channel;
  },
  async created() {
    await this.fetchData();
    var channel = ably.channels.get(this.channelName);
    channel.attach(err => {
      if (err) {
        return console.error("Error attaching to the channel");
      }
      console.log("We are now attached to the channel");
      channel.presence.enter(this.userlocation, function(err) {
        if (err) {
          return console.error("Error entering presence");
        }
        console.log("We are now successfully present");
      });
    });

    let self = this;
    channel.presence.subscribe("update", function(presenceMsg) {
      console.log(presenceMsg)
      console.log(
        "Received a " + presenceMsg.action + " from " + presenceMsg.clientId
      );
      channel.presence.get(function(err, members) {
        console.log(members)
        self.markers = members.map(mem => {
          if (JSON.stringify(self.userlocation) == JSON.stringify(mem.data)) {
            return {
              ...mem.data,
              icon: "http://maps.google.com/mapfiles/ms/icons/blue-dot.png"
            };
          } else {
            return {
              ...mem.data,
              icon: "http://maps.google.com/mapfiles/ms/icons/red-dot.png"
            };
          }
        });
        self.onlineUsers = members;
        console.log(
          "There are now " + members.length + " clients present on this channel"
        );
      });
    });
  },

  data() {
    return {
      usersName: null,
      gettingLocation: true,
      initialPosition: {
        lat: 10,
        lng: 10
      },
      zoom: 11,
      markers: [],
      userlocation: [],
      onlineUsers: [],
      channelName: null
    };
  },
  components: {
    Loading
  },
  methods: {
    fetchData() {
      if (!("geolocation" in navigator)) {
        this.errorStr = "Geolocation is not available.";
        return;
      }
      this.gettingLocation = true;
      navigator.geolocation.watchPosition(
        pos => {
          this.gettingLocation = false;
          this.initialPosition.lat = pos.coords.latitude;
          this.initialPosition.lng = pos.coords.longitude;
          const userData = {
            position: {
              lat: pos.coords.latitude,
              lng: pos.coords.longitude
            },
            userName: this.usersName
          };
          this.userlocation = userData;
          this.updateRoom(userData);
        },
        err => {
          this.gettingLocation = false;
          this.errorStr = err.message;
        }
      );
    },

    updateRoom(data) {
      var channel = ably.channels.get(this.channelName);
      channel.presence.update(data, function(err) {
        if (err) {
          return console.error("Error updating presence data");
        }
        console.log("We have successfully updated our data");
      });
    }
  },
};
</script>

<style>
body {
  margin: 0;
}

#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  background: #becbd8;
  height: 100vh;
  width: 100%;
  display: flex;
  flex-direction: column;
}

.notification {
  background: #ffffff;
  padding-left: 2rem;
  font-weight: bold;
  font-size: 13px;
  padding-bottom: 1rem;
}

i {
  font-style: italic;
}

main {
  text-align: center;
  margin-top: 40px;
  display: flex;
  flex-direction: column-reverse;
  justify-content: center;
  align-items: center;
  box-shadow: 0 19px 38px rgba(0, 0, 0, 0.3), 0 15px 12px rgba(0, 0, 0, 0.22);
  width: 25%;
  margin: 20px auto;
  height: 65%;
}
h1 {
  line-height: 0;
  font-size: 7em;
}

sup {
  font-size: 20px;
}

span {
  font-size: 20px;
}

img {
  width: 150px;
  vertical-align: top;
}

.details {
  color: #35495e;
  font-weight: bold;
  text-align: center;
}

header {
  margin: 0;
  height: 56px;
  padding: 0 16px 0 24px;
  background-color: #35495e;
  color: #ffffff;
}

header span {
  display: block;
  position: relative;
  font-size: 20px;
  line-height: 1;
  letter-spacing: 0.02em;
  font-weight: 400;
  box-sizing: border-box;
  padding-top: 16px;
}

@media screen and (max-width: 450px) {
  main {
    box-shadow: none;
    width: 100%;
  }
}
</style>
