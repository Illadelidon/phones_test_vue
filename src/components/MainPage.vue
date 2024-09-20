<template>
  <div>
    <div>
      <Form :validation-schema="validationSchema" :validate-on-mount="false" :validateOnChange="true">
      <!-- Поля для введення імені та прізвища -->
      <label for="first_name"></label>
<!--        <Field id="first_name" type="text" name="first_name"  />
        <ErrorMessage name="first_name" class="error"/>

        <label for="second_name"></label>
        <Field id="second_name" type="text" name="second_name"  />
        <ErrorMessage name="second_name" class="error"/>-->



        <Field id="first_name" type="text" name="first_name" placeholder="First Name" v-model="form.first_name"/>
        <ErrorMessage name="first_name" class="error" />

<!--      <input id="first_name" name="first_name" v-model="form.first_name" placeholder="First Name" />-->
        <Field id="last_name" type="text" name="last_name" placeholder="Last Name" v-model="form.last_name"/>
        <ErrorMessage name="last_name" class="error" />
<!--      <input v-model="form.last_name" placeholder="Last Name" />-->



      <!-- Поля для введення телефонів -->
      <div v-for="(phone, index) in form.phone_numbers" :key="index" style="margin-bottom: 10px;">

<!--        <Field
            style="width: 200px;"
            :id="'phone_number_' + index"
            type="text"
            :name="'phone_numbers[' + index + ']'"
            placeholder="Phone Number"
            v-model="form.phone_numbers[index]"
            @input="checkPhoneExists(index)"
        />-->

        <input
            v-model="form.phone_numbers[index]"
            @input="checkPhoneExists(index)"
            placeholder="Phone Number"
            style="width: 200px;"
        />
        <ErrorMessage name="phone_numbers" class="error" />
        <!-- Кнопка "+" для додавання нового поля -->
        <button @click="addPhone" v-if="index === form.phone_numbers.length - 1">+</button>

        <!-- Кнопка для видалення поля (з'являється, якщо більше одного поля) -->
        <button @click="removePhone(index)" v-if="form.phone_numbers.length > 1">Remove</button>

        <!-- Повідомлення про дублікати -->
        <p v-if="phoneErrors[index]">This phone number already exists in the database.</p>
      </div>

      <!-- Кнопка для сабміту форми -->
      <button @click="submitForm" :disabled="hasPhoneErrors">{{ form.id ? 'Update Contact' : 'Submit' }}</button>
      </Form>
    </div>

    <!-- Таблиця з контактами -->
    <div class="contacts-table">
      <h1>Contacts</h1>
      <table v-if="contacts.length > 0">
        <thead>
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
              <li v-for="phone in contact.phone_numbers" :key="phone.id">
                {{ phone.phone_number || 'N/A' }}
              </li>
            </ul>
          </td>
          <td>
            <button @click="editContact(contact)">Edit</button>
            <button @click="deleteContact(contact.id)">Delete</button>
          </td>
        </tr>
        </tbody>
      </table>
      <p v-else>No contacts available</p>

      <!-- Пагінація -->
      <div class="pagination" v-if="pagination.total > pagination.per_page">
        <button @click="fetchContacts(pagination.current_page - 1)" :disabled="!pagination.prev_page_url">
          &laquo; Previous
        </button>
        <span>Page {{ pagination.current_page }} of {{ pagination.last_page }}</span>
        <button @click="fetchContacts(pagination.current_page + 1)" :disabled="!pagination.next_page_url">
          Next &raquo;
        </button>
      </div>
    </div>
  </div>
</template>

<script>

import {toTypedSchema} from "@vee-validate/zod";
import {ErrorMessage, Field, Form} from "vee-validate";
import {z} from "zod";



export default {

  name: 'MainPage',
  components: {ErrorMessage, Field, Form},
  data() {
    return {
      contacts: [],
      pagination: {
        current_page: 1,
        last_page: 1,
        total: 0,
        per_page: 5,
        prev_page_url: null,
        next_page_url: null
      },
      form: {
        id: null, // Додаємо для редагування
        first_name: '',
        last_name: '',
        phone_numbers: ['']
      },
      phoneErrors: [], // Для зберігання помилок при перевірці телефонів
      hasPhoneErrors: false // Флаг для блокування сабміту форми, якщо є помилки
    };

  },
  setup() {
    const validationSchema = toTypedSchema(z.object({
      first_name: z.string().min(2, "This is required"),
      last_name: z.string().min(2, "This is required"),

    }));

    return {
      validationSchema,
    };
  },

  created() {
    this.fetchContacts();
  },
  methods: {
    async fetchContacts(page = 1) {
      try {
        const response = await fetch(`http://localhost:8000/api/all-contacts?page=${page}`);
        const data = await response.json();
        console.log('Fetched contacts:', data);

        // Оновлюємо список контактів і дані пагінації
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
      const phone = this.form.phone_numbers[index];
      try {
        const response = await fetch(`http://localhost:8000/api/check-phone?phone=${phone}`);
        const result = await response.json();

        if (result && result.exists) {
          this.phoneErrors[index] = true;
          this.hasPhoneErrors = true;
        } else {
          this.phoneErrors[index] = false;
          this.hasPhoneErrors = false;
        }
      } catch (error) {
        console.error('Error checking phone:', error);
      }
    },
    async submitForm() {
      const url = this.form.id
          ? `http://localhost:8000/api/contacts/${this.form.id}`
          : 'http://localhost:8000/api/contacts';
      const method = this.form.id ? 'PUT' : 'POST';

      try {
        const response = await fetch(url, {
          method: method,
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
          location.reload();

        } else {
          const errorDetails = await response.json();
          console.error('Error response:', errorDetails);
        }
      } catch (error) {
        console.error('Error submitting form:', error);
      }
    },
    async deleteContact(id) {
      try {
        const response = await fetch(`http://localhost:8000/api/contacts/${id}`, {
          method: 'DELETE'
        });

        if (response.ok) {
          console.log('Contact deleted');
          this.fetchContacts(); // Оновлюємо список контактів
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
      this.form.phone_numbers = contact.phone_numbers.map(p => p.phone_number); // Оновлюємо телефони для редагування
    },
    resetForm() {
      this.form = {
        id: null,
        first_name: '',
        last_name: '',
        phone_numbers: ['']
      };

    },


  }

};
</script>

<style scoped>
/* Стилі для форми та таблиці */
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
</style>
