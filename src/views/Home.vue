<template>
  <div class="home">
    <img alt="logo" src="../assets/img/bg1.png" class="logo" >
    <div class="btn-group" v-if="identity">
    <!--  <h1 class="title"> The Happy EOS Slot </h1>-->
      <FetchProfile :account_name="identity.accounts[0].name" symbol="EOS" v-if="identity" />
      <button class="button" @click="buy">Buy Credits</button>
      <button class="button" @click="withdraw">Sell Credits</button>

     <!-- <button class="button" @click="() => bet(100000)">Let's Bet</button>
      <button class="button" @click="luckyBet"> I'm feeling Lucky </button>
      <br>
      <button class="button is-danger" @click="updateAuth">updateAuth</button>
      <button class="button is-danger" @click="signOut">LOGOUT</button>-->
      <!-- <button class="button" @click="getPublicKey">getPublicKey</button> -->
    </div>
    <div class="btn-group" v-else>
      <button class="button" @click="requestId">Fetch Scatter ID</button>
    </div>
    
  </div>
</template>

<script>
// @ is an alias to /src
import { mapState, mapMutations } from "vuex";
import { sha256 } from "eosjs-ecc";
import HelloWorld from "@/components/HelloWorld.vue";
import FetchProfile from "@/components/FetchProfile.vue";
import axios from "axios";
import { networks } from "../config";
import randUuid from "uuid/v4";
const network = networks["kylin"];
const requiredFields = { accounts: [network] };

export default {
  name: "home",
  // data: () => ({
  //   account_name: "happyeosslot"
  // }),
  computed: {
    ...mapState(["identity", "scatter", "eos", "account"]),
    ...mapState({
      account_name: state => state.account.name
    })
  },
  components: {
    HelloWorld,
    FetchProfile
  },
  created() {
    this.getPublicKey();
    this.initContract();
  },
  methods: {
    ...mapMutations(["setIdentity"]),
    async suggestNetworkSetting() {
      try {
        await this.scatter.suggestNetwork(network);
      } catch (error) {
        console.info("User canceled to suggestNetwork");
        return;
      }
    },
    notification(msg1, msg2) {
      alert(msg2);
    },
    async requestId() {
      await this.suggestNetworkSetting();
      const identity = await scatter.getIdentity(requiredFields);
      this.setIdentity(identity);
    },
    async signOut(e) {
      try {
        await this.scatter.forgetIdentity(network);
        this.setIdentity(null);
      } catch (error) {}
    },
    async transfer() {
      const amount = Number(
        prompt("How much EOS you want to buy the credits?")
      ).toFixed(4);
      const { account_name } = this;
      this.eos
        .transfer(account_name,'tmonomononon', `${amount} EOS`, "")
        .then(() => {
          this.notification("succeeded", "转账成功");
          return Promise.resolve(null);
        })
        .catch(err => {
            alert(JSON.stringify(err))
          this.notification("error", "购买失败");
          return Promise.reject(err);
        });
    },
    async getPublicKey() {
      const { account_name } = this;
      const { data } = await axios({
        method: "post",
        url: "https://nodes.get-scatter.com/v1/chain/get_account",
        headers: {
          Accept: "application/json",
          "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"
        },
        data: { account_name }
      });
      const { permissions } = data;
      const permission = permissions.find(
        permit => permit.perm_name === "active"
      );
      const { key } = permission.required_auth.keys[0];
      return key;
    },
    updateAuth() {
      return this.getPublicKey()
        .then(key => {
          return this.eos.updateauth({
            account: this.account.name,
            permission: this.account.authority,
            parent: "owner",
            auth: {
              threshold: 1,
              keys: [
                {
                  key: key,
                  weight: 1
                }
              ],
              accounts: [
                {
                  permission: {
                    actor: "happyeosslot",
                    permission: "eosio.code"
                  },
                  weight: 1
                }
              ]
            }
          });
        })
        .then(() => {
          this.notification("succeeded", "对合约账户授权成功");
          return Promise.resolve(null);
        })
        .catch(err => {
          this.notification("error", "对合约账户授权失败", err.toString());
          return Promise.reject(err);
        });
    },
    withdraw() {
      const amount = parseInt(prompt("How much credits you want to sell?"));
      this.eos
        .contract("happyeosslot", { requiredFields })
        .then(contract =>
          contract.sell(this.account.name, amount, {
            authorization: [`${this.account.name}@${this.account.authority}`]
          })
        )
        .then(() => {
          this.notification("succeeded", "兑换成功");
        })
        .catch(err => {
          this.notification("error", "兑换失败", err.toString());
        });
    },
    bet({ seed, amount }) {
      console.info(`Seed converted into sha256: ${sha256(seed)}`);
      this.eos
        .contract("happyeosslot", { requiredFields })
        .then(contract =>
          contract.bet(this.account.name, parseInt(amount), sha256(seed), {
            authorization: [`${this.account.name}@${this.account.authority}`]
          })
        )
        .then(() => {
          this.notification("succeeded", "BET 交易发送成功");
        })
        .catch(err => {
          this.notification("error", "BET 交易发送失败", err.toString());
        });
    },
    luckyBet() {
      const amount = parseInt(prompt("How much credits you want to bet?"));
      const seed = randUuid();
      this.bet({ seed, amount });
    },
      initContract() {
          this.eos
              .contract("happyeosslot", { requiredFields })
              .then(contract =>
                  contract.init('happyeosslot', 'd533f24d6f28ddcef3f066474f7b8355383e485681ba8e793e037f5cf36e4883', {
                      authorization: [`${this.account.name}@${this.account.authority}`]
                  })
              )
              .then(() => {
//                  this.notification("succeeded", "init success");
              })
              .catch(err => {
                  this.notification("error", "init fail", err.toString());
              });
      },
      buy(){
          const amount = Number(
              prompt("How much EOS you want to buy the credits?")
          ).toFixed(4);
          const { account_name } = this;
          alert(this.account.authority + amount)
        this.eos.contract("happyeosslot",{requiredFields})
            .then(contract =>
                contract.buy(account_name,`${amount} EOS`,{
                      authorization:[`${account_name}@${this.account.authority}`]
                  })).then((data)=>{
            console.log(data)
                     alert(JSON.stringify(data))
        }).catch(err=>{
            alert(JSON.stringify(err))
        })
      }
  }
};
</script>

<style scoped>
.logo {
  max-width: 80%;
}
  .home{
    margin:20px;
  }
  .btn-group{
    width: 80%;
    text-align: center;
    height: 10px;
  }
  .button{
    margin: 10px;
    background: yellow;
  }
</style>
