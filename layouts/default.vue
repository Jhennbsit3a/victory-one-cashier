<template>
  <v-app>
    <!-- Navigation Bar -->
    <v-container v-if="showDrawer" fluid class="nav-bar" style="background-color: #f4f4f4; padding: 10px; text-align: center;">
      <v-btn
        v-for="(item, i) in filteredItems"
        :key="i"
        :to="item.to"
        router
        text
        class="nav-btn"
        style="margin: 0 10px; background-color: #333; color: white;"
      >
        <v-icon left>{{ item.icon }}</v-icon>
        {{ item.title }}
      </v-btn>
      <v-btn color="error" dark @click="showLogoutDialog = true" style="float: right; margin-left: auto;">
        <v-icon left>mdi-logout</v-icon>Logout
      </v-btn>
    </v-container>

    <!-- Main Content -->
    <v-main>
      <v-container>
        <Nuxt />
      </v-container>
    </v-main>

    <!-- Logout Confirmation Dialog -->
    <v-dialog v-model="showLogoutDialog" max-width="400px">
      <v-card>
        <v-card-title class="headline">Confirm Logout</v-card-title>
        <v-card-text>Are you sure you want to log out?</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="red" text @click="confirmLogout">Yes</v-btn>
          <v-btn color="grey" text @click="showLogoutDialog = false">Cancel</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-app>
</template>

<script>
import { auth, firestore } from '~/plugins/firebase';
import { onAuthStateChanged } from 'firebase/auth';
import { doc, getDoc } from 'firebase/firestore';

export default {
  name: 'DefaultLayout',
  data() {
    return {
      showDrawer: false,
      showLogoutDialog: false,
      userRole: null,
      items: [
        { icon: 'mdi-credit-card', title: 'Cashier', to: '/cashier/payment_method' },
        { icon: 'mdi-view-dashboard', title: 'Walk-in Order', to: '/cashier/walkin_order' },
        { icon: 'mdi-account-group', title: 'Orders', to: '/cashier/orders' },
        { icon: 'mdi-swap-horizontal', title: 'Transaction', to: '/cashier/transaction_history' },
      ],
      showDrawerOnRoutes: [
        '/cashier/orders',
        '/cashier/transaction_history',
        '/cashier/payment_method',
        '/cashier/walkin_order',
      ]
    };
  },
  watch: {
    $route(to) {
      this.checkDrawerVisibility(to.path);
    }
  },
  mounted() {
    this.fetchUserRole(); // Fetch user role on mount
  },
  methods: {
    async fetchUserRole() {
      onAuthStateChanged(auth, async (user) => {
        if (user) {
          const userDocRef = doc(firestore, 'Users', user.uid);
          const userDoc = await getDoc(userDocRef);
          if (userDoc.exists()) {
            this.userRole = userDoc.data().role;
          } else {
            console.error('User role not found in Firestore');
          }
        }
      });
    },
    checkDrawerVisibility(path) {
      this.showDrawer = this.showDrawerOnRoutes.includes(path);
    },
    async confirmLogout() {
      try {
        await auth.signOut();
        console.log('User has successfully logged out.');
        this.$router.replace('/');
      } catch (error) {
        console.error('Error logging out:', error);
      } finally {
        this.showLogoutDialog = false;
      }
    },
  },
  computed: {
    filteredItems() {
      // Filter items based on user role
      return this.items.filter((item) => !item.roles || item.roles.includes(this.userRole));
    },
  },
};
</script>

<style scoped>
.nav-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.nav-btn {
  border-radius: 8px;
  text-transform: uppercase;
}

.v-main {
  margin-top: 20px;
  padding: 20px;
}

.v-dialog .v-card {
  border-radius: 12px;
}
</style>
