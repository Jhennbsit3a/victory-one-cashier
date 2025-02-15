<template>
  <v-container class="d-flex justify-center" style="height: 100vh;">
    <v-row justify="space-between" align="start">
      <!-- First Column -->
      <v-col cols="12" md="6">
        <v-card class="elevation-3">
          <v-card-title class="headline text-center">
            Search Order
          </v-card-title>

          <v-card-text>
            <!-- Input Field for UID -->
            <v-text-field
              label="Enter Order ID"
              v-model="orderUID"
              outlined
              dense
              autofocus
              ref="orderUIDField"
              @change="fetchOrderDetails"
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
                    <td v-if="orderDetails.userId !== 'walkin_customer'">{{ orderDetails.estimatedDeliveryDate }}</td>
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
            <v-alert v-if="paid" type="error" class="mt-3" prominent>
              This order has already been paid and cannot be retrieved.
            </v-alert>
          </v-card-text>
        </v-card>
      </v-col>

      <!-- Second Column -->
      <v-col cols="12" md="6">
        <v-card class="pa-10 elevation-3">
            <div class="text-center pa-3">
              <h2>Payment Method</h2>
            </div>
          <!-- GCASH -->
          <v-card class="elevation-3 hover-card cursor-pointer" @click="openGcashDialog">
            <v-card-title>
              <v-img 
                    :src="require('@/assets/gcash-logo.png')"
                    alt="Gcash logo"
                    max-height="50"
                    max-width="50"
                    ></v-img>
              <h3 class="ps-3">GCASH</h3>
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
                :disabled="isFetching"
                :error="!!errorMessage"
              ></v-text-field>

              <v-text-field
                v-if="cashGiven !== null"
                label="Cash Given"
                v-model.number="cashGiven"
                outlined
                dense
                :error-messages="errorMessage" 
                :error="!!errorMessage" 
              ></v-text-field>
                  <!-- Change Display -->
              <v-alert v-if="change !== null && cashGiven >= totalAmount" dense>
                <!-- Change: ₱{{ change.toFixed(2) }} -->
                <p v-if="totalAmount !== null && totalAmount !== 0">
                  <strong>Total Amount:</strong> ₱{{ totalAmount.toFixed(2) }}
                </p>
                <p v-if="cashGiven !== null && cashGiven !== 0">
                  <strong>Cash Given:</strong> ₱{{ cashGiven.toFixed(2) }}
                </p>
                <p v-if="change !== null && change !== 0">
                  <strong>Change:</strong> ₱{{ change.toFixed(2) }}
                </p>
              </v-alert>

              <v-btn color="primary" class="mt-2" @click="calculateChange">
                Pay now
              </v-btn>
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
              <p class="gcash-instruction">
                Please follow the instructions below before proceeding with the payment:
              </p>
              <ol class="gcash-steps">
                <li>Scan the QR code below using your GCash app.</li>
                <li>Enter the exact payment amount and complete the transaction.</li>
                <li>Take a screenshot of the transaction receipt for reference.</li>
                <li>
                  Look on the <strong>Reference Number</strong> on your GCash receipt.  
                  This is a 13-digit number usually found in the transaction details.
                </li>
                <li>Enter the <strong>Reference Number</strong> in the field below before proceeding.</li>
              </ol>
              <img src="@/assets/Gcash.jpg" alt="GCash Image" class="gcash-image" />

              <!-- Input Fields -->
              <v-text-field 
                v-model="referenceNumber" 
                label="Reference Number"
                outlined
                dense
                maxlength="13"
                type="text"
                placeholder="Enter 13-digit reference number"
                @input="validateReferenceNumber"
              ></v-text-field>

              <v-text-field 
                v-model="totalAmount" 
                label="Total amount to pay" 
                outlined 
                dense
                :disabled="isFetching"
                required
              ></v-text-field>
            </div>
          </v-card-text>

          <v-card-actions class="justify-end">
            <v-btn @click="gcashPaid" color="green" :disabled="!isReferenceValid">
              Confirm Payment
            </v-btn>
            <v-btn @click="closeGcashDialog" color="red">Cancel</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

  <!-- Receipt Dialog -->
  <v-dialog v-model="receiptDialog" max-width="400px">
    <v-card>
      <v-card-text class="receipt-container" id="receipt-content">
        <!-- Receipt Header -->
        <div class="receipt-header text-center pt-5">
          <h3 class="company-name">VICTORY ONE</h3>
          <p class="company-description">San Matias, San Fernando, Pampanga</p>
          <p class="receipt-divider">----------------------------------------</p>
        </div>

        <!-- Receipt Body -->
        <div class="receipt-body">
          <p><strong>Date:</strong> {{ formattedDate }}</p>
          <p><strong>Receipt ID:</strong> {{ receiptId }}</p>
          <p class="receipt-divider">----------------------------------------</p>

          <!-- Product Information -->
          <div v-if="orderDetails">
            <h4>Order Details:</h4>
            <v-simple-table>
              <thead>
                <tr>
                  <th>Item</th>
                  <th>Qty</th>
                  <th>Price</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="(item, index) in orderDetails.cartItems" :key="index">
                  <td>{{ item.productName }}</td>
                  <td>{{ item.Quantity }}</td>
                  <td>₱{{ item.price }}</td>
                </tr>
              </tbody>
            </v-simple-table>
          </div>

          <p class="receipt-divider">----------------------------------------</p>
          <p v-if="orderDetails">
            <strong>Payment Method:</strong> {{ payment }}
          </p>
          <p v-if="payment === 'GCash'">
            <strong>Reference Number:</strong> {{ referenceNumber }}
          </p>
          <p v-if="totalAmount !== null && totalAmount !== 0">
            <strong>Total Amount:</strong> ₱{{ totalAmount.toFixed(2) }}
          </p>
          <p v-if="cashGiven !== null && cashGiven !== 0">
            <strong>Cash Given:</strong> ₱{{ cashGiven.toFixed(2) }}
          </p>
          <p v-if="change !== null && change !== 0">
            <strong>Change:</strong> ₱{{ change.toFixed(2) }}
          </p>
          <p class="receipt-divider">----------------------------------------</p>
        </div>

        <!-- Receipt Footer -->
        <div class="receipt-footer text-center">
          <p>Thank you for shopping with us!</p>
          <p>Visit Again!</p>
        </div>
      </v-card-text>
      
      <v-card-actions>
        <v-row class="w-100" dense>
          <v-col cols="6">
            <v-btn color="green" block @click="closeReceiptDialog">Done</v-btn>
          </v-col>
          <v-col cols="6">
            <v-btn color="blue" block @click="printReceipt">Print</v-btn>
          </v-col>
        </v-row>
      </v-card-actions>

    </v-card>
  </v-dialog>

  </v-container>
</template>


<script>
import { doc, getDoc,updateDoc } from "firebase/firestore";
import { firestore } from "~/plugins/firebase";
import { jsPDF } from "jspdf";
import html2canvas from "html2canvas";

export default {
  data() {
    return {
      referenceNumber: "",      // Stores the GCash reference number
      // receiptNumber: "",        // Stores the GCash receipt number
      isReferenceValid: false,  // Controls the "Confirm Payment" button state
      receiptId: '', // Example receipt ID
      payment:"",
      scannedData: "", // Scanned data from QR scanner
      orderUID: "", // UID input
      orderDetails: null, // Fetched order details
      userDetails: {}, // User details
      invalidOrder: false, // Invalid order 
      gcashDialog: false,
      totalAmount: 0,
      cashGiven: 0,
      change: null,
      receiptDialog: false,
      errorMessage: "",
      paid: false,
      isFetching: false,
    };
  },
  computed: {
    shouldFocus() {
      // Return true when you want the field to be focused
      // For example, after navigating to this component
      return this.$route.name === 'YourRouteName'
    },
    computedChange() {
      if (this.cashGiven >= this.totalAmount) {
        return this.cashGiven - this.totalAmount;
      }
      return null;
    },
    formattedDate() {
        const now = new Date(Date.now());
        const options = { weekday: 'short', month: 'short', day: 'numeric', year: 'numeric' };
        const formattedDate = now.toLocaleDateString('en-US', options).replace(',', '');
        return formattedDate
    },
  },
  watch:{
        // Sync computed change to `change` data
    cashGiven: 'updateChange',
    totalAmount: 'updateChange',
  },
  methods: {
    updateChange() {
      this.change = this.computedChange;
    },
    validateReferenceNumber() {
      this.referenceNumber = this.referenceNumber.replace(/\D/g, ""); // Remove non-numeric characters
      this.isReferenceValid = this.referenceNumber.length === 13; // Enable button only if 13 digits
    },
    generateReceiptId() {
      const now = new Date();
      const year = now.getFullYear();
      const month = String(now.getMonth() + 1).padStart(2, '0');
      const day = String(now.getDate()).padStart(2, '0');
      // const hours = String(now.getHours()).padStart(2, '0');
      const minutes = String(now.getMinutes()).padStart(2, '0');
      const seconds = String(now.getSeconds()).padStart(2, '0');

      this.receiptId = `${year}${month}${day}${minutes}${seconds}`;
    },
    saveReceiptAsImage() {
      const receiptElement = document.getElementById("receipt-content");
      html2canvas(receiptElement).then((canvas) => {
        const link = document.createElement("a");
        link.href = canvas.toDataURL("image/png");
        link.download = `receipt-${this.receiptId}`;
        link.click();
      });
    },
    async printReceipt() {
      const receiptElement = document.getElementById("receipt-content");

      if (!receiptElement) {
        console.error("Receipt element not found.");
        return;
      }

      try {
        const canvas = await html2canvas(receiptElement, { scale: 2 }); // High quality
        const imgData = canvas.toDataURL("image/png");

        // Standard receipt size: 80mm width, dynamic height
        const pdfWidth = 80; // 80mm width (standard)
        const pdfHeight = (canvas.height * pdfWidth) / canvas.width; // Maintain aspect ratio

        const pdf = new jsPDF("p", "mm", [pdfWidth, pdfHeight]); // Custom size
        pdf.addImage(imgData, "PNG", 0, 0, pdfWidth, pdfHeight);

        // Create an iframe to handle printing
        const iframe = document.createElement("iframe");
        iframe.style.position = "absolute";
        iframe.style.width = "0px";
        iframe.style.height = "0px";
        iframe.style.border = "none";
        document.body.appendChild(iframe);

        iframe.onload = () => {
          setTimeout(() => {
            iframe.contentWindow.focus(); // Ensure focus before printing
            iframe.contentWindow.print(); // Trigger print
          }, 500); // Delay to allow print dialog to open
        };

        // Write the PDF data to the iframe
        iframe.src = pdf.output("bloburl");
      } catch (error) {
        console.error("Error generating print:", error);
      }
    },


//     async saveReceiptAsPDF() {
//   const receiptElement = document.getElementById("receipt-content");

//   if (!receiptElement) {
//     console.error("Receipt element not found.");
//     return;
//   }

//   try {
//     const canvas = await html2canvas(receiptElement, { scale: 2 }); // High quality
//     const imgData = canvas.toDataURL("image/png");

//     // Standard receipt size: 80mm width, dynamic height
//     const pdfWidth = 80; // 80mm width (standard)
//     const pdfHeight = (canvas.height * pdfWidth) / canvas.width; // Maintain aspect ratio

//     const pdf = new jsPDF("p", "mm", [pdfWidth, pdfHeight]); // Custom size

//     pdf.addImage(imgData, "PNG", 0, 0, pdfWidth, pdfHeight);

//     // Save PDF file
//     const pdfFileName = `receipt-${this.receiptId}.pdf`;
//     pdf.save(pdfFileName);

//     // Open print dialog after a short delay (to ensure the file is saved first)
//     setTimeout(() => {
//       const printWindow = window.open(pdf.output("bloburl"), "_blank");
//       if (printWindow) {
//         printWindow.print(); // Trigger print dialog
//       } else {
//         console.error("Failed to open print dialog.");
//       }
//     }, 1000); // Adjust delay if needed
//   } catch (error) {
//     console.error("Error generating PDF:", error);
//   }
// },
    calculateChange() {
      if (this.cashGiven >= this.totalAmount && (this.totalAmount && this.cashGiven !== 0)) {
        this.generateReceiptId()
        this.change = this.cashGiven - this.totalAmount;
        this.receiptDialog = true; // Open the receipt dialog
        this.errorMessage = ""
        this.payment = "Cash"
      } else {
        this.errorMessage = "Input field must not 0 or less than to total amount value";
      }
    },
    gcashReceipt() {
        this.generateReceiptId()
        // this.change = this.cashGiven - this.totalAmount;
        this.change = 0;
        this.cashGiven = 0;
        this.totalAmount = 0;
        this.receiptDialog = true; // Open the receipt dialog
        this.errorMessage = ""
        this.payment = "GCash"
    },
    async closeReceiptDialog() {
      try {
        const orderRef = doc(firestore, "Orders", this.orderUID.trim());
        const orderSnap = await getDoc(orderRef);

        if (orderSnap.exists()) {
          const orderData = orderSnap.data();

          // Check if the order is already paid
          if (orderData.paymentMethod === "GCash" || orderData.paymentMethod === "Cash") {
            console.log("Order has already been paid.");
            this.clearCashierData();
            sessionStorage.clear();
            return;
          }

          // Update the payment method to Cash
          await updateDoc(orderRef, {
            paymentMethod: "Cash",
            // referenceNumber: this.referenceNumber.trim(), // Save reference number
            // paymentStatus: "Paid", // Optional: Add status for tracking
          });

          console.log("Payment successful with Cash.");
          this.clearCashierData();
          sessionStorage.clear();
          this.gcashDialog = false;
        } else {
          console.error("Order does not exist.");
        }
      } catch (error) {
        console.error("Error confirming order payment:", error);
      }
    },
    clearCashierData(){
      this.receiptDialog = false; // Close the receipt dialog
      this.totalAmount = 0;
      this.cashGiven = 0;
      this.resetForm()
    },
    async gcashPaid() {
      try {
        const orderRef = doc(firestore, "Orders", this.orderUID.trim());
        const orderSnap = await getDoc(orderRef);

        if (orderSnap.exists()) {
          const orderData = orderSnap.data();

          // Check if the order is already paid
          if (orderData.paymentMethod === "GCash" || orderData.paymentMethod === "Cash") {
            console.log("Order has already been paid.");
            this.clearCashierData();
            sessionStorage.clear();
            return;
          }

          // Update the payment method to GCash
          await updateDoc(orderRef, {
            paymentMethod: "GCash",
            referenceNumber: this.referenceNumber, // Save reference number
          });

          console.log("Payment successful through GCash.");
          this.gcashReceipt();
        } else {
          console.error("Order does not exist.");
        }

        this.gcashDialog = false;
      } catch (error) {
        console.error("Error processing payment:", error);
      }
    },
    // Fetch order details by UID
    async fetchOrderDetails() {
      try {
        if (!this.orderUID.trim()) {
        //   this.invalidOrder = true;
        //   this.orderDetails = null;
            this.resetForm()
          return;
        }

        const orderRef = doc(firestore, "Orders", this.orderUID.trim());
        const orderSnap = await getDoc(orderRef);
        // console.log(orderSnap.data())

        if (orderSnap.exists()) {
          this.orderDetails = orderSnap.data(); // Set order details
          this.invalidOrder = false; // Reset invalid flag
          this.totalAmount = this.orderDetails.total;
          this.isFetching = true; // Re-enable the text field
                // Check if the order is already paid
        if (this.orderDetails.paymentMethod === "GCash" || this.orderDetails.paymentMethod === "Cash") {
          // console.log("This order has already been paid and cannot be retrieved.");
          this.orderDetails = null;
          // this.invalidOrder = true;
          this.totalAmount = 0;
          this.paid = true;
          this.isFetching = false; // Re-enable the text field
          return;
        }else{
          this.paid = false;
        }
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
      this.orderUID = '';
      this.orderDetails = null;
      this.userDetails = {};
      this.invalidOrder = false;
      sessionStorage.clear();

      // Focus on the input field after resetting
      this.$nextTick(() => {
        this.$refs.orderUIDField.focus();
      });
    },
  },
  mounted(){
    // console.log(sessionStorage.getItem("orderId"))
    const walkinOrderId = sessionStorage.getItem("orderId")? sessionStorage.getItem("orderId"): "";
    this.orderUID = walkinOrderId;
    this.fetchOrderDetails()
  }
};
</script>

<style scoped>
.gcash-instructions {
  font-size: 14px;
  color: #333;
  background: #f3f4f6;
  padding: 10px;
  border-radius: 8px;
  margin-bottom: 15px;
  text-align: left;
  line-height: 1.5;
}
.gcash-steps{
  text-align: left;
}
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
/* Receipt Container Styling */
.receipt-container {
  background-color: white;
  padding: 20px;
  font-family: "Courier New", Courier, monospace;
  border: 1px solid #ccc;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
}

/* Header Styling */
.receipt-header {
  margin-bottom: 10px;
}
.company-name {
  font-size: 18px;
  font-weight: bold;
}
.company-description {
  font-size: 14px;
  color: gray;
}
.receipt-divider {
  text-align: center;
  font-family: "Courier New", monospace;
  margin: 5px 0;
  color: #aaa;
}

/* Body Styling */
.receipt-body p {
  margin: 5px 0;
}

/* Footer Styling */
.receipt-footer {
  margin-top: 10px;
  font-size: 14px;
  text-align: center;
  color: gray;
}
</style>
