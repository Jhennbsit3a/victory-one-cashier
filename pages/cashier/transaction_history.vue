<template>
  <v-container>
    <v-card>
      <v-card-title>Transaction History</v-card-title>
      <v-data-table :headers="headers" :items="customerOrders" item-value="userId" class="elevation-1" dense>
      </v-data-table>
    </v-card>
  </v-container>
</template>


<script>
import { collection, query, where, onSnapshot, getDoc,getDocs, doc } from "firebase/firestore"; // Import necessary Firestore methods
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

          // Query orders without filtering by userId
          const additionalOrdersQuery = query(
            collection(firestore, "Orders"),
            where("userId", "==", "walkin_customer") // Fetch all shipped or cancelled orders
          );

          const additionalOrdersSnapshot = await getDocs(additionalOrdersQuery);

          // Collect additional orders data
          const additionalOrders = additionalOrdersSnapshot.docs.map((doc) => ({
            ...doc.data(),
            orderId: doc.id,
            total: doc.total,
            status: doc.status
          }));
          // Process additionalOrders to extract productName and quantity
          const processedAdditionalOrders = additionalOrders.map((order) => {
            return order.cartItems.map((item) => {
              return {
                orderId: order.orderId,
                productName: item.productName || "Unknown Product",
                quantity: item.quantity || 0,
                total: order.total,
                status: order.status
              };
            });
          });

          for (const customerDoc of customersSnapshot.docs) {
            const customerData = customerDoc.data();
            const customerName = `${customerData.firstName} ${customerData.lastName}`;

            const ordersQuery = query(
              collection(firestore, "Orders"),
              where("userId", "==", customerDoc.id) // Filter orders by customer ID
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

              // Flatten processedAdditionalOrders into a single array
              const flattenedAdditionalOrders = processedAdditionalOrders.flat();

              // Merge additional orders with customerOrders
              const allOrders = [...customerOrders, ...flattenedAdditionalOrders];

              // Update the component's data
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
  },
};
</script>

