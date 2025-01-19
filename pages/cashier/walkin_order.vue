<template>
  <v-app>
    <v-container class="p-0">
      <v-row>
        <!-- Products Section -->
        <v-col :cols="cart.length > 0 ? 8 : 12">
          <!-- MAIN PRODUCTS Header -->
          <v-row class="mb-4">
            <v-col cols="12">
              <div class="header-container">
                <h2 class="header-title">MAIN PRODUCTS</h2>
                <v-divider class="header-divider"></v-divider>
              </div>
            </v-col>
          </v-row>

          <!-- Search Field -->
          <v-row class="mb-4">
            <v-col cols="12">
              <v-text-field
                v-model="searchQuery"
                label="Search Products"
                clearable
                placeholder="Search by product name"
                @input="searchProducts"
              ></v-text-field>
            </v-col>
          </v-row>


          <!-- Product Grid -->
          <v-row>
            <template v-if="loading">
              <v-col cols="12" md="4" v-for="n in 6" :key="n">
                <v-skeleton-loader type="card" />
              </v-col>
            </template>
            <template v-else>
              <v-col v-for="product in filteredProducts" :key="product.id" cols="12" md="4">
                <v-card class="hover-card" @click.stop="addToCart(product)">
                  <v-img :src="product.image" height="200px"></v-img>
                  <v-card-title class="d-flex justify-space-between align-center">
                    <span class="product-name">{{ product.name }}</span>
                  </v-card-title>
                  <v-card-subtitle class="product-price">₱{{ product.price }}</v-card-subtitle>
                </v-card>
              </v-col>
            </template>
          </v-row>
        </v-col>
        <!-- Vertical Divider -->
        <v-divider v-if="cart.length > 0" vertical class="vertical-divider"></v-divider>
  <!-- Cart Section -->
  <v-col v-if="cart.length > 0" cols="4" class="sticky-cart">
    <div class="header-container text-center">
      <h2>PRODUCT CHECK OUT</h2>
      <v-divider class="header-divider"></v-divider>
    </div>

    <!-- Customer Details -->
    <!-- <v-card class="mb-3">
      <v-card-title>
        <h3>Customer Details</h3>
      </v-card-title>
      <v-card-text>
        <p><strong>Name:</strong> {{ customerDetails.name }}</p>
        <p><strong>Address:</strong> {{ customerDetails.address }}</p>
      </v-card-text>
    </v-card> -->

    <!-- Cart List -->
    <v-list>
      <v-list-item-group>
        <v-list-item v-for="(item, index) in cart" :key="item.id" class="bordered-item">
          <v-list-item-content>
            <!-- Clickable Title -->
            <v-menu
              bottom
              offset-y
              :close-on-content-click="false"
              transition="slide-y-transition"
            >
              <template #activator="{ on, attrs }">
                <v-list-item-title
                  v-bind="attrs"
                  v-on="on"
                  class="cursor-pointer"
                >
                  <div class="d-flex justify-space-between align-center border">
                    <span>{{ index + 1 }}. {{ item.name }}</span>
                    <span>₱{{ item.price }}</span>
                  </div>
                </v-list-item-title>
              </template>

              <!-- Dropdown Content -->
              <v-card>
                <v-card-text>
                  <div class="d-flex justify-space-between align-center">
                    <span>Quantity:</span>
                    <div class="quantity-control">
                      <v-btn small outlined @click.stop="updateQuantity(item.id, -1)">-</v-btn>
                      <span class="mx-2">{{ item.Quantity }}</span>
                      <v-btn small outlined @click.stop="updateQuantity(item.id, 1)">+</v-btn>
                    </div>
                  </div>
                </v-card-text>
              </v-card>
            </v-menu>
          </v-list-item-content>

          <!-- Trash Icon -->
          <v-list-item-icon>
            <v-icon color="red" @click="removeFromCart(item.id)" class="cursor-pointer">
              mdi-trash-can
            </v-icon>
          </v-list-item-icon>
        </v-list-item>
        <v-divider></v-divider>
      </v-list-item-group>
    </v-list>

    <!-- Total and Checkout -->
    <div class="cart-summary mt-3">
      <p class="total-text">Total: <strong>₱{{ cartTotal }}</strong></p>
      <v-btn
        class="buy-now-btn"
        block
        @click="proceedToPayment"
        :disabled="cart.length === 0 || loading2"
        color="orange"
        dark
      >
        <template v-if="loading2">
          <v-progress-circular indeterminate size="24" color="orange" />
        </template>
        <template v-else>
          Continue to Payment
        </template>
      </v-btn>
    </div>
  </v-col>
      </v-row>

    </v-container>
  </v-app>
</template>

<script>
import { firestore } from '~/plugins/firebase';
import {
  collection,
  addDoc,
  onSnapshot,
  serverTimestamp,doc,getDocs
} from 'firebase/firestore';
import { getAuth } from "firebase/auth";
import QRCode from 'qrcode'; // Import QRCode library

export default {
  data() {
    return {
      products: [],
      filteredProducts: [],
      itemList: [],
      cart: [],
      searchQuery: '',
      loading: true,
      loading2: false,
      dialog: false, // Confirmation dialog state
      qrCodeDialog: false, // QR Code dialog state
      showDrawer: true,
    };
  },
  computed: {
    cartTotal() {
      // console.log('Calculating cart total...');
      return this.cart.reduce((total, item) => {
        const price = isNaN(parseFloat(item.price)) ? 0 : parseFloat(item.price);
        // console.log(`Item: ${item.name}, Price: ${price}, Quantity: ${item.Quantity}`);
        return total + price * item.Quantity;
      }, 0);
    },
  },
  async created() {
    try {
      onSnapshot(collection(firestore, 'Categories'), (snapshot) => {
        this.itemList = snapshot.docs.map((doc) => ({
          id: doc.id,
          ProductType: doc.data().ProductType,
        }));
      });
      // Displaying firestore Data for checking
      const ordersRef = collection(firestore, "Orders");

      try {
        const querySnapshot = await getDocs(ordersRef);
        
        const ordersData = [];
        querySnapshot.forEach((doc) => {
          ordersData.push({ id: doc.id, ...doc.data() });
        });
        // console.log(ordersData);
      } catch (error) {
        console.error("Error fetching data:", error);
      }
      // Displaying firestore Data for checking

    onSnapshot(collection(firestore, 'Products'), (snapshot) => {
      this.products = snapshot.docs.map((doc) => {
        const rawData = doc.data();
        // console.log('Raw product data:', rawData);
        const price = parseFloat(rawData.Price);
        // console.log('Parsed price:', price);
        return {
          id: doc.id,
          name: rawData.ProductName,
          productName: rawData.ProductName,
          price: price,
          image: rawData.Image,
        };
      });
      // console.log('Processed products:', this.products);
      this.filteredProducts = this.products;
    });
  } catch (error) {
    console.error('Error fetching data:', error);
  }
  },
  methods: {
    async proceedToPayment() {
      if (this.cart.length === 0) {
        alert("Your cart is empty! Please add items before proceeding.");
        return;
      }

      this.loading2 = true; // Show loading animation

      try {
        const now = new Date(Date.now());
        const options = { weekday: 'short', month: 'short', day: 'numeric', year: 'numeric' };
        const formattedDate = now.toLocaleDateString('en-US', options).replace(',', '');

        // const auth = getAuth();
        // const user = auth.currentUser;

        // if (!user) {
        //   alert("You must be logged in to proceed.");
        //   this.loading = false;
        //   return;
        // }
        // Calculate totals
        const subtotal = this.cartTotal;
        const total = subtotal; // Assuming no additional charges for simplicity

        // // Prepare customer data
        // const customerData = {
        //   firstName: sessionStorage.getItem('customerFirstName'),
        //   lastName: sessionStorage.getItem('customerLastName'),
        //   role: "customer", // Assign the role
        // };
        // Prepare order data
        const orderData = {
          userId: "walkin_customer",
          cartItems: this.cart,
          paymentMethod: "Not yet Paid",
          subtotal: subtotal,
          total: total,
          status: "walk-in-order",
          createdAt: formattedDate,
        };

        // Save order data under autogenerated ID in the Orders collection
        const orderRef = collection(firestore, "Orders");
        const orderDocRef = await addDoc(orderRef, orderData);
        // const usersRef2 = collection(firestore, "Users");
        // const userDocRef2 = await addDoc(usersRef2, customerData);

        // // Save walkin-order under the autogenerated document ID
        // const walkinOrderRef = doc(orderRef, orderDocRef.id, "walkin-order");
        // await setDoc(walkinOrderRef, orderData);

        // Save the order ID to session storage
        const orderId = orderDocRef.id;
        sessionStorage.setItem("orderId", orderId);
        // sessionStorage.setItem("orderId", orderDocRef.data);

        // Route to payment method page
        this.$router.push("/cashier/payment_method");
      } catch (error) {
        console.error("Error saving order:", error);
        alert("There was an issue saving your order. Please try again.");
      } finally {
        this.loading = false; // Hide loading animation
      }
    },
    filterProducts(category) {
      this.searchQuery = '';
      this.filteredProducts = this.products.filter(
        (product) => product.ProductType === category.ProductType
      );
    },
    searchProducts() {
      const query = this.searchQuery.toLowerCase();
      this.filteredProducts = this.products.filter((product) =>
        product.name.toLowerCase().includes(query)
      );
    },
    updateQuantity(productId, change) {
      const product = this.cart.find((item) => item.id === productId);
      if (product) {
        product.Quantity = Math.max(product.Quantity + change, 1);
      }
    },
    addToCart(product) {
      // console.log('Adding to cart:', product);
      const existingProduct = this.cart.find((item) => item.id === product.id);
      if (existingProduct) {
        existingProduct.Quantity += 1;
      } else {
        const cartItem = {
          ...product,
          Quantity: 1,
          price: parseFloat(product.price)
        };
        // console.log('CartItem before push:', cartItem);
        this.cart.push(cartItem);
      }
      // console.log('Updated cart:', this.cart);
    },
    removeFromCart(productId) {
      this.cart = this.cart.filter((item) => item.id !== productId);
    },
    async checkout() {
  if (this.cart.length === 0) {
    alert('Your cart is empty! Please add some items to proceed.');
    return; // Stop the checkout process if the cart is empty
  }

  this.loading2 = true;

  try {
    // Define the order data
    const order = {
      userId: 'user123', // Replace with the actual user ID if available
      cartItems: this.cart.map((item) => ({
        id: item.id,
        productName: item.productName, // Use `productName` instead of `name`
        price: item.price,
        quantity: item.Quantity,
      })),
      deliveryAddress: '123 Main Street, City', // Replace with the actual delivery address
      paymentMethod: 'Cash on Delivery', // Replace with the actual payment method
      subtotal: this.cartTotal, // Total without tax
      tax: this.cartTotal * 0.12, // Assuming 12% tax
      total: this.cartTotal * 1.12, // Total with tax
      estimatedDeliveryDate: new Date(Date.now() + 5 * 24 * 60 * 60 * 1000), // 5 days from now
      status: 'Pending',
      createdAt: new Date(),
    };

    // Save the order in Firestore
    const docRef = await addDoc(collection(firestore, 'Orders'), order);

    // Generate the QR Code for the order
    const qrData = docRef.id;

    this.qrCodeDialog = true; // Trigger dialog to open

    // this.$nextTick(() => this.generateQRCode(qrData)); // Generate QR after rendering

    // Clear the cart
    this.cart = [];
  } catch (error) {
    console.error('Error during checkout:', error);
  } finally {
    this.loading2 = false;
  }
},
    generateQRCode(data) {
      this.$nextTick(() => {
        const canvas = this.$refs.qrCanvas;
        if (canvas) {
          QRCode.toCanvas(canvas, data, { width: 200 }, (error) => {
            if (error) {
              console.error('QR Code generation error:', error);
            } else {
              console.log('QR Code generated successfully');
            }
          });
        } else {
          console.error('QR Code canvas not found');
        }
      });
    },
    closeDialog() {
      this.dialog = false;
    },
    goBack() {
      this.qrCodeDialog = false; // Close QR Code dialog
    },
  },
  mounted() {
    setTimeout(() => {
      // Simulate data loading
      this.products = [
        { id: 1, name: "Product 1", price: 100, image: "image1.jpg" },
        { id: 2, name: "Product 2", price: 200, image: "image2.jpg" },
        { id: 3, name: "Product 3", price: 300, image: "image3.jpg" },
      ];
      this.loading = false;
    }, 4000); // Simulates 2-second delay
    const customerName = sessionStorage.getItem('customerFirstName') + " " + sessionStorage.getItem('customerLastName');
    const customerAddress = sessionStorage.getItem('customerAddress');
    if (customerName && customerAddress) {
      this.customerDetails.name = customerName;
      this.customerDetails.address = customerAddress;
    }
    // console.table(this.cart)
  },
};
</script>

<style scoped>
/* Add your styles here */
.sticky-cart {
  position: -webkit-sticky;
  position: sticky;
  top: 50px; /* Set the offset distance from the top of the viewport */
  z-index: 10;
  max-height: calc(100vh - 60px); /* Ensure it does not overflow */
  overflow-y: auto;
}

.qr-description {
  margin-top: 10px;
  font-size: 14px;
  color: #666;
}

/* Header Styles */
.header-container {
  text-align: center;
  position: relative;
  margin-bottom: 20px;
}

.header-title {
  font-size: 36px;
  font-weight: bold;
  color: #333;
}

.header-divider {
  background-color: #FFA900;
  height: 4px;
  width: 80px;
  margin: 0 auto;
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

.product-name {
  font-size: 18px;
  font-weight: bold;
  color: #333;
}

.product-price {
  font-size: 16px;
  color: #FFA900;
}

.cart-table th, .cart-table td {
  text-align: inherit; /* Ensures header and data cells share alignment */
  padding: 8px;        /* Adds consistent spacing */
}

.cart-summary {
  margin-top: 10px;
  align-items: center;
}

.cart-table td {
  vertical-align: middle;
}

.cart-table th {
  text-align: center;
}

.total-container {
  display: flex;
  align-items: center;
}

.total-text {
  font-size: 18px;
  font-weight: bold;
  color: #333;
}

/* Checkout Button */
.buy-now-btn {
  background-color: #FFA900;
  color: white;
  font-weight: bold;
  border-radius: 8px;
  padding: 10px 20px;
  transition: background-color 0.3s;
}

.buy-now-btn:hover {
  background-color: #FF8C00;
}
.cart-dropdown-btn {
  border-radius: 8px;
  font-weight: bold;
  margin-bottom: 10px;
}

.cart-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 15px;
  border-bottom: 1px solid #f0f0f0;
}

.cart-item:last-child {
  border-bottom: none;
}
.quantity-control {
  display: flex;
  align-items: center;
}
.cursor-pointer {
  cursor: pointer;
}
.bordered-item {
  border: 1px solid #ddd;
  border-radius: 8px;
  margin-bottom: 8px; /* Adds spacing between items */
}
</style>