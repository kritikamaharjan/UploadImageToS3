
<template>
  <div id="app">
    <div class="registration-card">
      <h1>Employee Registration</h1>

      <form @submit.prevent="registerEmployee">

        <div class="form-group">
          <label>Employee Code</label>
          <input
            type="text"
            :value="employee.employeeCode"
            readonly
            placeholder="Generated during registration"
          />
          <small>Automatically generated</small>
        </div>

        <div class="form-group">
          <label>Name</label>
          <input
            v-model.trim="employee.name"
            type="text"
            placeholder="Enter employee name"
            required
          />
        </div>

        <div class="form-group">
          <label>Phone Number</label>
          <input
            v-model.trim="employee.phoneNumber"
            type="tel"
            placeholder="Enter phone number"
            required
          />
        </div>

        <div class="form-group">
          <label>Email ID</label>
          <input
            v-model.trim="employee.email"
            type="email"
            placeholder="Enter email address"
            required
          />
        </div>

        <div class="form-group">
          <label>ID Card Photo</label>
          <input
            type="file"
            accept="image/jpeg"
            @change="onFileChange"
            required
          />

          <img
            v-if="image"
            :src="image"
            class="preview"
            alt="ID card preview"
          />
        </div>

        <button
          type="submit"
          :disabled="isSubmitting || !selectedFile"
        >
          {{ isSubmitting ? "Registering..." : "Register Employee" }}
        </button>

        <p v-if="successMessage" class="success">
          {{ successMessage }}
        </p>

        <p v-if="errorMessage" class="error">
          {{ errorMessage }}
        </p>

      </form>
    </div>
  </div>
</template>

<script>
import axios from "axios";

const MAX_IMAGE_SIZE = 1000000;

const API_ENDPOINT =
  "https://74ie1wykx4.execute-api.us-east-1.amazonaws.com/default/lambda1";

const EMPLOYEE_API_ENDPOINT =
  "https://24vukuy57c.execute-api.us-east-1.amazonaws.com/prod/employees";

export default {
  name: "HelloWorld",

  data() {
    return {
      image: "",
      selectedFile: null,
      isSubmitting: false,
      successMessage: "",
      errorMessage: "",

      employee: {
        employeeCode: "",
        name: "",
        phoneNumber: "",
        email: ""
      }
    };
  },

  methods: {
    onFileChange(event) {
      const file = event.target.files[0];

      this.image = "";
      this.selectedFile = null;
      this.errorMessage = "";
      this.successMessage = "";

      if (!file) return;

      if (file.type !== "image/jpeg") {
        this.errorMessage = "Please select a JPG image.";
        event.target.value = "";
        return;
      }

      if (file.size > MAX_IMAGE_SIZE) {
        this.errorMessage = "Image must be less than 1 MB.";
        event.target.value = "";
        return;
      }

      this.selectedFile = file;
      this.image = URL.createObjectURL(file);
    },

    async registerEmployee() {
      if (this.isSubmitting) return;

      this.isSubmitting = true;
      this.errorMessage = "";
      this.successMessage = "";

      try {
        if (
          !this.employee.name ||
          !this.employee.phoneNumber ||
          !this.employee.email
        ) {
          throw new Error("Please complete all employee details.");
        }

        if (!this.selectedFile) {
          throw new Error("Please select an ID card photo.");
        }

        // Step 1: Request S3 presigned URL
        const urlResponse = await axios.get(API_ENDPOINT);

        const { uploadURL, photoFilename } = urlResponse.data;

        if (!uploadURL || !photoFilename) {
          throw new Error("Could not generate S3 upload URL.");
        }

        // Step 2: Upload image directly to S3
        const uploadResponse = await fetch(uploadURL, {
          method: "PUT",
          body: this.selectedFile,
          headers: {
            "Content-Type": "image/jpeg"
          }
        });

        if (!uploadResponse.ok) {
          throw new Error("Could not upload ID card to S3.");
        }

        // Step 3: Save employee details
        const employeeData = {
          name: this.employee.name,
          phoneNumber: this.employee.phoneNumber,
          email: this.employee.email,
          photoFilename: photoFilename
        };

        const response = await axios.post(
          EMPLOYEE_API_ENDPOINT,
          employeeData,
          {
            headers: {
              "Content-Type": "application/json"
            }
          }
        );

        // Step 4: Display generated employee code
        this.employee.employeeCode = response.data.employeeCode;

        this.successMessage =
          `Employee registered successfully! Employee Code: ${response.data.employeeCode}`;

        // Clear form fields after successful registration
        this.employee.name = "";
        this.employee.phoneNumber = "";
        this.employee.email = "";
        this.selectedFile = null;
        this.image = "";

      } catch (error) {
        console.error("Registration error:", error);

        this.errorMessage =
          error.response?.data?.message ||
          error.message ||
          "Employee registration failed.";

      } finally {
        this.isSubmitting = false;
      }
    }
  },

  beforeUnmount() {
    if (this.image) {
      URL.revokeObjectURL(this.image);
    }
  }
};
</script>

<style scoped>
#app {
  min-height: 100vh;
  background: #f4f6f9;
  padding: 40px 15px;
  font-family: Arial, sans-serif;
}

.registration-card {
  max-width: 480px;
  margin: 0 auto;
  padding: 30px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.08);
}

h1 {
  text-align: center;
  font-size: 25px;
  margin-bottom: 25px;
  color: #263b55;
}

.form-group {
  margin-bottom: 18px;
}

label {
  display: block;
  margin-bottom: 7px;
  font-weight: 600;
  color: #333;
}

input {
  box-sizing: border-box;
  width: 100%;
  padding: 11px;
  border: 1px solid #ccc;
  border-radius: 5px;
  font-size: 14px;
}

input[readonly] {
  background: #f0f2f5;
  color: #555;
}

small {
  display: block;
  margin-top: 5px;
  color: #777;
}

.preview {
  display: block;
  max-width: 180px;
  max-height: 180px;
  margin-top: 12px;
  border-radius: 5px;
  object-fit: contain;
}

button {
  width: 100%;
  padding: 13px;
  border: none;
  border-radius: 5px;
  background: #263b55;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.success {
  color: #16803c;
  text-align: center;
  margin-top: 18px;
}

.error {
  color: #c62828;
  text-align: center;
  margin-top: 18px;
}

@media (max-width: 480px) {
  .registration-card {
    padding: 20px;
  }
}
</style>
