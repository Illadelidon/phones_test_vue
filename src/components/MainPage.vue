<template>
  <div class="container mt-4">
    <div>
      <Form :validation-schema="validationSchema" :validate-on-mount="false" :validateOnChange="true">
        <div class="form-group">
          <label for="first_name">First Name</label>
          <Field type="text" name="first_name" class="form-control" placeholder="First Name" v-model="form.first_name" />
          <ErrorMessage name="first_name" class="text-danger" />
        </div>

        <div class="form-group">
          <label for="last_name">Last Name</label>
          <Field id="last_name" type="text" name="last_name" class="form-control" placeholder="Last Name" v-model="form.last_name" />
          <ErrorMessage name="last_name" class="text-danger" />
        </div>

        <div v-for="(phone, index) in form.phone_numbers" :key="index" class="form-group">
          <input
              v-model="form.phone_numbers[index]"
              @input="checkPhoneExists(index, $event.target.value)"
              class="form-control"
              placeholder="Phone Number"
          />
          <ErrorMessage name="phone_numbers" class="text-danger" />
          <button type="button" class="btn btn-secondary mt-2" @click="addPhone" v-if="index === form.phone_numbers.length - 1">+</button>
          <button type="button" class="btn btn-danger mt-2" @click="removePhone(index)" v-if="form.phone_numbers.length > 1">Remove</button>
          <p v-if="phoneErrors[index]" class="text-danger">This phone number already exists in the database.</p>
        </div>

        <div>
          <button class="btn btn-primary" v-if="!form.id" @click.prevent="submitForm" :disabled="hasPhoneErrors">Submit</button>
          <button class="btn btn-warning" v-else @click.prevent="updateContact" :disabled="hasPhoneErrors">Update Contact</button>
        </div>
      </Form>
    </div>

    <div class="contacts-table mt-4">
      <h1>Contacts</h1>
      <table class="table table-bordered table-responsive" v-if="contacts.length > 0">
        <thead class="thead-light">
        <tr>
          <th>First Name</th>
          <th>Last Name</th>
          <th>Phone Numbers</th>
          <th>Actions</th>
        </tr>
        </thead>
        <tbody>
        <tr v-for="contact in contacts" :key="contact.id">
          <td>{{ contact.first_name || 'N/A' }}</td>
          <td>{{ contact.last_name || 'N/A' }}</td>
          <td>
            <ul>
              <li v-for="phone in contact.phone_numbers" :key="phone.id" class="phone-list">
                {{ phone.phone_number || 'N/A' }}
              </li>
            </ul>
          </td>
          <td>
            <button class="btn btn-info" @click="editContact(contact)">Edit</button>
            <button class="btn btn-danger" @click="deleteContact(contact.id)">Delete</button>
          </td>
        </tr>
        </tbody>
      </table>
      <p v-else>No contacts available</p>

      <div class="pagination" v-if="pagination.total > pagination.per_page">
        <button class="btn btn-secondary" @click="fetchContacts(pagination.current_page - 1)" :disabled="!pagination.prev_page_url">
          &laquo; Previous
        </button>
        <span>Page {{ pagination.current_page }} of {{ pagination.last_page }}</span>
        <button class="btn btn-secondary" @click="fetchContacts(pagination.current_page + 1)" :disabled="!pagination.next_page_url">
          Next &raquo;
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { toTypedSchema } from "@vee-validate/zod";
import { ErrorMessage, Field, Form } from "vee-validate";
import { z } from "zod";

export default {
  name: 'MainPage',
  components: { ErrorMessage, Field, Form },
  data() {
    return {
      contacts: [],
      pagination: {
        current_page: 1,
        last_page: 1,
        total: 0,
        per_page: 5,
        prev_page_url: null,
        next_page_url: null,
      },
      form: {
        id: null,
        first_name: '',
        last_name: '',
        phone_numbers: ['']
      },
      phoneErrors: [],
      hasPhoneErrors: false
    };
  },
  setup() {
    const validationSchema = toTypedSchema(z.object({
      first_name: z.string().min(2, "This is required"),
      last_name: z.string().min(2, "This is required"),

    }));

    return { validationSchema };
  },
  created() {
    this.fetchContacts();
  },
  methods: {
    async fetchContacts(page = 1) {
      try {
        const response = await fetch(`http://localhost:8000/api/all-contacts?page=${page}`);
        const data = await response.json();
        if (data.data && Array.isArray(data.data)) {
          this.contacts = data.data;
          this.pagination = {
            current_page: data.current_page,
            last_page: data.last_page,
            total: data.total,
            per_page: data.per_page,
            prev_page_url: data.prev_page_url,
            next_page_url: data.next_page_url
          };
        } else {
          console.error('Unexpected data structure:', data);
        }
      } catch (error) {
        console.error('Error fetching contacts:', error);
      }
    },
    addPhone() {
      this.form.phone_numbers.push('');
    },
    removePhone(index) {
      if (this.form.phone_numbers.length > 1) {
        this.form.phone_numbers.splice(index, 1);
      }
    },
    async checkPhoneExists(index) {
      if (!this.form.phone_numbers[index]) return;

      try {
        const response = await fetch(`http://localhost:8000/api/check-phone?phone=${this.form.phone_numbers[index]}`);
        const result = await response.json();


        this.phoneErrors[index] = result.exists;

        this.hasPhoneErrors = this.phoneErrors.includes(true);
      } catch (error) {
        console.error('Error checking phone:', error);
      }
    },
    async submitForm() {
      const url = 'http://localhost:8000/api/contacts';
      try {
        const response = await fetch(url, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            first_name: this.form.first_name,
            last_name: this.form.last_name,
            phone_number: this.form.phone_numbers
          })
        });

        if (response.ok) {
          const result = await response.json();
          console.log('Form submitted:', result);
          this.resetForm();

          window.location.reload();
          //await this.fetchContacts();
        } else {
          const errorDetails = await response.json();
          console.error('Error response:', errorDetails);
        }
      } catch (error) {
        console.error('Error submitting form:', error);
      }
    },
    async updateContact() {
      const url = `http://localhost:8000/api/contacts/${this.form.id}`;
      try {
        const response = await fetch(url, {
          method: 'PUT',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            first_name: this.form.first_name,
            last_name: this.form.last_name,
            phone_number: [...this.form.phone_numbers],
          }),
        });

        if (response.ok) {
          console.log('Contact updated successfully');
          await this.fetchContacts();
          this.resetForm();

          window.location.reload();
        } else {
          const errorDetails = await response.json();
          console.error('Error updating contact:', errorDetails);
        }
      } catch (error) {
        console.error('Error:', error);
      }
    },
    async deleteContact(id) {
      try {
        const response = await fetch(`http://localhost:8000/api/contacts/${id}`, {
          method: 'DELETE'
        });
        if (response.ok) {
          console.log('Contact deleted');
          await this.fetchContacts();
        } else {
          const errorDetails = await response.json();
          console.error('Error deleting contact:', errorDetails);
        }
      } catch (error) {
        console.error('Error deleting contact:', error);
      }
    },
    editContact(contact) {
      this.form.id = contact.id;
      this.form.first_name = contact.first_name;
      this.form.last_name = contact.last_name;
      this.form.phone_numbers = contact.phone_numbers.map(p => p.phone_number);
      console.log('Contact to edit:', this.form);
    },
    resetForm() {
      this.form = {
        id: null,
        first_name: '',
        last_name: '',
        phone_numbers: ['']
      };
      this.phoneErrors = [];
      this.hasPhoneErrors = false;
    },
  }
};
</script>

<style scoped>
.contacts-table {
  margin-top: 20px;
}

.pagination {
  margin-top: 20px;
}

button {
  margin: 5px;
}

.error {
  color: red;
  font-size: 12px;
}
.phone-list {

  white-space: nowrap;
}
@media (max-width: 576px) {
  .phone-list {
    font-size: 0.9rem;
  }
}
</style>
