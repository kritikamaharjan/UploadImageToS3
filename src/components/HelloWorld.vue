
<template>
  <div id="app">
    <h1>Employee Registration Application</h1>

    <!-- Employee details -->
    <div class="employee-form">
      <h2>Employee Details</h2>

      <input
        v-model="employee.name"
        type="text"
        placeholder="Employee Name"
        required
      />

      <input
        v-model="employee.phoneNumber"
        type="tel"
        placeholder="Phone Number"
        required
      />

      <input
        v-model="employee.email"
        type="email"
        placeholder="Email Address"
        required
      />

      <input
        v-model="employee.employeeCode"
        type="text"
        placeholder="Employee Code"
        required
      />
    </div>

    <!-- Image selection -->
    <div v-if="!image">
      <h2>Upload ID Card</h2>
      <input type="file" accept="image/jpeg" @change="onFileChange">
    </div>

    <!-- Image preview -->
    <div v-else>
      <img :src="image" />

      <button
        v-if="!imageUploaded"
        @click="removeImage"
      >
        Remove image
      </button>

      <button
        v-if="!imageUploaded"
        @click="uploadImage"
        :disabled="isUploading"
      >
        {{ isUploading ? "Uploading..." : "Upload ID Card" }}
      </button>
    </div>

    <p v-if="imageUploaded">
      ID card uploaded successfully to S3!
    </p>

    <!-- Register employee -->
    <button
      v-if="imageUploaded"
      @click="registerEmployee"
      :disabled="isSubmitting"
    >
      {{ isSubmitting ? "Registering..." : "Register Employee" }}
    </button>

    <p v-if="successMessage" class="success">
      {{ successMessage }}
    </p>

    <p v-if="errorMessage" class="error">
      {{ errorMessage }}
    </p>
  </div>
</template>

<script>
import axios from "axios";

const MAX_IMAGE_SIZE = 1000000;

// Existing API: Generates S3 presigned URL
const API_ENDPOINT =
  'https://74ie1wykx4.execute-api.us-east-1.amazonaws.com/default/lambda1';

// New API: Saves employee details in DynamoDB
const EMPLOYEE_API_ENDPOINT =
  'https://24vukuy57c.execute-api.us-east-1.amazonaws.com/prod/employees';

export default {
  name: 'app',

  data() {
    return {
      image: '',
      imageUploaded: false,
      photoFilename: '',

      isUploading: false,
      isSubmitting: false,

      successMessage: '',
      errorMessage: '',

      employee: {
        name: '',
        phoneNumber: '',
        email: '',
        employeeCode: ''
      }
    };
  },

  methods: {

    // Select an image
    onFileChange(e) {
      let files = e.target.files || e.dataTransfer.files;

      if (!files.length) return;

      this.createImage(files[0]);
    },

    // Preview and validate image
    createImage(file) {
      if (file.type !== 'image/jpeg') {
        return alert('JPG images only.');
      }

      if (file.size > MAX_IMAGE_SIZE) {
        return alert('Image must be less than 1 MB.');
      }

      let reader = new FileReader();

      reader.onload = (e) => {
        this.image = e.target.result;
        this.imageUploaded = false;
        this.photoFilename = '';
        this.successMessage = '';
        this.errorMessage = '';
      };

      reader.readAsDataURL(file);
    },

    // Remove selected image
    removeImage() {
      this.image = '';
      this.imageUploaded = false;
      this.photoFilename = '';
    },

    // Upload image to S3
    async uploadImage() {
      try {
        this.isUploading = true;
        this.errorMessage = '';

        // Step 1: Get presigned URL from existing Lambda
        const response = await axios({
          method: 'GET',
          url: API_ENDPOINT
        });

        const { uploadURL, photoFilename } = response.data;

        if (!uploadURL || !photoFilename) {
          throw new Error('Could not get S3 upload URL.');
        }

        // Step 2: Convert image to binary
        let binary = atob(this.image.split(',')[1]);
        let array = [];

        for (let i = 0; i < binary.length; i++) {
          array.push(binary.charCodeAt(i));
        }

        let blobData = new Blob(
          [new Uint8Array(array)],
          { type: 'image/jpeg' }
        );

        // Step 3: Upload image directly to S3
        const result = await fetch(uploadURL, {
          method: 'PUT',
          body: blobData
        });

        if (!result.ok) {
          throw new Error('S3 image upload failed.');
        }

        // Store the S3 filename for DynamoDB
        this.photoFilename = photoFilename;
        this.imageUploaded = true;

        this.successMessage = 'Image uploaded successfully!';

      } catch (error) {
        console.error('Image upload error:', error);
        this.errorMessage =
          error.message || 'Image upload failed.';

      } finally {
        this.isUploading = false;
      }
    },

    // Save employee details in DynamoDB
    async registerEmployee() {
      try {
        this.isSubmitting = true;
        this.errorMessage = '';
        this.successMessage = '';

        // Validate employee details
        if (
          !this.employee.name.trim() ||
          !this.employee.phoneNumber.trim() ||
          !this.employee.email.trim() ||
          !this.employee.employeeCode.trim()
        ) {
          throw new Error('Please complete all employee details.');
        }

        if (!this.imageUploaded || !this.photoFilename) {
          throw new Error('Please upload the ID card first.');
        }

        // Prepare employee information
        const employeeData = {
          name: this.employee.name.trim(),
          phoneNumber: this.employee.phoneNumber.trim(),
          email: this.employee.email.trim(),
          employeeCode: this.employee.employeeCode.trim(),
          photoFilename: this.photoFilename
        };

        // Send details to API Gateway
        const response = await axios.post(
          EMPLOYEE_API_ENDPOINT,
          employeeData,
          {
            headers: {
              'Content-Type': 'application/json'
            }
          }
        );

        this.successMessage =
          response.data.message;

        console.log('Registration response:', response.data);

      } catch (error) {
        console.error('Registration error:', error);

        this.errorMessage =
          error.response?.data?.message ||
          error.message ||
          'Employee registration failed.';

      } finally {
        this.isSubmitting = false;
      }
    }
  }
};
</script>

<style scoped>
body {
  background: #20262E;
  padding: 20px;
  font-family: sans-serif;
}

#app {
  background: #fff;
  border-radius: 4px;
  padding: 20px;
  text-align: center;
}

.employee-form {
  margin: 20px auto;
}

input {
  display: block;
  width: 80%;
  max-width: 400px;
  padding: 10px;
  margin: 10px auto;
}

button {
  padding: 10px 20px;
  margin: 10px;
  cursor: pointer;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

img {
  width: 30%;
  margin: auto;
  display: block;
  margin-bottom: 10px;
}

.success {
  color: green;
}

.error {
  color: red;
}
</style>
