<template>
  <v-container class="fill-height d-flex align-center justify-center" fluid>
    <v-row align="center" justify="center">
      <v-col cols="12" sm="10" md="6" lg="4">
        <v-card class="sign-in-card">
          <v-card-text class="text-center mt-4">
            <h2>Cashier Sign In</h2>
          </v-card-text>

          <v-card-text>
            <v-form ref="form" v-model="valid" lazy-validation>
              <!-- Email Input -->
              <v-text-field 
                v-model="email" 
                :rules="emailRules" 
                label="Email" 
                required 
                class="input-field mt-2"
              >
                <template v-slot:prepend>
                  <v-icon>mdi-email</v-icon>
                </template>
              </v-text-field>

              <!-- Password Input -->
              <v-text-field 
                v-model="password" 
                :rules="passwordRules" 
                label="Password" 
                :type="showPassword ? 'text' : 'password'" 
                required 
                class="input-field mt-2"
              >
                <template v-slot:prepend>
                  <v-icon>mdi-lock</v-icon>
                </template>
                <template v-slot:append>
                  <v-btn icon @click="togglePasswordVisibility">
                    <v-icon>{{ showPassword ? 'mdi-eye-off' : 'mdi-eye' }}</v-icon>
                  </v-btn>
                </template>
              </v-text-field>

              <!-- Sign In Button -->
              <v-btn 
                @click="signIn" 
                :disabled="loading" 
                class="primary-btn full-width"
                color="orange" 
                block
              >
                <v-icon v-if="loading" left>mdi-loading</v-icon>
                <span v-if="!loading">Sign In</span>
                <span v-else>Signing In...</span>
              </v-btn>
            </v-form>
          </v-card-text>

          <!-- <div class="text-center mt-1">
            <v-btn 
              text 
              @click="forgotPasswordDialog = true" 
              class="text-link"
            >
              Forgot Password?
            </v-btn>
          </div> -->
        </v-card>
      </v-col>
    </v-row>

    <!-- Forgot Password Dialog -->
    <v-dialog v-model="forgotPasswordDialog" max-width="400px">
      <v-card>
        <v-card-title class="headline">Reset Password</v-card-title>
        <v-card-text>
          <p>Please enter your email address to reset your password:</p>
          <v-text-field 
            v-model="resetEmail" 
            :rules="emailRules" 
            label="Email" 
            required
          >
            <template v-slot:prepend>
              <v-icon>mdi-email</v-icon>
            </template>
          </v-text-field>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn 
            color="orange darken-2" 
            dark 
            @click="sendResetEmail"
          >
            Send
          </v-btn>
          <v-btn text @click="forgotPasswordDialog = false">Cancel</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </v-container>
</template>

<script>
import { auth, firestore } from '~/plugins/firebase';
import { signInWithEmailAndPassword, sendPasswordResetEmail } from 'firebase/auth';
import { doc, getDoc, query, collection, where, getDocs } from 'firebase/firestore';

export default {
  data() {
    return {
      loading: false,
      valid: false,
      email: '',
      password: '',
      showPassword: false,
      forgotPasswordDialog: false,
      resetEmail: '',
      emailRules: [v => !!v || 'Email is required', v => /.+@.+\..+/.test(v) || 'E-mail must be valid'],
      passwordRules: [v => !!v || 'Password is required', v => v.length >= 6 || 'Password must be at least 6 characters'],
    };
  },
  methods: {
     async signIn() {
      if (this.$refs.form.validate()) {
        this.loading = true;
        try {
          // Query Firestore for a user with the provided email and password
          const usersQuery = query(
            collection(firestore, "Users"),
            where("email", "==", this.email),
            where("password", "==", this.password)
          );
          const usersSnapshot = await getDocs(usersQuery);

          if (!usersSnapshot.empty) {
            // Assuming only one user matches the email and password
            const userDoc = usersSnapshot.docs[0];
            const userData = userDoc.data();
            console.log(userData);

            const userRole = userData.role;
            if (["admin", "cashier", "owner"].includes(userRole)) {
              // console.log(`Role "${userRole}" is validated. Access granted.`);
              this.$router.push("/cashier/payment_method");
            } else {
              console.log(`Role "${userRole}" is not authorized. Redirecting to no-access page.`);
              this.$router.push("/no-access");
            }
          } else {
            // console.error("Invalid email or password");
            alert("Invalid email or password. Please try again.");
          }
        } catch (error) {
          console.error("Error during sign-in:", error);
          alert("Sign-in failed: " + error.message);
        } finally {
          this.loading = false;
        }
      }
    },
    async sendResetEmail() {
      if (this.resetEmail) {
        try {
          await sendPasswordResetEmail(auth, this.resetEmail);
          this.forgotPasswordDialog = false;
          alert('Password reset email sent!');
        } catch (error) {
          console.error('Error sending password reset email:', error);
          alert('Error sending password reset email: ' + error.message);
        }
      } else {
        alert('Please enter your email to reset your password.');
      }
    },
    togglePasswordVisibility() {
      this.showPassword = !this.showPassword;
    },
  },
};
</script>

<style scoped>
.fill-height {
  height: 100vh;
  background-color: white;
}

.sign-in-card {
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.primary-btn {
  font-weight: bold;
  border-radius: 24px;
  height: 48px;
  background-color: #000;
  color: white;
}

.input-field {
  margin-top: 10px;
}

.text-link {
  font-size: 14px;
  color: #007bff;
  text-decoration: underline;
  cursor: pointer;
}

.text-center h2 {
  font-size: 24px;
  color: black;
}

@media (max-width: 768px) {
  .sign-in-card {
    padding: 16px;
  }

  .text-center h2 {
    font-size: 20px;
  }
}
</style>
