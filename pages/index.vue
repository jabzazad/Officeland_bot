<template>
  <div>
    <span
      ><button
        v-if="!accountName"
        type="button"
        class="
          black--text
          font-weight-black
          mx-1
          v-btn v-btn--is-elevated v-btn--has-bg
          theme--dark
          v-size--x-small
          white
        "
        @click="CallLogin"
      >
        <span class="v-btn__content"
          ><div
            class="v-image v-responsive mx-1 theme--dark"
            style="max-height: 20px; max-width: 50px"
          >
            <div
              class="v-responsive__sizer"
              style="padding-bottom: 35.5%"
            ></div>
            <div
              class="v-image__image v-image__image--cover"
              style="
                background-image: url('https://legacy.wax.io/uploads/press-assets/wax-primary-logo.png');
                background-position: center;
              "
            ></div>
            <div class="v-responsive__content" style="width: 800px"></div>
          </div>
          Login
        </span>
      </button>
      <div v-else>{{ accountName }}</div>
    </span>
    <div v-for="(finish, index) in finishIds" :key="index">
      <div>
        AssetID: {{ finish.asset_id }} Reward: {{ finish.reward }} Fee:
        {{ FindTime(finish.finish_time).toFixed(2) }}%
      </div>
      <button @click="ClaimReward(finish.finish_id)">claim</button>
    </div>
  </div>
</template>

<script>
import * as waxjs from "@waxio/waxjs/dist";
import moment from "moment";
var wax;

export default {
  name: "IndexPage",
  data() {
    return {
      accountName: this.$cookies.get("account"),
      assetIds: [],
      maxslot: 0,
      taskIds: [],
      finishIds: [],
      accountProfile: [],
      blockHeadID: 0,
      assets: [],
    };
  },
  methods: {
    async CallLogin() {
      this.accountName = await wax.login();
      this.pubKey = wax.pubKeys;
      this.$cookies.set("account", this.accountName);
      this.$cookies.set("pub", this.pubKey);

      this.fetchConfig();
      this.fetchTask();
      this.fetchFinishTask();
      this.fetchProfile();
    },
    async autoLogin() {
      let isAutoLoginAvailable = await wax.isAutoLoginAvailable();
      if (isAutoLoginAvailable) {
        this.CallLogin();
      } else {
        this.CallLogin();
      }
    },
    autoSetWax() {
      wax = new waxjs.WaxJS({
        rpcEndpoint: "https://wax.greymass.com",
        userAccount: this.$cookies.get("account"),
        pubKeys: this.$cookies.get("pub"),
        tryAutoLogin: false,
      });
    },
    async fetchConfig() {
      this.$axios
        .post("https://wax.pink.gg/v1/chain/get_table_rows", {
          json: true,
          code: "officegameio",
          scope: "officegameio",
          table: "publicspace",
          lower_bound: this.accountName,
          upper_bound: this.accountName,
          index_position: 1,
          key_type: "",
          limit: 100,
          reverse: false,
          show_payer: false,
        })
        .then((response) => {
          this.assetIds = response.data.rows[0].asset_ids;
          this.maxSlot = response.data.rows[0].max_slot;
        });
    },
    async fetchTask() {
      this.$axios
        .post("https://wax.pink.gg/v1/chain/get_table_rows", {
          json: true,
          code: "officegameio",
          scope: "officegameio",
          table: "taskassign",
          lower_bound: this.accountName,
          upper_bound: this.accountName,
          index_position: 3,
          key_type: "name",
          limit: 100,
          reverse: false,
          show_payer: false,
        })
        .then((response) => {
          this.taskIds = response.data.rows;
        });
    },
    async fetchFinishTask() {
      this.$axios
        .post("https://wax.pink.gg/v1/chain/get_table_rows", {
          json: true,
          code: "officegameio",
          scope: "officegameio",
          table: "finishtask",
          lower_bound: this.accountName,
          upper_bound: this.accountName,
          index_position: 2,
          key_type: "name",
          limit: 1000,
          reverse: false,
          show_payer: false,
        })
        .then((response) => {
          this.finishIds = response.data.rows;
        });
    },
    async fetchProfile() {
      this.$axios
        .post("https://wax.pink.gg/v1/chain/get_table_rows", {
          json: true,
          code: "officegameio",
          scope: "officegameio",
          table: "balances",
          lower_bound: this.accountName,
          upper_bound: this.accountName,
          index_position: 1,
          key_type: "",
          limit: 1,
          reverse: false,
          show_payer: false,
        })
        .then((response) => {
          this.accountProfile = response.data.rows;
        });
    },
    async GetAsset() {
      for (assetID of assetIds) {
        if (assetID != 0) {
          url = "https://aa.dapplica.io/atomicassets/v1/assets/";
          this.$axios.get(url.concat(assetID)).then((response) => {
            if (response.data.owner == accountName) {
              assets.push({
                id: response.data.asset_id,
                object: response.data.data,
              }); 
            }
          });
        }
      }
    },
    FindTime(timeUnix) {
      var now = moment().format("X");
      return (((432000 - (now - timeUnix)) / 432000) * 100) / 2;
    },
    async ClaimReward(taskID) {
      try {
        const result = await wax.api.transact(
          {
            actions: [
              {
                account: "officegameio",
                name: "claimreward",
                authorization: [
                  {
                    actor: this.accountName,
                    permission: "active",
                  },
                ],
                data: {
                  finish_id: taskID,
                  player: this.accountName,
                },
              },
            ],
          },
          {
            blocksBehind: 3,
            expireSeconds: 1200,
          }
        );
        console.log(result);
      } catch (e) {
        console.log("\nCaught exception: " + e);
        if (e instanceof RpcError) console.log(JSON.stringify(e.json, null, 2));
      }
    },
  },
  mounted() {
    this.autoSetWax();
    this.autoLogin();
  },
};
</script>
