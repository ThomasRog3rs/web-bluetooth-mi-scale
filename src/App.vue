<template>
  <main>
    <NavBar title="Web BLE Vue App" :bleDeviceName="connectedDeviceName" />
    <Button @clicked="read" text="Connect to Bluetooth" />
    <Button @clicked="start" id="start" text="Start Reading" />
    <Button @clicked="stop" id="stop" text="Stop Reading" />
    <h1>
      weighment from Bluetooth: <span id="weighment">{{ weight }}</span
      >KG
    </h1>
    <ProgressBar :output="parseFloat(weight)" />
    <hr />
    <Button @clicked="save">Save Weighment</Button>
    <div id="batches">
      <h1>Saved Weigments</h1>
      <ul>
        <li v-for="weighment in weighments" :key="weighment.id">
          {{ weighment.weight }}
        </li>
      </ul>
    </div>
  </main>
</template>

<script>
import NavBar from "./components/NavBar";
import Button from "./components/Button";
import ProgressBar from "./components/ProgressBar";

import axios from "axios";

let deviceName = "MIBFS";
let bleServiceUUID = "0000181b-0000-1000-8000-00805f9b34fb";
let bleCharacteristic = "00002a9c-0000-1000-8000-00805f9b34fb"; //Body Composition
let bluetoothDeviceDetected;
let gattCharacteristic;

export default {
  name: "App",
  components: {
    NavBar,
    Button,
    ProgressBar,
  },
  data() {
    return {
      weight: "0.00",
      connectedDeviceName: null,
      weighments: [],
    };
  },
  beforeMount() {
    window.addEventListener("beforeunload", (e) => {
      e.preventDefault();
    });
  },
  async mounted() {
    if (bluetoothDeviceDetected != true) {
      document.querySelector("#start").disabled = true;
      document.querySelector("#stop").disabled = true;
    }
    this.fetchWeighments();
  },
  methods: {
    fetchWeighments() {
      axios
        .get("http://localhost:3000/weighments")
        .then((response) => {
          console.log(response);
          this.weighments = response.data;
        })
        .catch((error) => {
          console.error(error);
        });
    },
    save() {
      console.log(`Saving ${this.weight}`);
      const data = {
        weight: this.weight + " KG",
      };
      axios
        .post("http://localhost:3000/weighments", data)
        .then((res) => {
          console.log(res);
          this.fetchWeighments();
        })
        .catch((err) => {
          console.error(err);
        });
    },
    read() {
      return (
        bluetoothDeviceDetected ? Promise.resolve() : this.getDeviceInfo()
      )
        .then(this.connectGATT)
        .then(() => {
          console.log("Ready to read");
          this.start();
        })
        .catch((err) => {
          console.error("Waiting to start reading: " + err);
        });
    },
    getDeviceInfo() {
      let options = {
        optionalServices: [bleServiceUUID.toLowerCase()],
        filters: [{ name: deviceName }],
      };
      console.log(options);
      console.log("requesting BLE device...");
      return navigator.bluetooth
        .requestDevice(options)
        .then((device) => {
          bluetoothDeviceDetected = device;
          console.log(bluetoothDeviceDetected);
          this.connectedDeviceName = bluetoothDeviceDetected.name;
        })
        .catch((err) => {
          console.error("Request device error: " + err);
        });
    },
    connectGATT() {
      if (bluetoothDeviceDetected.gatt.connected && gattCharacteristic) {
        return Promise.resolve();
      }

      return bluetoothDeviceDetected.gatt
        .connect()
        .then((server) => {
          console.log("Getting GATT Service...");
          console.log(server);
          return server.getPrimaryService(bleServiceUUID.toLowerCase());
        })
        .then((service) => {
          console.log("Getting GATT Characteristic...");
          console.log(service);
          return service.getCharacteristic(bleCharacteristic.toLowerCase());
        })
        .then((characteristic) => {
          console.log(characteristic);
          gattCharacteristic = characteristic;
          gattCharacteristic.addEventListener(
            "characteristicvaluechanged",
            this.handleChangedValue
          );
        })
        .catch((err) => {
          console.error("error: ", err);
        });
    },
    handleChangedValue(event) {
      console.log(event);
    },
    start() {
      if (gattCharacteristic != undefined) {
        gattCharacteristic
          .startNotifications()
          .then((_) => {
            console.log("Started reading");
            gattCharacteristic.addEventListener(
              "characteristicvaluechanged",
              (event) => {
                const value = event.target.value;
                // Process and output the received data
                console.log("Received data:", value.buffer);
                const buffer = new Uint8Array(value.buffer);
                const ctrlByte1 = buffer[1];
                const stabilized = ctrlByte1 & (1 << 5);
                const weight = ((buffer[12] << 8) + buffer[11]) / 200;
                const impedance = (buffer[10] << 8) + buffer[9];

                console.log(weight);
                this.weight = weight;
              }
            );

            document.querySelector("#start").disabled = true;
            document.querySelector("#stop").disabled = false;
          })
          .catch((error) => {
            console.error("[ERROR] Start: " + error);
          });
      }
    },
    stop() {
      gattCharacteristic
        .stopNotifications()
        .then((_) => {
          console.log("Stopped reading");
          document.querySelector("#start").disabled = false;
          document.querySelector("#stop").disabled = true;
        })
        .catch((err) => {
          console.error(`[ERROR] Stop: ${err}`);
        });
    },
    disconnect() {
      console.log(bluetoothDeviceDetected.gatt.connected);
      if (bluetoothDeviceDetected && bluetoothDeviceDetected.gatt.connected) {
        bluetoothDeviceDetected.gatt.disconnect();
        console.log("Device disconnected");
        this.weight = "0.00";
        this.connectedDeviceName = null;
        document.querySelector("#stop").disabled = true;
      }
    },
  },
};
</script>

<style>
@import url("https://fonts.googleapis.com/css?family=Nunito:400,700&display=swap");
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
body {
  background: #fdfdfd;
  font-family: "Nunito", sans-serif;
  font-size: 1rem;
}
main {
  max-width: 900px;
  margin: auto;
  padding: 0.5rem;
  text-align: center;
}
h1,
h2,
h3,
h4,
h5,
h6 {
  color: #e74c3c;
  margin-bottom: 0.5rem;
}

h3 {
  color: #e74d3cca;
}

.container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(15rem, 1fr));
  grid-gap: 1rem;
  justify-content: center;
  align-items: center;
  margin: auto;
  padding: 1rem 0;
}

.indicate {
  display: block;
  width: 50px;
  height: 50px;
  margin: auto auto;
  margin-bottom: 10px;
  background-color: rgb(43, 138, 255);
  border-radius: 5px;
}

nav {
  margin-bottom: 100px;
}

form {
  margin-bottom: 15px;
}

#myProgress {
  width: 100%;
  background-color: #ddd;
  border: 1px solid #e74c3c;
}

#myBar {
  width: 1%;
  height: 30px;
  background-color: rgb(43, 138, 255);
}

div#batches {
  border: 1px solid #e74c3c;
  border-radius: 20px;
  margin-top: 20px;
}

ul {
  list-style-type: none;
  margin: 0;
  padding: 0;
}
</style>
