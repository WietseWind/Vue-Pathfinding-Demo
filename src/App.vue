<template>
  <div class="container mt-3 pb-5">
    <h3>
      <div class="ledger-state float-end">
        <div
          class="badge alert-primary border-primary border shadow-sm"
          v-if="ready.ledger && lastLedger !== ''"
        >
          <small style="font-size: 0.6em" class="pe-2">Ledger</small>
          <code class="text-primary">{{ lastLedger }}</code>
        </div>
      </div>
      XRPL Pathfinding Demo
    </h3>

    <p class="h5 fw-normal my-4">
      This sample VueJS (TS, Vue3) project shows how easy it is to use
      pathfinding on the XRP Ledger, obtaining a list of assets and amounts to
      send to deliver a specific amount of another asset.
    </p>

    <div class="card mb-3 shadow-sm">
      <h5
        class="card-header"
        :class="{ 'text-white bg-success': ready.ledger && lastLedger !== '' }"
      >
        1. XRP Ledger connection
      </h5>
      <div class="card-body">
        <!-- <h5 class="card-title">Special title treatment</h5> -->
        <p class="card-text">
          As soon as this page is loaded a WebSocket connection will be made
          using the
          <a href="https://www.npmjs.com/package/xrpl-client" target="_blank"
            ><code>xrpl-client</code></a
          >
          package to the public nodes of
          <a href="https://xrplcluster.com/" target="_blank"
            ><code>xrplcluster.com</code></a
          >.
        </p>
        <!-- <a href="#" class="btn btn-primary">Go somewhere</a> -->
      </div>
    </div>

    <div v-if="ready.ledger && lastLedger !== ''">
      <AccountAndCurrencyPicker
        step="2"
        accountTypeLabel="destination account, asset and amount"
        :xrpl="xrpl"
        :ready="destinationReady"
        >Enter an activated XRP Ledger account (r...) on <i>mainnet</i>.
        Preferably an account with multiple Trust Lines setup, because cross
        asset path finding is a lot of fun.
      </AccountAndCurrencyPicker>

      <AccountAndCurrencyPicker
        v-if="ready.destination"
        step="3"
        accountTypeLabel="source account"
        :xrpl="xrpl"
        :ready="sourceReady"
        >Now select a source account. Same applies here: preferably an account
        with multiple Trust Lines setup, because cross asset path finding is a
        lot of fun.</AccountAndCurrencyPicker
      >

      <PathFinder
        :xrpl="xrpl"
        :params="pathfindingParams"
        v-if="ready.destination && ready.source"
        >Where the magic happens... (Spoiler alert: on the XRPL DEX). This is
        where we ask the XRP Ledger to find us the assets &amp; their amount we
        can use to satisfy delivering the selected destination asset &amp;
        amount.</PathFinder
      >
    </div>
    <div v-else class="d-flex align-items-center mt-4">
      <strong>Connecting to the XRP Ledger...</strong>
      <div
        class="spinner-border ms-auto"
        role="status"
        aria-hidden="true"
      ></div>
    </div>
    <!-- <pre>{{ pathfindingParams }}</pre> -->
    <div class="py-2">
      <a href="https://github.com/WietseWind/Vue-Pathfinding-Demo"
        >Source code</a
      >
      by <a href="https://wietse.com">Wietse</a>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import AccountAndCurrencyPicker from "./components/AccountAndCurrencyPicker.vue";
import PathFinder from "./components/PathFinder.vue";
import { XrplClient } from "xrpl-client";

const xrpl = new XrplClient();

export default defineComponent({
  name: "App",
  computed: {
    xrpl() {
      return xrpl;
    },
  },
  methods: {
    destinationReady(ready: boolean | string) {
      console.log("dready", ready);
      this.ready.destination = !!ready;
      if (this.ready.destination) {
        [
          this.pathfindingParams.destination.account,
          this.pathfindingParams.destination.asset,
          this.pathfindingParams.destination.amount,
        ] = String(ready || "").split("|");

        this.$nextTick(() => {
          window.scrollTo(0, document.body.scrollHeight);
        });
      }
    },
    sourceReady(ready: boolean | string) {
      console.log("sready", ready);
      this.ready.source = !!ready;
      if (this.ready.source) {
        [this.pathfindingParams.source.account] = String(ready || "").split(
          "."
        );

        this.$nextTick(() => {
          window.scrollTo(0, document.body.scrollHeight);
        });
      }
    },
  },
  data() {
    return {
      ready: {
        ledger: false,
        destination: false,
        source: false,
      },
      pathfindingParams: {
        destination: {
          account: "",
          asset: "",
          amount: "",
        },
        source: {
          account: "",
        },
      },
      lastLedger: "",
      state: {},
      ledger: {},
    };
  },
  components: {
    AccountAndCurrencyPicker,
    PathFinder,
  },
  mounted() {
    this.xrpl.ready().then(() => {
      this.ready.ledger = true;
    });
    this.xrpl.on("state", (state) => {
      console.log("XRPL Node:", state.server.publicKey);
      this.state = state;
    });
    this.xrpl.on("ledger", (ledger) => {
      // console.log(ledger);
      this.ledger = ledger;
      this.lastLedger = (ledger?.validated_ledgers || "")
        .split("-")
        .reverse()[0]
        .split(",")
        .reverse()[0];
    });
  },
});
</script>

<style lang="scss">
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
}
</style>
