<template>
    <v-container>
      <v-row>
        <!-- Payment Methods Table -->
        <v-col cols="12" class="mb-4">
          <v-card class="elevation-2">
            <v-card-title class="d-flex justify-between align-center">
              <div class="headline">
                <v-icon class="mr-2">mdi-credit-card</v-icon>
                Payment Methods
              </div>
              <v-spacer></v-spacer>
              <v-btn color="primary" icon @click="openDialog('Payment')">
                <v-icon>mdi-plus</v-icon>
              </v-btn>
            </v-card-title>
            <v-data-table :headers="paymentHeaders" :items="payments" item-key="id" class="elevation-1"
              :header-class="{ 'text-align-center': true }">
              <template v-slot:item.actions="{ item }">
                <v-btn @click="setSelectedPayment(item)" color="blue" icon>
                  <v-icon>mdi-pencil</v-icon>
                </v-btn>
                <v-btn @click="confirmDeletePayment(item.id)" color="red" icon>
                  <v-icon>mdi-delete</v-icon>
                </v-btn>
              </template>
            </v-data-table>
          </v-card>
        </v-col>
  
        <!-- Delete Confirmation Dialog -->
        <v-dialog v-model="deleteDialog" max-width="400">
          <v-card>
            <v-card-title class="headline">Confirm Deletion</v-card-title>
            <v-card-text>
              Are you sure you want to delete this payment method?
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="red" @click="deletePayment">Delete</v-btn>
              <v-btn color="grey" @click="deleteDialog = false">Cancel</v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
  
        <!-- Logs Table -->
        <v-col cols="12" class="mt-4">
          <v-card class="elevation-2">
            <v-card-title class="d-flex align-center">
              <v-icon class="mr-2">mdi-history</v-icon>
              Logs
            </v-card-title>
            <v-data-table :headers="logHeaders" :items="logs" item-key="id" class="elevation-1"
              :header-class="{ 'text-align-center': true }">
              <template v-slot:item.timestamp="{ item }">
                {{ formatDate(item.timestamp) }}
              </template>
            </v-data-table>
          </v-card>
        </v-col>
      </v-row>
  
      <!-- Add Item Dialog -->
      <v-dialog v-model="dialog.visible" max-width="500">
        <v-card>
          <v-card-title class="headline">{{ dialog.title }}</v-card-title>
          <v-divider></v-divider>
          <v-form class="pa-4">
            <v-text-field v-model="dialog.data.method" label="Payment Method" outlined dense required />            
          </v-form>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn text color="green" @click="saveDialogData">Save</v-btn>
            <v-btn text color="grey" @click="closeDialog">Cancel</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-container>
  </template>
  
  <script>
  import { firestore } from '~/plugins/firebase';
  import { collection, addDoc, deleteDoc, doc, updateDoc, getDocs, onSnapshot } from 'firebase/firestore';
  import { getAuth } from 'firebase/auth';
  
  export default {
    data() {
      return {
        payments: [],
        logs: [],
        paymentHeaders: [
          { text: 'Payment Method', value: 'method' },          
          { text: 'Actions', value: 'actions', sortable: false }
        ],
        logHeaders: [
          { text: 'Action', value: 'action' },
          { text: 'Payment Method', value: 'method' },          
          { text: 'Timestamp', value: 'timestamp' }
        ],
        paymentMethod: { method: '', description: '' },
        selectedPayment: null,
        deleteDialog: false,
        paymentToDelete: null,
        dialog: {
          visible: false,
          title: '',
          data: { method: '', description: '' },
          type: '' // Identifies which table is being updated
        }
      };
    },
    created() {
      this.loadPayments();
      this.loadLogs(); // Fetch logs
    },
    methods: {
      // Fetch user ID for authenticated actions
      getUserId() {
        const user = getAuth().currentUser;
        return user ? user.uid : null;
      },
      async loadLogs() {
        const logsCollection = collection(firestore, 'Logs');
        onSnapshot(logsCollection, (snapshot) => {
          this.logs = snapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
        });
      },
      // Format timestamps for display
      formatDate(timestamp) {
        if (timestamp && timestamp.toDate) {
          const date = timestamp.toDate();
          return date.toLocaleString();
        }
        return '';
      },
      async logAction(action, payment) {
        const userId = this.getUserId();
        const logEntry = {
          action, // 'Add', 'Edit', or 'Delete'
          method: payment.method,
          description: payment.description || '',
          timestamp: new Date(),
          userID: userId
        };
        await addDoc(collection(firestore, 'Logs'), logEntry);
      },
      // Load payments from Firebase
      async loadPayments() {
        const querySnapshot = await getDocs(collection(firestore, 'Payments'));
        this.payments = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      },
      // Set selected payment for editing
      setSelectedPayment(payment) {
  this.dialog.visible = true; // Open the dialog
  this.dialog.title = "Edit Payment Method"; // Set the title
  this.dialog.data = { ...payment }; // Populate the dialog with the selected payment data
},  
      // Clear the form
      clearForm() {
        this.paymentMethod = { method: '', description: '' };
        this.selectedPayment = null;
      },
  
      // Add or update a payment
      async handleSubmit() {
        if (this.selectedPayment) {
          await this.updatePayment(this.selectedPayment.id);
        } else {
          await this.addPayment();
        }
        this.dialog.visible = false; // Close the dialog
        this.clearForm();
        this.loadPayments();
      },
  
      // Add a new payment to Firebase
      async addPayment() {
        const newPayment = {
          method: this.paymentMethod.method,
          description: this.paymentMethod.description || '',
          createdAt: new Date(),  // No userID field here
        };
        const docRef = await addDoc(collection(firestore, 'Payments'), newPayment);
        this.logAction('Add', { ...newPayment, id: docRef.id });
      },
      // Add log when editing a payment
      async updatePayment(paymentId) {
        const userId = this.getUserId();
        const paymentRef = doc(firestore, 'Payments', paymentId);
        const updatedPayment = {
          method: this.paymentMethod.method,
          description: this.paymentMethod.description || ''
        };
        await updateDoc(paymentRef, updatedPayment);
        this.logAction('Edit', { id: paymentId, ...updatedPayment });
      },
  
      // Prompt delete confirmation
      confirmDeletePayment(paymentId) {
        this.paymentToDelete = paymentId;
        this.deleteDialog = true;
      },
  
      // Add log when deleting a payment
      async deletePayment() {
        if (this.paymentToDelete) {
          const payment = this.payments.find((p) => p.id === this.paymentToDelete);
          await deleteDoc(doc(firestore, 'Payments', this.paymentToDelete));
          this.logAction('Delete', payment);
          this.payments = this.payments.filter((p) => p.id !== this.paymentToDelete);
          this.paymentToDelete = null;
        }
        this.deleteDialog = false;
      },
  
      // Open the dialog for adding data
      openDialog(type) {
        this.dialog.visible = true;
        this.dialog.title = `Add New ${type}`;
        this.dialog.type = type;
        this.dialog.data = { method: '', description: '' };
      },
  
      // Close the dialog
      closeDialog() {
        this.dialog.visible = false;
      },
  
      // Save new data from the dialog
      async saveDialogData() {
  const paymentData = this.dialog.data;
  
  if (paymentData.id) {
    // Update existing payment
    const paymentRef = doc(firestore, 'Payments', paymentData.id);
    const updatedPayment = {
      method: paymentData.method,
      description: paymentData.description || '',
    };
    await updateDoc(paymentRef, updatedPayment);
    this.logAction('Edit', { id: paymentData.id, ...updatedPayment });
  } else {
    // Add new payment
    const newPayment = {
      method: paymentData.method,
      description: paymentData.description || '',
      createdAt: new Date(),
    };
    const docRef = await addDoc(collection(firestore, 'Payments'), newPayment);
    this.logAction('Add', { ...newPayment, id: docRef.id });
  }

  this.closeDialog(); // Close the dialog
  this.loadPayments(); // Reload payments to refresh the table
},
    }
  };
  </script>
  
  <style scoped>
  .v-data-table {
    background-color: white;
  }
  
  .v-btn {
    margin: 5px;
  }
  </style>
  