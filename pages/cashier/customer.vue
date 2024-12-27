<template>
    <v-app>
        <v-container>
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
                    <v-text-field v-model="searchQuery" label="Search Products" clearable
                        placeholder="Search by product name" @input="searchProducts"></v-text-field>
                </v-col>
            </v-row>

            <!-- Product Grid -->
            <v-row>
                <v-col v-for="product in filteredProducts" :key="product.id" cols="12" md="4">
                    <v-card class="hover-card">
                        <v-img :src="product.image" height="200px"></v-img>

                        <v-card-title class="d-flex justify-space-between align-center">
                            <span class="product-name">{{ product.name }}</span>
                            <v-btn icon class="cart-icon" @click.stop="addToCart(product)">
                                <v-icon style="color: #FFA900;">mdi-cart-plus</v-icon>
                            </v-btn>
                        </v-card-title>

                        <v-card-subtitle class="product-price">₱{{ product.price }}</v-card-subtitle>
                    </v-card>
                </v-col>
            </v-row>

            <!-- Cart Summary -->
            <v-row class="mt-5">
                <v-col cols="12">
                    <div class="header-container">
                        <h2 class="header-title">CART SUMMARY</h2>
                        <v-divider class="header-divider"></v-divider>
                    </div>

                    <v-data-table :headers="cartHeaders" :items="cart" item-value="id" class="elevation-1 cart-table"
                        dense>
                        <template v-slot:body="{ items }">
                            <tbody>
                                <tr v-for="item in items" :key="item.id">
                                    <!-- Ensure alignment matches the header -->
                                    <td class="text-start">{{ item.name }}</td>
                                    <td class="text-end">₱{{ item.price }}</td>
                                    <td class="text-center">{{ item.quantity }}</td>
                                    <td class="text-center">
                                        <v-btn icon @click="removeFromCart(item.id)">
                                            <v-icon color="red">mdi-delete</v-icon>
                                        </v-btn>
                                    </td>
                                </tr>
                            </tbody>
                        </template>
                    </v-data-table>

                    <v-row class="cart-summary">
                        <v-col cols="6" class="total-container">
                            <p class="total-text">Total: <strong>₱{{ cartTotal }}</strong></p>
                        </v-col>
                        <v-col cols="6" class="text-right">
                            <v-btn class="buy-now-btn" @click="checkout" :disabled="cart.length === 0 || loading">
                                <template v-if="loading">
                                    <v-progress-circular indeterminate size="24" color="white" />
                                </template>
                                <template v-else>
                                    Checkout
                                </template>
                            </v-btn>
                        </v-col>
                    </v-row>
                </v-col>
            </v-row>

            <!-- Checkout Confirmation Dialog -->
            <v-dialog v-model="dialog" max-width="400px">
                <v-card>
                    <v-card-title class="headline">Order Confirmation</v-card-title>
                    <v-card-text>Your order has been successfully placed!</v-card-text>
                    <v-card-actions>
                        <v-spacer></v-spacer>
                        <v-btn color="primary" text @click="closeDialog">OK</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>

            <!-- QR Code Dialog -->
            <v-dialog v-model="qrCodeDialog" max-width="400px" v-if="qrCodeDialog">
                <v-card>
                    <v-card-title class="headline">Order QR Code</v-card-title>
                    <v-card-text>
                        <div class="text-center">
                            <canvas ref="qrCanvas" width="200" height="200"></canvas>
                            <p class="qr-description">
                                You can use this QR code for pickup or as proof of your order.
                            </p>
                        </div>
                    </v-card-text>
                    <v-card-actions>
                        <v-btn @click="goBack" color="green" style="color: white;">Done</v-btn>
                    </v-card-actions>
                </v-card>
            </v-dialog>
        </v-container>
    </v-app>
</template>

<script>
import { firestore } from '~/plugins/firebase';
import {
    collection,
    addDoc,
    onSnapshot,
    serverTimestamp,
} from 'firebase/firestore';
import QRCode from 'qrcode'; // Import QRCode library

export default {
    data() {
        return {
            products: [],
            filteredProducts: [],
            itemList: [],
            cart: [],
            searchQuery: '',
            loading: false,
            dialog: false, // Confirmation dialog state
            qrCodeDialog: false, // QR Code dialog state
            cartHeaders: [
                { text: 'Product Name', value: 'name', align: 'start' },
                { text: 'Price', value: 'price', align: 'end' },
                { text: 'Quantity', value: 'quantity', align: 'center' },
                { text: 'Actions', value: 'actions', align: 'center', sortable: false },
            ],
        };
    },
    async created() {
        try {
            onSnapshot(collection(firestore, 'Categories'), (snapshot) => {
                this.itemList = snapshot.docs.map((doc) => ({
                    id: doc.id,
                    ProductType: doc.data().ProductType,
                }));
            });

            onSnapshot(collection(firestore, 'Products'), (snapshot) => {
                this.products = snapshot.docs.map((doc) => ({
                    id: doc.id,
                    name: doc.data().ProductName, // Map the new productName field
                    productName: doc.data().ProductName, // Add productName field
                    price: doc.data().Price,
                    image: doc.data().Image,
                }));
                this.filteredProducts = this.products;
            });
        } catch (error) {
            console.error('Error fetching data:', error);
        }
    },
    methods: {
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
        addToCart(product) {
            const existingProduct = this.cart.find((item) => item.id === product.id);
            if (existingProduct) {
                existingProduct.quantity += 1;
            } else {
                this.cart.push({ ...product, quantity: 1 });
            }
        },
        removeFromCart(productId) {
            this.cart = this.cart.filter((item) => item.id !== productId);
        },
        async checkout() {
            if (this.cart.length === 0) {
                alert('Your cart is empty! Please add some items to proceed.');
                return; // Stop the checkout process if the cart is empty
            }

            this.loading = true;

            try {
                // Define the order data
                const order = {
                    userId: 'user123', // Replace with the actual user ID if available
                    cartItems: this.cart.map((item) => ({
                        id: item.id,
                        productName: item.productName, // Use `productName` instead of `name`
                        price: item.price,
                        quantity: item.quantity,
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

                this.$nextTick(() => this.generateQRCode(qrData)); // Generate QR after rendering

                // Clear the cart
                this.cart = [];
            } catch (error) {
                console.error('Error during checkout:', error);
            } finally {
                this.loading = false;
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
    computed: {
        cartTotal() {
            return this.cart.reduce((total, item) => total + item.price * item.quantity, 0);
        },
    },
};
</script>

<style scoped>
/* Add your styles here */
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

.cart-table th,
.cart-table td {
    text-align: inherit;
    /* Ensures header and data cells share alignment */
    padding: 8px;
    /* Adds consistent spacing */
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
</style>