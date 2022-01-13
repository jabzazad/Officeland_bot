<template>
  <div class="w-full min-h-screen bg-gray-200 p-2">
    <button @click="CallLogin" v-if="!accountName">Login in</button>
    <span>{{ accountName }}</span>
    <div class="grid grid-cols-2 gap-2 sm:grid-cols-7">
      <div v-for="(finish, index) in finishIds" :key="index" class="m-2">
        <div class="border rounded-xl shadow-lg p-4 bg-white">
          <div>AssetID: {{ finish.asset_id }}</div>
          <div>Reward: {{ Fix2(finish.reward) }}</div>
          <div>Fee: {{ FindTime(finish.finish_time).toFixed(2) }}%</div>
          <button
            class="
              py-2
              px-4
              mt-2
              w-full
              transition-colors
              duration-700
              transform
              bg-indigo-500
              text-xs
              hover:bg-blue-400
              text-gray-100 text-lg
              rounded-lg
              focus:border-4
              border-indigo-300
            "
            @click="ClaimReward(finish.finish_id, 'claimreward')"
          >
            Claim
          </button>
        </div>
      </div>
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
    Fix2(value) {
      if (value) {
        return parseInt(value).toFixed(2);
      }
    },
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
    async ClaimReward(taskID, task) {
      try {
        let data = {
          player: this.accountName,
        };

        if (task === "claimreward") {
          Object.assign(data, {
            finish_id: taskID,
          });
        } else {
          Object.assign(data, {
            asset_id: taskID,
            task_id: 2,
          });
        }

        const result = await wax.api.transact(
          {
            actions: [
              {
                account: "officegameio",
                name: task,
                authorization: [
                  {
                    actor: this.accountName,
                    permission: "active",
                  },
                ],
                data,
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
