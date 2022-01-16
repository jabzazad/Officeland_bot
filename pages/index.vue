<template>
  <div class="flex-row bg-gray-200 p-2">
    <button @click="CallLogin" v-if="!accountName">Login in</button>
    <span>{{ accountName }}</span>
    <div class="flex">
      <div class="w-1/2">
        <div class="border rounded-xl shadow-lg p-4 bg-white">
          <div class="w-full bg-gray-200 rounded-full">
            <div
              v-if="calculateBar() >= 100"
              class="
                bg-red-600
                text-xs
                font-medium
                text-red-100 text-center
                p-0.5
                leading-none
                rounded-l-full
              "
              style="width: 100%"
            >
              {{ calculateBar().toFixed(2) }}%
            </div>
            <div
              v-else
              class="
                bg-blue-600
                text-xs
                font-medium
                text-blue-100 text-center
                p-0.5
                leading-none
                rounded-l-full
              "
              :style="{ width: calculateBar() + '%' }"
            >
              {{ calculateBar().toFixed(2) }}%
            </div>
            <br />
            Steaking: {{ steakCPU }}
            <br />
            <br />
            <br />

            <div
              v-for="(worker, index) in workers"
              :key="index"
              class="grid grid-cols-1 gap-1 sm:grid-cols-1"
            >
              AssetID: {{ worker.assetId }} TaskID:
              {{ worker.taskAssignID }} Rarity: {{ worker.rarity }} Time:
              {{ convertHMS(calculateTime(worker)) }}
            </div>
          </div>
        </div>
      </div>
      <div class="grid w-1/2 grid-cols-1 gap-1 sm:grid-cols-1">
        <div v-for="(finish, index) in finishIds" :key="index" class="m-2">
          <div class="border rounded-xl shadow-lg p-4 bg-white">
            <div>AssetID: {{ finish.asset_id }}</div>
            <div>Reward: {{ Fix2(finish.reward) }}</div>
            <div>
              Fee: {{ FindTime(finish.finish_time, finish).toFixed(2) }}%
            </div>
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
              @click="Working(finish.finish_id, 'claimreward', 0)"
            >
              Claim
            </button>
          </div>
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
      breaker: 0,
      accountName: this.$cookies.get("account"),
      assetIds: [],
      maxslot: 0,
      taskIds: [],
      finishIds: [],
      accountProfile: [],
      blockHeadID: 0,
      assets: [],
      accountInfomation: {},
      maxCPU: 0,
      usedCPU: 0,
      steakCPU: "",
      workers: [],
    };
  },
  methods: {
    Fix2(value) {
      if (value) {
        return parseInt(value).toFixed(2);
      }
    },
    calculateBar() {
      return 100 - ((this.maxCPU - this.usedCPU) / this.maxCPU) * 100;
    },
    async CallLogin() {
      this.accountName = await wax.login();
      this.pubKey = wax.pubKeys;
      this.$cookies.set("account", this.accountName);
      this.$cookies.set("pub", this.pubKey);

      var timer = 0;
      var callFunc = setInterval(() => {
        timer++;
        if (timer >= 500) {
          this.breaker = 0;
          timer = 0;
        }

        if (this.breaker > 40) {
          clearInterval(callFunc);
        }

        this.callEveryFunction();
      }, 25000);
    },
    async callEveryFunction() {
      await this.fetchConfig();
      await this.fetchTask();
      await this.GetAsset();
      await this.fetchFinishTask();
      await this.fetchProfile();
      await this.fetchAccount();
      await this.mapTasking();
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
      await this.$axios
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
      await this.$axios
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
      await this.$axios
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
      await this.$axios
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
    async assignTask() {
      for (this.asset of this.assets) {
        let isFound = false;
        for (this.task of this.taskIds) {
          if (this.task.asset_id == this.asset.id) {
            isFound = true;
          }
        }
        if (!isFound) {
          let taskID = 0
          switch (this.asset.object.data.rarity) {
            case "junior":
              taskID = 3;
              break;
            case "senior":
              taskID = 4;
              break;
            case "leader":
              taskID = 5;
              break;
            case "manager":
              taskID = 6;
              break;
          }
          setTimeout(() => {
            this.Working(this.asset.id, "assigntask", taskID);
          }, 500);
        }
      }
    },
    async fetchAccount() {
      await this.$axios
        .post("https://wax.pink.gg/v1/chain/get_account", {
          account_name: this.accountName,
        })
        .then((response) => {
          this.accountInfomation = response.data;
          this.maxCPU = response.data.cpu_limit.max;
          this.usedCPU = response.data.cpu_limit.used;
          this.steakCPU = response.data.total_resources.cpu_weight;
        });
    },
    calculateTime(worker) {
    let timenow = moment().format("X")

      if ((worker.taskEnd - timenow) <= 0) {
        setTimeout(() => {
          this.Working(worker.taskAssignID, "taskfinished");
        }, 4000);
      }

      return worker.taskEnd - timenow;
    },
    convertHMS(value) {
      const sec = parseInt(value, 10); // convert value to number if it's string
      let hours = Math.floor(sec / 3600); // get hours
      let minutes = Math.floor((sec - hours * 3600) / 60); // get minutes
      let seconds = sec - hours * 3600 - minutes * 60; //  get seconds
      // add 0 if value < 10; Example: 2 => 02
      if (hours < 10) {
        hours = "0" + hours;
      }
      if (minutes < 10) {
        minutes = "0" + minutes;
      }
      if (seconds < 10) {
        seconds = "0" + seconds;
      }
      return hours + ":" + minutes + ":" + seconds;
    },
    async mapTasking() {
      this.workers = [];
      for (this.task of this.taskIds) {
        for (this.asset of this.assets) {
          if (this.task.asset_id == this.asset.id) {
            this.workers.push({
              assetId: this.task.asset_id,
              taskEnd: this.task.task_end,
              rare: this.asset.object.data.rarity,
              taskAssignID: this.task.taskassign_id,
            });
          }
        }
      }

      if (this.taskIds.length != this.assets.length) {
        setTimeout(() => {
          this.assignTask();
        }, 4000);
      }
    },
    async GetAsset() {
      this.assets = [];
      for (this.assetID of this.assetIds) {
        if (this.assetID != 0) {
          let url = "https://aa.dapplica.io/atomicassets/v1/assets/";
          await this.$axios.get(url + this.assetID).then((response) => {
            if (response.data.data.owner === this.accountName) {
              this.assets.push({
                id: response.data.data.asset_id,
                object: response.data.data,
              });
            }
          });
        }
      }
    },
    FindTime(timeUnix, finish) {
     let now = moment().format("X");
      console.log("test now:",now)
      if ((((432000 - (now - timeUnix)) / 432000) * 100) / 2 <= 0) {
        setTimeout(() => {
          this.Working(finish.finish_id, "claimreward", 0);
        }, 4000);
      }

      return (((432000 - (now - timeUnix)) / 432000) * 100) / 2;
    },
    async Working(id, task, taskID) {
      console.log(id, task, taskID);
      this.breaker++;
      try {
        let data = {
          player: this.accountName,
        };

        switch (task) {
          case "claimreward":
            Object.assign(data, {
              finish_id: id,
            });
            break;
          case "assigntask":
            Object.assign(data, {
              asset_id: id,
              task_id: taskID,
            });
            break;
          case "taskfinished":
            Object.assign(data, {
              taskassign_id: id,
            });
            break;
        }

        console.log(data);

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
