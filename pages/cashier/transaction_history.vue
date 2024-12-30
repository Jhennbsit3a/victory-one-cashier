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
      </v-data-table>
    </v-card>
  </v-container>
</template>


<script>
import { collection, query, where, onSnapshot, getDoc,getDocs, doc  } from "firebase/firestore"; // Import necessary Firestore methods
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
      ],
      searchQuery: "", // For search input
      orders: [],
      dialogVisible: false,
      selectedOrder: {},
      customerOrders: [], // Data property to hold customer order data
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

                orderData.cartItems.forEach((item) => {
                  if (orderData.status === "Shipped" || orderData.status === "Cancelled") {
                    const order = {
                      orderId: orderDoc.id,
                      customerName,
                      productName: item.productName || "Unknown Products",
                      total: orderData.total || 0,
                      quantity: item.Quantity || "N/A",
                      status: orderData.status || "Pending",
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
    // this.updateQuantityField("3ZQ6U8vU1viw0Y3fiDGD");
    // this.deleteOrder("UxgwjhArYBh0uIIZiQEI");
  },
};
</script>

