<template>
  <v-app>
    <!-- App Bar -->
    <v-app-bar app dense color="primary" dark>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer"></v-app-bar-nav-icon>
      <v-toolbar-title>Cashier POS</v-toolbar-title>
    </v-app-bar>

    <!-- Navigation Drawer -->
    <v-navigation-drawer
      v-model="drawer"
      app
      :mini-variant="miniVariant"
      permanent
      class="drawer"
    >
      <!-- Drawer Header -->
      <v-list-item class="drawer-header">
          <v-img    
          :src="require('@/assets/logo 3.png')"
          alt="Company logo"
          max-height="400"
          max-width="400"></v-img>
      </v-list-item>
      <v-divider></v-divider>

      <!-- Navigation Items -->
      <v-list dense>
        <v-list-item
          v-for="(item, i) in filteredItems"
          :key="i"
          :to="item.to"
          router
          exact
          active-class="active-link"
        >
          <v-list-item-icon>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-icon>
          <v-list-item-title>{{ item.title }}</v-list-item-title>
        </v-list-item>
      </v-list>

      <v-spacer></v-spacer>

      <!-- Logout Button -->
      <v-list>
        <v-list-item>
          <v-btn block color="error" dark @click="showLogoutDialog = true">
            <v-icon left>mdi-logout</v-icon>
            Logout
          </v-btn>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>

    <!-- Main Content -->
    <v-main>
      <v-container class="main-container">
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
      showLogoutDialog: false,
      clipped: false,
      drawer: true,
      miniVariant: false,
      userRole: null, // Add a property to store the user role
      items: [
        { icon: 'mdi-credit-card', title: 'Cashier', to: '/cashier/payment_method' },
        { icon: 'mdi-view-dashboard', title: 'Walk-in Order', to: '/cashier/walkin_order' },
        { icon: 'mdi-account-group', title: 'Orders', to: '/cashier/orders' },
        { icon: 'mdi-swap-horizontal', title: 'Transaction', to: '/cashier/transaction_history' },
      ],
    };
  },
  watch: {
    $route(to) {
      this.checkDrawerVisibility(to.path);
    },
  },
  mounted() {
    this.fetchUserRole(); // Fetch user role on mount
    this.checkDrawerVisibility(this.$route.path);
  },
  methods: {
    async confirmLogout() {
      // Perform logout and close the dialog
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
    async fetchUserRole() {
      onAuthStateChanged(auth, async (user) => {
        if (user) {
          const userDocRef = doc(firestore, 'Users', user.uid);
          const userDoc = await getDoc(userDocRef);
          if (userDoc.exists()) {
            this.userRole = userDoc.data().role;
            console.log(`User role fetched: ${this.userRole}`); // Log the role for confirmation
          } else {
            console.error('User role not found in Firestore');
          }
        }
      });
    },
    checkDrawerVisibility(path) {
      // Adjust visibility based on current route (logic retained)
    },
    // async logout() {
    //   try {
    //     await auth.signOut();
    //     console.log('User has successfully logged out.');
    //     this.$router.replace('/');
    //   } catch (error) {
    //     console.error('Error logging out:', error);
    //   }
    // },
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
.drawer-header {
  padding: 14px;
  text-align: center;
}

.drawer-header .v-list-item-avatar {
  margin: 0 auto;
}

.active-link {
  background-color: rgba(0, 123, 255, 0.2);
  border-radius: 8px;
}

.main-container {
  background-color: #f9f9f9;
  padding: 16px;
  min-height: 100vh;
}

.logout-button {
  margin-top: 16px;
}

.v-navigation-drawer {
  background: #ffffff;
  box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
}
</style>
