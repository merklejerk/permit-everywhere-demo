<template>
    <div id="title">Poopiswap</div>
    <form>
        <span class="label">Sell Token</span>
        <select v-model="sellToken">
            <option v-for="(v,k) in tokens" :key="k" :value="v">{{k}}</option>
        </select>
        <span class="label">Buy Token</span>
        <select v-model="buyToken">
            <option v-for="(v,k) in tokens" :key="k" :value="v">{{k}}</option>
        </select>
        <span class="label">Sell amount</span>
        <input type="text" v-model="sellAmount" />
        <span class="label">Buy amount</span>
        <input type="text" v-model="buyAmount" disabled />
        <button @click="submit">Swap</button>
    </form>
</template>

<script>
import { Buffer } from 'buffer';
import * as ethers from 'ethers';
import Uni2RouterAbi from './assets/Uni2Router.abi.json';
import TestUni2RouterAbi from './assets/TestUni2Router.abi.json';
import ERC20PermitEverywhereAbi from './assets/ERC20PermitEverywhere.abi.json';

const TEST_ROUTERS = {
    '4': '0xE3e12cEAe9da93b81EA3fc4A722756790171D4Fe',
};

const UNI_ROUTERS = {
    '4': '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D',
};

const PES = {
    '4': '0x3979B968f44C05ad2b7481578829EF90562C9610'
};

const TOKENS = {
    '4': {
        'WETH': '0xc778417e063141139fce010982780140aa0cd5ab',
        'LUNA': '0xce603A86c89B062618f514D1d6eB42263890F33f',
    },
};

export default {
  name: 'App',
  components: {},
  data() {
      return {
          chainId: 4,
          sellToken: TOKENS[4].LUNA,
          buyToken: TOKENS[4].WETH,
          sellAmount: '0',
          buyAmount: '0',
      };
  },
  computed: {
      tokens() {
          return TOKENS[this.chainId];
      },
      fields() {
          return this.sellToken + this.buyToken + this.sellAmount;
      },
  },
  watch: {
      async fields() {
          this.buyAmount = await this._quote(this.sellToken, this.buyToken, this.sellAmount);
      },
  },
  async mounted() {
        this.provider = new ethers.providers.Web3Provider(window.ethereum);
        // MetaMask requires requesting permission to connect users accounts
        await this.provider.send("eth_requestAccounts", []);
        this.signer = this.provider.getSigner();
        this.uniRouter = new ethers.Contract(UNI_ROUTERS[this.chainId], Uni2RouterAbi, this.signer);
        this.testRouter = new ethers.Contract(TEST_ROUTERS[this.chainId], TestUni2RouterAbi, this.signer);
        this.pe = new ethers.Contract(PES[this.chainId], ERC20PermitEverywhereAbi, this.signer);
  },
  methods: {
      submit(e) {
          e.preventDefault(true);
          (async () => {
              const signerAddress = await this.signer.getAddress();
              const userNonce = await this.pe.currentNonce(signerAddress);
              const deadline = ethers.BigNumber.from(Math.floor(Date.now() / 1e3) + 60 * 60);
              const msgParams = {
                domain: {
                    chainId: this.chainId,
                    name: 'ERC20PermitEverywhere',
                    verifyingContract: this.pe.address,
                    version: '1.0.0',
                },
                message: {
                    token: this.sellToken,
                    spender: this.testRouter.address,
                    maxAmount: ethers.BigNumber.from(this.sellAmount),
                    deadline: deadline,
                    nonce: userNonce,
                },
                primaryType: 'PermitTransferFrom',
                types: {
                    PermitTransferFrom: [
                        { name: 'token', type: 'address' },
                        { name: 'spender', type: 'address' },
                        { name: 'maxAmount', type: 'uint256' },
                        { name: 'deadline', type: 'uint256' },
                        { name: 'nonce', type: 'uint256' },
                    ],
                },
            };
            const sigRaw = await this.signer._signTypedData(msgParams.domain, msgParams.types, msgParams.message);
            const sigBuf = Buffer.from(sigRaw.slice(2), 'hex');
            const sig = {
                r: '0x'+sigBuf.slice(0, 32).toString('hex'),
                s: '0x'+sigBuf.slice(32, 64).toString('hex'),
                v: parseInt(sigBuf.slice(64, 65).toString('hex'), 16),
            };
            const r = await (await this.testRouter.swapExactTokensForETH(
                msgParams.message.maxAmount,
                ethers.BigNumber.from(0),
                [this.sellToken, this.buyToken],
                signerAddress,
                deadline,
                {
                    token: this.sellToken,
                    spender: this.testRouter.address,
                    maxAmount: ethers.BigNumber.from(this.sellAmount),
                    deadline: deadline,
                },
                sig,
            )).wait();
            console.log(r);
          })();
      },
      async _quote(sellToken, buyToken, sellAmount) {
          const r = await this.uniRouter.getAmountsOut(ethers.BigNumber.from(sellAmount), [sellToken,buyToken]);
          return r[1].toString();
      },
  },
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

#title {
    font-size: 2em;
    font-weight: bold;
    text-align: center;
}

#app > form {
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    width: 64ex;
}

#app > form > .label {
    display: block;
    text-align: left;
}

#app > form > input,select {
    display: block;
    margin: 1em 0;
}
</style>
