<template>
  <div class="pathfinder">
    <div class="card mb-3 shadow-sm">
      <h5 class="card-header" :class="{ 'text-white bg-primary': 1 === 1 }">
        4. Path Finding 🎉
      </h5>
      <div class="card-body pb-1">
        <!-- <h5 class="card-title">Special title treatment</h5> -->
        <div class="card-text">
          <div class="intro pb-4"><slot></slot></div>
        </div>

        <div v-if="loading" class="my-3 d-flex align-items-center">
          <strong>Asked the XRP Ledger to find paths...</strong>
          <div
            class="spinner-border ms-auto"
            role="status"
            aria-hidden="true"
          ></div>
        </div>
        <div v-else>
          <div v-if="paths && paths.length > 0">
            <h5 class="mt-0 pt-0 mb-3">
              Found {{ paths.length }} path(s) to deliver
              {{
                typeof this.command.destination_amount === "string"
                  ? Number(this.command.destination_amount) / 1_000_000
                  : this.command.destination_amount.value
              }}
              {{
                typeof this.command.destination_amount === "string"
                  ? "XRP"
                  : currencyCodeFormat(this.command.destination_amount.currency)
              }}
            </h5>
            <div class="row">
              <div
                class="mb-4 col-12 col-lg-3 col-md-4 col-sm-6 d-flex align-self-stretch flex-wrap"
                v-for="(path, i) in paths"
                v-bind:key="i"
              >
                <div class="card w-100 shadow-sm">
                  <div
                    class="card-header text-center text-white bg-primary fw-bold h4 py-1"
                    style="
                      white-space: nowrap;
                      overflow: hidden;
                      text-overflow: ellipsis;
                    "
                  >
                    {{
                      typeof path.source_amount === "string"
                        ? "XRP"
                        : currencyCodeFormat(path.source_amount.currency)
                    }}
                  </div>
                  <div class="card-body pb-4 text-center">
                    <!-- <img :src="'https://xumm.app/avatar/' + account + '_150.png'" width="150" height="150" class="img-fluid img-thumbnail" /> -->
                    <a
                      target="_blank"
                      v-if="typeof path.source_amount !== 'string'"
                      :href="
                        'https://bithomp.com/explorer/' +
                        path.paths_computed?.[0]?.[0]?.account
                      "
                      ><small
                        style="
                          bottom: 0;
                          position: absolute;
                          left: 0;
                          right: 0;
                          margin-bottom: 10px;
                        "
                        ><code
                          class="d-block px-2"
                          style="
                            white-space: nowrap;
                            text-overflow: ellipsis;
                            overflow: hidden;
                          "
                          >{{ path.paths_computed?.[0]?.[0]?.account }}</code
                        ></small
                      ></a
                    >
                    <div v-if="typeof path.source_amount === 'string'">
                      <h4>{{ path.source_amount / 1_000_000 }}</h4>
                    </div>
                    <div v-else style="position: relative">
                      <h4>{{ path.source_amount.value }}</h4>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
          <div v-else>
            <div class="alert alert-danger text-center">
              <div class="pb-2"><b>No paths found.</b></div>
              Could be due to insufficient liquidity.
            </div>
          </div>
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

interface XrplPathComponent {
  currency?: string;
  issuer?: string;
  type: number;
}

interface XrplCurrency {
  currency: string;
  issuer: string;
  value: string;
}

interface XrplPath {
  paths_canonical: XrplPathComponent[];
  paths_computed: XrplPathComponent[];
  source_amount: XrplCurrency | number;
}

type XrplPaths = XrplPath[];

let watchToGo: ReturnType<typeof setTimeout>;

let paths: XrplPaths = [];

export default defineComponent({
  name: "PathFinder",
  data() {
    return {
      lastId: "",
      loading: true,
      error: "",
      pfResults: {},
      paths,
      command: {},
    };
  },
  props: {
    xrpl: XrplClient,
    params: Object,
  },
  computed: {},
  watch: {
    params: {
      deep: true,
      handler() {
        this.pfResults = {};
        this.loading = true;
        this.error = "";

        clearTimeout(watchToGo);
        watchToGo = setTimeout(() => {
          this.go();
        }, 3_000);
      },
    },
  },
  mounted() {
    console.log(this.params?.destination);
    console.log(this.params?.source);
    this.go();
  },
  methods: {
    rand(size: number) {
      return [...Array(size)]
        .map(() => Math.floor(Math.random() * 16).toString(16))
        .join("");
    },
    currencyCodeFormat(code: string) {
      return utils.currencyCodeFormat(code);
    },
    async go() {
      this.loading = true;
      this.error = "";

      this.xrpl?.send({
        id: this.lastId,
        command: "path_find",
        subcommand: "close",
      });

      this.lastId = this.rand(6);

      const command: Call = {
        id: this.lastId,
        command: "path_find",
        subcommand: "create",
        source_account: this.params?.source?.account || "",
        destination_account: this.params?.destination?.account || "",
        destination_amount:
          (this.params?.destination?.asset || "") === "XRP"
            ? String(
                (Number(this.params?.destination?.amount || 0) || 0) * 1_000_000
              )
            : {
                value: this.params?.destination?.amount || "0",
                currency: this.params?.destination?.asset || "",
                issuer: this.params?.destination?.account || "",
              },
      };

      console.log("Pathfinding command", { command });

      this.command = command;

      const results = await this.xrpl?.send(command);
      console.log("pfcall", { command, results });

      this.paths = (results?.result?.alternatives || []) as XrplPaths;

      this.xrpl?.on("path", (path) => {
        if (path?.destination_amount) {
          console.log("Async path", path.id, path.id === this.lastId, path);
          if (path.id === this.lastId) {
            this.paths = (path?.alternatives || []) as XrplPaths;
          } else {
            // Old command, close
            this.xrpl?.send({
              id: path.id,
              command: "path_find",
              subcommand: "close",
            });
          }
        }
      });

      this.loading = false;

      if (results?.error) {
        this.error = (results.error + " " + results?.error_message).trim();
      }

      return results;
    },
  },
});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss"></style>
