<template>
  <div class="picker">
    <div class="card mb-3 shadow-sm">
      <h5 class="card-header" :class="{ 'text-white bg-success': 1 === 2 }">
        {{ step }}. Select {{ accountTypeLabel }}
      </h5>
      <div class="card-body pb-1">
        <!-- <h5 class="card-title">Special title treatment</h5> -->
        <div class="card-text">
          <div class="intro pb-4"><slot></slot></div>

          <div class="input-group mb-3">
            <input
              v-model="account"
              type="search"
              class="form-control"
              placeholder="XRP Ledger account address (r...)"
            />
            <button class="btn btn-primary" type="button" @click="go">
              Go
            </button>
          </div>

          <div v-if="!loading && error === '' && currencies.length > 0">
            <div v-if="lookedUpAccount" class="row mb-3">
              <div class="col-2">
                <img
                  :src="'https://xumm.app/avatar/' + account + '_150.png'"
                  width="150"
                  height="150"
                  class="img-fluid img-thumbnail"
                />
              </div>
              <div class="col-10">
                <h4 v-if="accountName !== lookedUpAccount" class="py-0 mb-0">
                  <b>{{ accountName }}</b>
                </h4>
                <code>{{ lookedUpAccount }}</code>
              </div>
            </div>

            <div v-if="isDestination">
              <h5>
                Pick currency &amp; amount to
                {{ isDestination ? "deliver" : "send" }}:
              </h5>

              <div class="input-group mb-3">
                <select class="input form-control" v-model="selectedCurrency">
                  <optgroup label="Assets">
                    <option
                      :value="currency.code"
                      v-for="currency in currencies"
                      v-bind:key="currency.code"
                    >
                      {{ currency.label }}
                    </option>
                  </optgroup>
                </select>
                <input
                  class="input form-control"
                  type="text"
                  placeholder="Amount (selected asset)"
                  v-model.number="amount"
                />
              </div>
            </div>
          </div>
        </div>

        <div v-if="loading" class="my-3 d-flex align-items-center">
          <strong>Talking to the XRP Ledger...</strong>
          <div
            class="spinner-border ms-auto"
            role="status"
            aria-hidden="true"
          ></div>
        </div>
        <div
          class="alert alert-danger mb-2 mt-3"
          v-if="!loading && error !== ''"
        >
          {{ error }}
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import { XrplClient, Call } from "xrpl-client";
import { utils } from "xrpl-txdata";

export default defineComponent({
  name: "AccountAndCurrencyPicker",
  data() {
    return {
      loading: false,
      error: "",
      account: "",
      accountName: "",
      lookedUpAccount: null,
      currencies: {},
      selectedCurrency: "XRP",
      amount: "",
    };
  },
  props: {
    accountTypeLabel: String,
    xrpl: XrplClient,
    ready: Function,
    step: String,
  },
  computed: {
    isDestination() {
      return this.accountTypeLabel?.match(/destination/i);
    },
  },
  watch: {
    account() {
      this.lookedUpAccount = null;
      this.accountName = "";
      this.selectedCurrency = "XRP";
      this.currencies = {};
      this.amount = "";

      if (this.ready) {
        this.ready(false);
      }
    },
    lookedUpAccount() {
      if (!this.isDestination && this.ready && this.lookedUpAccount) {
        this.ready(
          this.lookedUpAccount === this.account ? this.lookedUpAccount : false
        );
      }
    },
    selectedCurrency() {
      this.handleReadyDestination();
    },
    amount() {
      this.handleReadyDestination();
    },
  },
  mounted() {
    if (process.env.NODE_ENV === "development") {
      this.account = !this.accountTypeLabel?.match(/destination/i)
        ? "rwietsevLFg8XSmG3bEZzFein1g8RBqWDZ"
        : "rpePPeRpC89vpCY3CDzhzMCs78nPoNnAKm";
    }
  },
  methods: {
    handleReadyDestination() {
      if (!isNaN(Number(this.amount)) && this.ready) {
        this.ready(
          Number(this.amount) > 0
            ? this.lookedUpAccount +
                "|" +
                this.selectedCurrency +
                "|" +
                String(this.amount)
            : false
        );
      }
    },
    async call(command: Call) {
      this.loading = true;
      this.error = "";

      const results = await this.xrpl?.send(command);

      this.loading = false;

      if (results?.error) {
        this.error = (results.error + " " + results?.error_message).trim();
      }

      return results;
    },
    async go() {
      // console.log(this.account, this.accountTypeLabel);
      this.accountName = "";

      const results = await this.call({
        command: "account_currencies",
        account: this.account,
      });

      console.log({ results });

      this.currencies = [
        { code: "XRP", label: "XRP (native asset)" },
        ...(results?.receive_currencies || []).map((code: string) => {
          return {
            code,
            label: utils.currencyCodeFormat(code),
          };
        }),
      ];

      if (this.ready && !this.loading && this.error === "") {
        this.loading = true;

        this.accountName = "Loading account name...";
        const accountInfoCall = await fetch(
          "https://xumm.app/api/v1/platform/account-meta/" + this.account
        );
        const accountInfo = await accountInfoCall.json();
        this.lookedUpAccount = accountInfo?.account;
        this.accountName =
          accountInfo?.xummProfile?.accountAlias ||
          accountInfo?.thirdPartyProfiles?.[0]?.accountAlias;

        this.loading = false;
      }

      this.$nextTick(() => {
        window.scrollTo(0, document.body.scrollHeight);
      });
    },
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss"></style>
