<template>
  <v-container class="d-flex justify-center" style="height: 100vh;">
    <v-row justify="space-between" align="start">
      <!-- First Column -->
      <v-col cols="12" md="6">
        <v-card class="elevation-3">
          <v-card-title class="headline text-center">
            Order ID Details
          </v-card-title>

          <v-card-text>
            <!-- Input Field for UID -->
            <v-text-field
              label="Enter Order ID"
              v-model="orderUID"
              outlined
              dense
              @change="fetchOrderDetails"
              autofocus
            ></v-text-field>
            <!-- Reset Button -->
            <v-btn color="primary" class="mt-1 reset-btn" @click="resetForm" fab small>
              <v-icon>mdi-refresh</v-icon>
            </v-btn>

            <!-- User Information Section -->
            <v-card class="mt-3" v-if="userDetails && Object.keys(userDetails).length > 0">
              <v-card-title>User Information</v-card-title>
              <v-simple-table>
                <thead>
                  <tr>
                    <th>Field</th>
                    <th>Value</th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td>Customer Name</td>
                    <td>{{ userDetails.firstName }} {{ userDetails.lastName }}</td>
                  </tr>
                </tbody>
              </v-simple-table>
            </v-card>

            <!-- Order Information Section -->
            <v-card class="mt-5" v-if="orderDetails">
              <v-card-title>Order Information</v-card-title>
              <v-simple-table>
                <thead>
                  <tr>
                    <th>Field</th>
                    <th>Value</th>
                  </tr>
                </thead>
                <tbody>
                  <tr>
                    <td>Payment Method</td>
                    <td>{{ orderDetails.paymentMethod }}</td>
                  </tr>
                  <tr>
                    <td>Delivery Address</td>
                    <td>{{ orderDetails.deliveryAddress }}</td>
                  </tr>
                  <tr>
                    <td>Estimated Delivery Date</td>
                    <td>{{ orderDetails.estimatedDeliveryDate }}</td>
                  </tr>
                  <tr v-for="(item, index) in orderDetails.cartItems" :key="index">
                    <td>{{ item.productName }}</td>
                    <td>Qty - {{ item.Quantity }}</td>
                  </tr>
                  <tr>
                    <td>Total</td>
                    <td>{{ orderDetails.total }}</td>
                  </tr>
                </tbody>
              </v-simple-table>
            </v-card>

            <!-- Error Alert for Invalid UID -->
            <v-alert v-if="invalidOrder" type="error" class="mt-3" prominent>
              Invalid Order UID or the Order does not exist.
            </v-alert>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Second Column -->
      <!-- Second Column -->
      <v-col cols="12" md="6">
        <v-card class="pa-16 elevation-3">
          <v-card-title>
            <h3>Payment Method</h3>
          </v-card-title>
          <!-- GCASH -->
          <v-card class="elevation-3 hover-card cursor-pointer" @click="openGcashDialog">
            <v-card-title>
              <h3>GCASH</h3>
            </v-card-title>
          </v-card>

          <!-- CASH -->
          <v-card class="mt-5 elevation-3 hover-card cursor-pointer">
            <v-card-title>
              <h3>CASH</h3>
            </v-card-title>

            <v-card-text>
              <v-text-field
                label="Total Amount"
                v-model.number="totalAmount"
                outlined
                dense
              ></v-text-field>

              <v-text-field
                label="Cash Given"
                v-model.number="cashGiven"
                outlined
                dense
              ></v-text-field>

              <v-btn color="primary" class="mt-2" @click="calculateChange">
                Calculate Change
              </v-btn>

              <v-alert type="success" class="mt-3" v-if="change !== null">
                Change: {{ change.toFixed(2) }}
              </v-alert>
            </v-card-text>
          </v-card>
        </v-card>
      </v-col>
    </v-row>

          <!-- GCash Info Dialog -->
      <v-dialog v-model="gcashDialog" max-width="500px">
        <v-card>
          <v-card-title class="headline">GCash Payment</v-card-title>
          <v-card-text>
            <div class="text-center">
              <img src="@/assets/Gcash.jpg" alt="GCash Image" class="gcash-image" />
              <p class="gcash-description">Scan the QR code to pay via GCash.</p>
            <v-text-field
            label="Gcash Reference Number"
            required
          ></v-text-field>         
            </div>
          </v-card-text>
          <v-card-actions>
            <v-btn @click="closeGcashDialog" color="green">Done</v-btn>
            <v-btn @click="closeGcashDialog" color="red">Cancel</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
  </v-container>
</template>


<script>
import { doc, getDoc } from "firebase/firestore";
import { firestore } from "~/plugins/firebase";


export default {
  data() {
    return {
      scannedData: "", // Scanned data from QR scanner
      orderUID: "", // UID input
      orderDetails: null, // Fetched order details
      userDetails: {}, // User details
      invalidOrder: false, // Invalid order 
      gcashDialog: false,
      totalAmount: 0,
      cashGiven: 0,
      change: null,
    };
  },
  computed: {
    shouldFocus() {
      // Return true when you want the field to be focused
      // For example, after navigating to this component
      return this.$route.name === 'YourRouteName'
    }
  },
  methods: {
    calculateChange() {
      if (this.cashGiven >= this.totalAmount) {
        this.change = this.cashGiven - this.totalAmount;
      } else {
        this.change = null;
        alert('Insufficient cash provided!');
      }
    },
    // Fetch order details by UID
    async fetchOrderDetails() {
        console.log(sessionStorage.getItem("orderId"))
      try {
        if (!this.orderUID.trim()) {
        //   this.invalidOrder = true;
        //   this.orderDetails = null;
            this.resetForm()
          return;
        }

        const orderRef = doc(firestore, "Orders", this.orderUID.trim());
        const orderSnap = await getDoc(orderRef);

        if (orderSnap.exists()) {
          this.orderDetails = orderSnap.data(); // Set order details
          this.invalidOrder = false; // Reset invalid flag

          // Fetch user details using the userId from the order
          const userRef = doc(firestore, "Users", this.orderDetails.userId);
          const userSnap = await getDoc(userRef);
          if (userSnap.exists()) {
            this.userDetails = userSnap.data(); // Set user details
          } else {
            this.userDetails = {}; // Reset if user not found
          }
        } else {
          this.orderDetails = null;
          this.invalidOrder = true; // UID not found
        }
      } catch (error) {
        console.error("Error fetching order details:", error);
        this.orderDetails = null;
        this.invalidOrder = true;
      }
    },
    openGcashDialog() {
      this.gcashDialog = true; // Show GCash dialog
    },
    closeGcashDialog() {
      this.gcashDialog = false; // Close GCash dialog
    },

    // Reset form fields
    resetForm() {
      this.scannedData = "";
      this.orderUID = "";
      this.orderDetails = null;
      this.userDetails = {};
      this.invalidOrder = false;
    },
  },
};
</script>

<style scoped>
.v-container {
  max-width: 600px;
  margin: 0 auto;
}

.v-card {
  background-color: #f5f5f5;
}

.v-card-title {
  font-size: 1.5rem;
  font-weight: 600;
  color: #1976d2;
}

.scanned-data-textarea {
  background-color: #ffffff;
  border-radius: 8px;
  font-size: 1rem;
  width: 100%;
}

.reset-btn {
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
}

.v-alert {
  padding: 15px;
}
/* Product Card Styles */
.hover-card {
  transition: transform 0.3s, box-shadow 0.3s;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.hover-card:hover {
  transform: scale(1.05);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}
.cursor-pointer {
  cursor: pointer;
}
.gcash-image {
  width: 100%;
  height: auto;
  max-width: 300px;
  margin-bottom: 10px;
}

.gcash-description {
  font-size: 1.2rem;
  color: #444;
}
</style>
