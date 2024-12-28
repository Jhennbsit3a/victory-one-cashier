<template>
  <v-container class="d-flex align-center justify-center" style="height: 100vh;">
    <v-card class="elevation-3" max-width="600">
      <v-card-title>
        <h3>Customer Details</h3>
      </v-card-title>
      <v-card-text>
        <!-- Instruction -->
        <p class="text-center mb-4">
          Please fill out the form below to proceed with ordering products.
        </p>

        <!-- Customer Name Input -->
        <v-text-field 
          v-model="customerFirstName" 
          label="Customer First Name" 
          outlined 
          dense 
          required
        ></v-text-field>
        <v-text-field 
          v-model="customerLastName" 
          label="Customer Last Name" 
          outlined 
          dense 
          required
        ></v-text-field>

        <!-- Customer Address Input -->
        <v-text-field 
          v-model="customerAddress" 
          label="Customer Address" 
          outlined 
          dense 
          required
        ></v-text-field>

        <!-- Proceed Button -->
        <v-btn 
          color="primary" 
          block 
          class="mt-4" 
          @click="saveCustomerDetails">
          Proceed
        </v-btn>
      </v-card-text>
    </v-card>
  </v-container>
</template>

<script>
export default {
  data() {
    return {
      customerFirstName: '',
      customerLastName: '',
      customerAddress: ''
    };
  },
  methods: {
    saveCustomerDetails() {
      if (!this.customerFirstName || !this.customerAddress || !this.customerLastName) {
        alert('Please fill in all fields.');
        return;
      }
      // Save details to sessionStorage
      sessionStorage.setItem('customerFirstName', this.customerFirstName);
      sessionStorage.setItem('customerLastName', this.customerLastName);
      sessionStorage.setItem('customerAddress', this.customerAddress);

      // Navigate to the next route
      this.$router.push('/cashier/walkin_order');
    }
  }
};
</script>

<style scoped>
.v-container {
  max-width: 700px;
  margin: auto;
}
</style>
