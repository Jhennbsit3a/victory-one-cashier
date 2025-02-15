<template>
  <v-container>
    <v-card>
      <v-card-title>Transaction History</v-card-title>
      
      <!-- Search Input -->
      <v-text-field
        v-model="searchQuery"
        label="Search by Order ID"
        class="mx-4"
        dense
        outlined
      ></v-text-field>
      
      <v-data-table
        :headers="headers"
        :items="filteredOrders"
        item-value="userId"
        class="elevation-1"
        dense
      >
        <template #item.actions="{ item }">
          <v-btn small color="primary" @click="viewDetails(item)">View Orders</v-btn>
          <v-btn small color="red" @click="showDeleteDialog(item)" class="white--text">Delete</v-btn>
        </template>
      </v-data-table>

            <!-- Delete Confirmation Dialog -->
      <v-dialog v-model="deleteDialogVisible" max-width="400px">
        <v-card>
          <v-card-title class="text-h5">Confirm Deletion</v-card-title>
          <v-card-text>Are you sure you want to delete this order together?</v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue darken-1" text @click="deleteDialogVisible = false">Cancel</v-btn>
            <v-btn color="blue darken-1" text @click="confirmDeleteOrder()">Yes</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>

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
                <v-list-item-title><strong>Customer Name:</strong> {{ selectedOrder.customerName }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Product Name:</strong> {{ selectedOrder.productName }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Quantity:</strong> {{ selectedOrder.quantity }}</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item>
              <v-list-item-content>
                <v-list-item-title><strong>Payment Method:</strong> {{ selectedOrder.paymentMethod }}</v-list-item-title>
              </v-list-item-content>
              <v-list-item-content>
                <v-list-item-title v-if="selectedOrder.paymentMethod !== 'Cash'"><strong>Reference No.:</strong> {{ selectedOrder.gcashReferenceNumber }}</v-list-item-title>
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
    </v-card>
  </v-container>
</template>


<script>
import { collection, query, where, onSnapshot, getDoc,getDocs, doc,deleteDoc   } from "firebase/firestore"; // Import necessary Firestore methods
import { firestore } from "~/plugins/firebase"; // Ensure Firestore is correctly initialized

export default {
  data() {
    return {
      headers: [
        { text: "Order ID", value: "orderId" },
        { text: "Product Name", value: "productName" },
        { text: "Total", value: "total" },
        { text: "Quantity", value: "quantity" },
        { text: "Status", value: "status" },
        { text: "Date", value: "estimatedDeliveryDate" },
        { text: "Action", value: "actions" },
      ],
      searchQuery: "", // For search input
      orders: [],
      dialogVisible: false,
      selectedOrder: {},
      customerOrders: [], // Data property to hold customer order data
      deleteDialogVisible: false,
      itemToDelete: null,
    };
  },
  computed: {
    // Filter orders based on search query
    filteredOrders() {
      if (!this.searchQuery) {
        return this.customerOrders;
      }
      return this.customerOrders.filter((order) =>
        order.orderId.toLowerCase().includes(this.searchQuery.toLowerCase())
      );
    },
  },
  methods: {
    viewDetails(order) {
      this.selectedOrder = order;
      this.dialogVisible = true;
    },
    showDeleteDialog(item) {
      this.itemToDelete = item;
      this.deleteDialogVisible = true;
      // console.log(item.orderId)
    },
    async confirmDeleteOrder() {
      try {
        const orderRef = doc(firestore, "Orders", this.itemToDelete.orderId);
        await deleteDoc(orderRef);
        // console.log(`Order with ID ${this.itemToDelete.orderId} has been deleted successfully`);

        // Remove the item from the local state
        this.customerOrders = this.customerOrders.filter(order => order.orderId !== this.itemToDelete.orderId);

        this.deleteDialogVisible = false;
      } catch (error) {
        console.error("Error deleting order:", error);
      }
    },
//     async deleteOrder(orderId) {
//   try {
//     // Reference to the Firestore document
//     const orderRef = doc(firestore, "Orders", orderId);

//     // Delete the document
//     await deleteDoc(orderRef);
//     console.log(`Order with ID ${orderId} has been deleted successfully.`);
//   } catch (error) {
//     console.error(`Error deleting order with ID ${orderId}:`, error);
//   }
// },
  async deleteOrders(orderIds) {
    try {
      if (!Array.isArray(orderIds) || orderIds.length === 0) {
        console.error("Please provide an array of order IDs to delete.");
        return;
      }

      // Use Promise.all to delete multiple orders in parallel
      await Promise.all(
        orderIds.map(async (orderId) => {
          const orderRef = doc(firestore, "Orders", orderId);
          await deleteDoc(orderRef);
          console.log(`Order with ID ${orderId} has been deleted successfully.`);
        })
      );

      console.log("All specified orders have been deleted successfully.");
    } catch (error) {
      console.error("Error deleting orders:", error);
    }
  },
//     async updateQuantityField(documentId) {
//   try {
//     // Reference to the Firestore document
//     const orderRef = doc(firestore, "Orders", documentId);

//     // Fetch the document
//     const orderSnapshot = await getDoc(orderRef);
//     if (!orderSnapshot.exists()) {
//       console.error("Order not found!");
//       return;
//     }

//     // Get the document data
//     const orderData = orderSnapshot.data();

//     // Check if cartItems exists and is an array
//     if (Array.isArray(orderData.cartItems)) {
//       // Create a new array with updated keys
//       const updatedCartItems = orderData.cartItems.map((item) => {
//         const { quantity, ...rest } = item; // Destructure to remove Quantity
//         return {
//           ...rest,
//           Quantity: quantity || 0, // Rename Quantity to quantity with fallback
//         };
//       });

//       // Update the document with the modified cartItems
//       await updateDoc(orderRef, { cartItems: updatedCartItems });
//       console.log("Cart items updated successfully!");
//     } else {
//       console.error("cartItems is missing or not an array!");
//     }
//   } catch (error) {
//     console.error("Error updating cart items:", error);
//   }
// },
    // Fetch product name from Firestore
    // async fetchProductName(productID) {
    //   try {
    //     const productDoc = await getDoc(doc(firestore, "Products", productID)); // Assume Products is the collection name
    //     if (productDoc.exists()) {
    //       return productDoc.data().productName || "Unknown Product";
    //     }
    //     return "Unknown Product";
    //   } catch (error) {
    //     console.error("Error fetching product name:", error);
    //     return "Unknown Product";
    //   }
    // },

    async fetchCustomerOrders() {
      try {
        const customersQuery = query(collection(firestore, "Users"));

        onSnapshot(customersQuery, async (customersSnapshot) => {
          const customerOrders = [];

          const additionalOrdersQuery = query(
            collection(firestore, "Orders"),
            where("userId", "==", "walkin_customer")
          );

          const additionalOrdersSnapshot = await getDocs(additionalOrdersQuery);

          const additionalOrders = additionalOrdersSnapshot.docs.map((doc) => {
            const data = doc.data();
            return {
              ...data,
              orderId: doc.id,
              total: data.subtotal || 0,
              status: data.status || "Unknown",
              // estimatedDeliveryDate: data.estimatedDeliveryDate
            };
          });

          const processedAdditionalOrders = additionalOrders.flatMap((order) => {
            if (Array.isArray(order.cartItems)) {
              return order.cartItems.map((item) => ({
                orderId: order.orderId,
                productName: item.productName || "Unknown Product",
                quantity: item.Quantity || 0,
                total: order.total,
                status: order.status,
                estimatedDeliveryDate: order.estimatedDeliveryDate,
                paymentMethod: order.paymentMethod,
                gcashReferenceNumber: order.referenceNumber
              }));
            }
            return [];
          });

          for (const customerDoc of customersSnapshot.docs) {
            const customerData = customerDoc.data();
            const customerName = `${customerData.firstName} ${customerData.lastName}`;

            const ordersQuery = query(
              collection(firestore, "Orders"),
              where("userId", "==", customerDoc.id)
            );

            onSnapshot(ordersQuery, (ordersSnapshot) => {
              for (const orderDoc of ordersSnapshot.docs) {
                const orderData = orderDoc.data();
                // console.table(orderData)

                orderData.cartItems.forEach((item) => {
                  if (orderData.status === "Completed" || orderData.status === "Cancelled") {
                    const order = {
                      orderId: orderDoc.id,
                      customerName,
                      productName: item.productName || "Unknown Products",
                      total: orderData.total || 0,
                      quantity: item.Quantity || "N/A",
                      status: orderData.status || "Pending",
                      estimatedDeliveryDate: orderData.estimatedDeliveryDate
                    };
                    customerOrders.push(order);
                  }
                });
              }

              const flattenedAdditionalOrders = processedAdditionalOrders.flat();

              const allOrders = [...customerOrders, ...flattenedAdditionalOrders];

              this.customerOrders = [...allOrders];
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
//     const orderIdsToDelete = [];
// const orderData = [
//   "2Z4VZTZ0RGOMLB1aJez6",
//   "5fFjAgiHWUFmNBheO1Zs",
//   "Bz6M18QhKzqqPtLMiA7K",
//   "JDCOUR6UXpyMbseylY25",
//   "OAI3u2zkG2wivU69K8Ig",
//   "Sa4KAALap3pcCpz45EpF",
//   "T1CeQo8RXOHJj0IVPAUK",
//   "VTK9UnfcwblvYUj7oOTl",
//   "YH6bRCEATcbRF0aT7gqY",
//   "cLQ92K1L5MlbvtiIcg9C",
//   "eZr8QTdc7e2ovkMPmrGU",
//   "lo0XqVeIUe5r1UsOTkwI",
//   "rP6gfDQwb3njXV4U7Q0w",
//   "s3eWOD6X6nqAbfh3DRPq",
//   "uJ6J95exQICZXhOj1HpS",
//   "wMRrBjPvJmwSSCVBRg4K"
// ];


    // this.updateQuantityField("3ZQ6U8vU1viw0Y3fiDGD");
    // this.deleteOrders(orderData);
  },
};
</script>

