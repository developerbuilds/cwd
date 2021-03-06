<template>
  <div>
    <v-card class="mb-3">
    <v-btn router text color="dark" retain-focus-on-click to="/">Home </v-btn>
      <v-icon>mdi-chevron-right</v-icon>
      <v-btn router small text to="/cart">Cart </v-btn>
      <v-icon>mdi-chevron-right</v-icon>
      <v-btn router small text to="/cart/checkout">Checkout </v-btn>
    </v-card>
    <v-container>
      <v-layout row wrap>
        <v-flex xs12 sm12 md6 lg5>
          <v-card class="mx-auto mb-7" max-width="400" shaped elevation="23">
            <v-toolbar color="deep-purple darken-2" dark flat shaped>
              <v-toolbar-title>Checkout Details</v-toolbar-title>
            </v-toolbar>
            <v-card-text>
              <div class="font-weight-bold ml-8 mb-2">Today</div>
              <v-timeline align-top dense>
                <v-timeline-item
                  v-for="message in messages"
                  :key="message.time"
                  :color="message.color"
                  small>
                  <div>
                    <div class="font-weight-normal">
                      <strong>{{ message.from }}</strong>
                      <span v-if="message.id == 1"
                        >{{ last_added_item_date }} ago</span
                      >
                      <span v-else>{{ message.time }}</span>
                    </div>
                    <div>{{ message.message }}</div>
                  </div>
                </v-timeline-item>
              </v-timeline>
            </v-card-text>
          </v-card>
        </v-flex>
        <v-flex xs12 sm12 md6 lg6>
          <v-card shaped class="mb-7">
            <v-toolbar color="deep-purple darken-2" dark shaped>
              <v-toolbar-title>Checkout Form</v-toolbar-title>
            </v-toolbar>
            <v-card-text class="pa-7">
              <small class="ml-10">( * ) indicates mandatory fields</small
              ><br /><br />
              <validation-observer ref="observer">
                <form>
                  <validation-provider
                    v-slot="{ errors }"
                    name="address"
                    rules="required">
                    <v-textarea
                      outlined
                      shaped
                      rows="1"
                      auto-grow
                      v-model="shipping.address"
                      prepend-icon="mdi-map-marker"
                      :error-messages="errors"
                      label="Address*"
                      required>
                    </v-textarea>
                  </validation-provider>
                  <validation-provider>
                    <v-text-field
                      outlined
                      v-model="shipping.housenumber"
                      prepend-icon="mdi-home"
                      label="House Number (Optional)"
                      required>
                    </v-text-field>
                  </validation-provider>
                  <validation-provider
                    v-slot="{ errors }"
                    name="Phone Number"
                    rules="required">
                    <v-text-field
                      type="number"
                      outlined
                      prepend-icon="mdi-phone"
                      v-model="shipping.phone"
                      :counter="10"
                      hide-controls
                      :error-messages="errors"
                      label="Phone Number*"
                      required>
                    </v-text-field>
                  </validation-provider>
                  <validation-provider v-slot="{ errors }" name="Village Or City" rules="required">
                    <v-select
                      outlined
                      v-model="shipping.village_or_city"
                      prepend-icon="mdi-city"
                      :items="villages_or_cities"
                      :error-messages="errors"
                      label="Select City Or Village*"
                      data-vv-name="select"
                      required>
                    </v-select>
                  </validation-provider>
                  <validation-provider v-slot="{ errors }" name="Special Instructions" rules="required">
                    <v-textarea
                      :error-messages="errors" label="Special Instructions*"
                      v-model="shipping.special_instructions" auto-grow outlined required rows="3" row-height="25" shaped
                      prepend-icon="mdi-information"></v-textarea>
                  </validation-provider>
                </form>
              </validation-observer>
            </v-card-text>
            <v-progress-linear v-if="placing" indeterminate color="green"></v-progress-linear>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn class="mr-4 mb-4" @click="clear" text> clear </v-btn>
              <v-btn
                class="mr-10 mb-4"
                @click="submit()"
                color="deep-purple darken-1">
                Place Order</v-btn>
            </v-card-actions>
          </v-card>
        </v-flex>
      </v-layout>
    </v-container>
  </div>
</template>

<script>
import { required, email, max } from 'vee-validate/dist/rules'
import {
  extend,
  ValidationObserver,
  ValidationProvider,
  setInteractionMode,
} from 'vee-validate'
setInteractionMode('eager')
extend('required', {
  ...required,
  message: '{_field_} can not be empty',
})
extend('max', {
  ...max,
  message: '{_field_} may not be greater than {length} characters',
})
export default {
  components: { ValidationProvider, ValidationObserver },
  data: () => ({
    shipping: {
      address: '',
      housenumber: '',
      phone: '',
      village_or_city: null,
      errors: null,
      special_instructions: '',
    },
    time_after_30_mins: '',
    last_added_item_date: '',
    messages: [
      {
        id: 1,
        from: 'Last Item Added',
        color: 'deep-purple lighten-1',
      },
      {
        from: 'Order Placement',
        message: 'Maybe Today you want to Place the order ?',
        time: 'in just a moment',
        color: 'green',
      },
      {
        from: 'Order Arrival',
        message:
          'Most Probably You will get the order within half hour of placement',
        time: '',
        color: 'deep-purple lighten-1',
      },
    ],
    villages_or_cities: [
      'Churriwala Dhanna',
      'Nihal Khera',
      'Danger Kheda',
      'Killian wali',
      'Bathinda',
    ],
    placing: false,
    rules: {
      required: (value) => !!value || 'Required.',
    },
  }),
  activated() {
    // Call fetch again if last fetch more than 3 sec ago
    if (this.$fetchState.timestamp <= Date.now() - 3000) {
      this.$fetch()
    }
  },
  async fetch() {
    await this.settingDeliveryTime()
    await this.$axios
      .get('/store/cart-total-items')
      .then((response) => {
        if (response.data < 1){
          this.$toast.error("No Products in Cart")
          this.$router.push('/cart')
        }
        else{
          this.getLastAddedProduct()
        }
      })
    this.messages[2].time = 'at ' + this.time_after_30_mins
  },
  methods: {
    getLastAddedProduct() {
      this.$axios.get('/store/cart').then((response) => {
        let total = response.data.length
        this.last_added_item_date = response.data[total - 1].humanized_date
      })
    },
    submit() {
      this.$refs.observer.validate().then((response) => {
        if (response == true) {
          this.placing = true
          this.$axios
            .post('/store/place-order', this.shipping)
            .then((response) => {
              if (response.status == 202) {
                this.$toast.success('Order Placed Successfully')
                this.clear()
                this.placing = false
              } else {
                this.$toast.error('Please fill the shipping details correctly.')
              }
            })
            .catch((error)=>{
              if (error.response) {
            // client received an error response (5xx, 4xx)
            } else if (error.request) {
              // client never received a response, or request never left
            } else {
              // anything else
            }
            })
        } else {
          this.$toast.error('Please enter the required details.')
        }
      })
    },
    clear() {
      this.shipping.address = ''
      this.shipping.housenumber = ''
      this.shipping.phone = ''
      this.shipping.special_instructions = ''
      this.shipping.village_or_city = null
      this.$refs.observer.reset()
    },
    settingDeliveryTime() {
      let date = new Date()
      let hours = date.getHours()
      let days = date.getDay()
      let minutes = date.getMinutes() + 30
      if (minutes >= 60) {
        hours = hours + 1
        minutes = minutes - 60
      }
      var ampm = hours >= 12 ? 'pm' : 'am'
      hours = hours % 12
      hours = hours ? hours : 12
      minutes = minutes < 10 ? '0' + minutes : minutes
      var strTime = hours + ':' + minutes + ' ' + ampm
      this.time_after_30_mins = strTime
    },
  },
}
</script>
<style>
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
</style>