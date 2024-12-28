<template>
  <v-container>
    <v-card>
      <v-card-title>Orders</v-card-title>
      <v-data-table :headers="headers" :items="customerOrders" item-value="userId" class="elevation-1" dense>
        <template #item.actions="{ item }">
          <v-btn small color="primary" @click="viewDetails(item)">View Orders</v-btn>
        </template>
      </v-data-table>
    </v-card>

    <!-- Order Details Dialog -->
    <v-dialog v-model="dialogVisible" max-width="600px">
      <v-card>
        <v-card-title>
          Order Details
          <v-spacer></v-spacer>
          <v-btn icon @click="closeDialog">
            <v-icon>mdi-close</v-icon>
          </v-btn>
        </v-card-title>
        <v-card-text>
          <v-list>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Order ID:</strong> {{ selectedOrder.orderId }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Customer Name:</strong> {{ selectedOrder.customerName }} {{
                  selectedOrder.userLastName }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Product Name:</strong> {{ selectedOrder.productName }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Quantity:</strong> {{ selectedOrder.quantity
                  }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Payment Method:</strong> {{ selectedOrder.paymentMethod
                  }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Delivery Address:</strong> {{ selectedOrder.deliveryAddress
                  }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Total:</strong> {{ selectedOrder.total }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Status:</strong> {{ selectedOrder.status }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
          </v-list>
        </v-card-text>
        <v-card-actions>
          <v-btn text @click="closeDialog">Close</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>


<script>
import { collection, query, where, onSnapshot, getDoc, doc } from "firebase/firestore"; // Import necessary Firestore methods
import { firestore } from "~/plugins/firebase"; // Ensure Firestore is correctly initialized

export default {
  data() {
    return {
      headers: [
        { text: "Order ID", value: "orderId" },
        { text: "Date", value: "estimatedDeliveryDate" },
        { text: "Actions", value: "actions", sortable: false },
      ],
      orders: [],
      dialogVisible: false,
      selectedOrder: {},
      customerOrders: [], // Data property to hold customer order data
    };
  },
  methods: {
    // Fetch product name from Firestore
    async fetchProductName(productID) {
      try {
        const productDoc = await getDoc(doc(firestore, "Products", productID)); // Assume Products is the collection name
        if (productDoc.exists()) {
          return productDoc.data().productName || "Unknown Product";
        }
        return "Unknown Product";
      } catch (error) {
        console.error("Error fetching product name:", error);
        return "Unknown Product";
      }
    },

    async fetchCustomerOrders() {
      try {
        const customersQuery = query(collection(firestore, "Users"));

        onSnapshot(customersQuery, async (customersSnapshot) => {
          const customerOrders = [];

          for (const customerDoc of customersSnapshot.docs) {
            const customerData = customerDoc.data();
            const customerName = `${customerData.firstName} ${customerData.lastName}`;

            const ordersQuery = query(
              collection(firestore, "Orders"),
              where("userId", "==", customerDoc.id) // Filter orders by customer ID
            );

        onSnapshot(ordersQuery, async (ordersSnapshot) => {
        for (const orderDoc of ordersSnapshot.docs) {
            const orderData = orderDoc.data();
            // Loop through cartItems
            orderData.cartItems.forEach((item) => {
            
            // Check the status of the order
            if (orderData.status === "Pending") {
                const order = {
                orderId: orderDoc.id,
                customerName: customerName,
                paymentMethod: orderData.paymentMethod,
                productName: item.productName || "Unknown Products",
                total: orderData.total || 0,
                quantity: item.Quantity || "N/A",
                status: orderData.status || "Pending",
                deliveryAddress: orderData.deliveryAddress,
                estimatedDeliveryDate: orderData.estimatedDeliveryDate
                };

                // Push the order only if the status matches the criteria
                customerOrders.push(order);
            }
            });
        }

  // Update the component's data with a copy of the orders
  this.customerOrders = [...customerOrders];
});

          }
        });
      } catch (error) {
        console.error("Error fetching customer orders:", error);
      }
    },

    viewDetails(order) {
      this.selectedOrder = order;
      this.dialogVisible = true;
    },

    closeDialog() {
      this.dialogVisible = false;
    },
  },

  mounted() {
    this.fetchCustomerOrders(); // Fetch customer orders when the component is mounted
  },
};
</script>

