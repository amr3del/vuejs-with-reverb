<template>
  <div class="hello">
    <h1>{{ msg }}</h1>

    <div class="connection-status">
      <p>
        <strong>Connection Status:</strong>
        <span :class="connectionStatusClass">{{ connectionStatus }}</span>
      </p>
    </div>

    <div v-if="vehicles && vehicles.length > 0" class="vehicle-data">
      <h3>🚗 Active Vehicles ({{ vehicles.length }})</h3>

      <div v-for="vehicle in vehicles" :key="vehicle.id" class="vehicle-card">
        <div class="vehicle-header">
          <h4>{{ vehicle.brand }} - {{ vehicle.model }}</h4>
          <span class="status" :class="vehicle.status">{{
            vehicle.status
          }}</span>
        </div>

        <div class="vehicle-info">
          <p><strong>Vehicle Number:</strong> {{ vehicle.vehicle_number }}</p>
          <p><strong>License Plate:</strong> {{ vehicle.license_plate }}</p>
          <p><strong>Odometer:</strong> {{ vehicle.odometer_reading }} km</p>

          <div v-if="vehicle.location" class="location-info">
            <p><strong>📍 Location:</strong></p>
            <p>Latitude: {{ vehicle.location.lat }}</p>
            <p>Longitude: {{ vehicle.location.lng }}</p>
            <p>Address: {{ vehicle.location.address }}</p>
          </div>

          <div v-if="vehicle.driver" class="driver-info">
            <p><strong>👤 Driver:</strong> {{ vehicle.driver.name }}</p>
            <p>
              Phone: {{ vehicle.driver.country_code }}{{ vehicle.driver.phone }}
            </p>
            <p>Email: {{ vehicle.driver.email }}</p>
          </div>
        </div>
      </div>
    </div>

    <p>
      Connected to Laravel Reverb server at:<br />
      <code>ws://apidabt.cayan.co:45133/app/sjaj94p7s6kzgifoct6t</code>
    </p>
    <p>
      Listening on channel: <code>company.2.tracking.vehicles</code><br />
      Event: <code>tracking.vehicles</code>
    </p>
  </div>
</template>

<script>
import Echo from "laravel-echo";
import Pusher from "pusher-js";

export default {
  name: "HelloWorld",
  props: {
    msg: String,
  },
  data() {
    return {
      echo: null,
      channel: null,
      vehicles: [],
      vehicleData: null,
      connectionStatus: "Disconnecting...",
    };
  },
  mounted() {
    this.initializeEcho();
  },
  computed: {
    connectionStatusClass() {
      return this.connectionStatus;
    },
  },
  methods: {
    initializeEcho() {
      window.Pusher = Pusher;

      this.echo = new Echo({
        broadcaster: "pusher",
        key: "sjaj94p7s6kzgifoct6t",
        wsHost: "apidabt.cayan.co",
        wsPort: 45133,
        wssPort: 45133,
        cluster: "mt1",
        forceTLS: false,
        encrypted: true,
        disableStats: true,
        enabledTransports: ["ws"],
      });

      // Listen for connection
      this.echo.connector.pusher.connection.bind("connected", () => {
        this.connectionStatus = "Connected";
        console.log("Connected to Reverb");
      });

      this.echo.connector.pusher.connection.bind("disconnected", () => {
        this.connectionStatus = "Disconnected";
        console.log("Disconnected from Reverb");
      });

      this.echo.connector.pusher.connection.bind("error", (error) => {
        this.connectionStatus =
          "Error: " + (error?.data?.code || error?.message || "Unknown");
        console.error("Reverb connection error:", error);
      });

      // Log connection state
      const connState = this.echo.connector.pusher.connection.state;
      console.log("Connection state:", connState);

      // Subscribe to channel
      this.subscribeToChannel();
    },
    subscribeToChannel() {
      // Subscribe to public channel
      this.channel = this.echo.channel("company.2.tracking.vehicles");

      console.log("Channel object:", this.channel);
      console.log("Channel name:", this.channel.name);

      // Access the underlying Pusher channel to listen to all events
      const pusherChannel = this.echo.connector.pusher.channel(
        "company.2.tracking.vehicles"
      );
      console.log("Pusher channel:", pusherChannel);

      // Bind to all events using * wildcard
      if (pusherChannel) {
        pusherChannel.bind_global((eventName, data) => {
          console.log("=== GLOBAL EVENT ===");
          console.log("Event name:", eventName);
          console.log("Event data:", data);
          this.handleVehicleData(data);
        });
      }

      // Also use Echo listen with explicit event names
      this.channel.listen("tracking.vehicles", (eventData) => {
        console.log("=== Echo: tracking.vehicles ===");
        console.log("Event data:", eventData);
        this.handleVehicleData(eventData);
      });

      console.log(
        "Channel subscription initialized: company.2.tracking.vehicles"
      );
    },

    handleVehicleData(eventData) {
      try {
        // The event data might be the vehicles array directly
        let parsedData = eventData;

        // If it's wrapped in a data property, unwrap it
        if (eventData && eventData.data) {
          parsedData = eventData.data;
          if (typeof parsedData === "string") {
            parsedData = JSON.parse(parsedData);
          }
        }

        // Ensure it's an array
        if (Array.isArray(parsedData)) {
          this.vehicles = parsedData;
          console.log("Vehicles updated:", parsedData.length, "vehicles");
        } else if (parsedData && typeof parsedData === "object") {
          this.vehicles = [parsedData];
          console.log("Single vehicle updated");
        }

        // Log individual vehicle details
        this.vehicles.forEach((vehicle, index) => {
          console.log("Vehicle " + (index + 1) + ":", {
            id: vehicle.id,
            brand: vehicle.brand,
            model: vehicle.model,
            plate: vehicle.license_plate,
            status: vehicle.status,
          });
        });
      } catch (error) {
        console.error("Error parsing vehicle data:", error);
        console.log("Raw eventData:", eventData);
      }
    },
  },
  beforeUnmount() {
    if (this.echo) {
      this.echo.disconnect();
    }
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
code {
  background-color: #f5f5f5;
  padding: 2px 6px;
  border-radius: 3px;
  font-family: "Courier New", monospace;
}
.connection-status {
  margin: 20px 0;
  padding: 15px;
  background-color: #f0f0f0;
  border-radius: 5px;
  border-left: 4px solid #42b983;
}
.connection-status span.Connected {
  color: #28a745;
  font-weight: bold;
}
.connection-status span.Disconnected {
  color: #dc3545;
  font-weight: bold;
}
.connection-status span:not(.Connected):not(.Disconnected) {
  color: #ffc107;
  font-weight: bold;
}
.vehicle-data {
  margin: 20px 0;
  padding: 15px;
  background-color: #f9f9f9;
  border: 1px solid #ddd;
  border-radius: 5px;
}
.vehicle-data pre {
  background-color: #fff;
  padding: 10px;
  border-radius: 3px;
  overflow-x: auto;
}

.vehicle-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 15px;
  background-color: #fff;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.vehicle-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 2px solid #f0f0f0;
  padding-bottom: 10px;
  margin-bottom: 10px;
}

.vehicle-header h4 {
  margin: 0;
  color: #333;
}

.status {
  padding: 5px 12px;
  border-radius: 20px;
  font-size: 12px;
  font-weight: bold;
  text-transform: uppercase;
}

.status.moving {
  background-color: #d4edda;
  color: #155724;
}

.status.stopped {
  background-color: #f8d7da;
  color: #721c24;
}

.status.idle {
  background-color: #fff3cd;
  color: #856404;
}

.vehicle-info {
  font-size: 14px;
  line-height: 1.6;
  color: #666;
}

.vehicle-info p {
  margin: 5px 0;
}

.vehicle-info strong {
  color: #333;
}

.location-info,
.driver-info {
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid #f0f0f0;
}
</style>
