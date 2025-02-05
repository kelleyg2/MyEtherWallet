<template>
  <div class="dapps-stakewise-stake pt-8 pb-13 px-3 pa-sm-15">
    <v-row>
      <v-col
        :order="$vuetify.breakpoint.smAndDown ? 'last' : ''"
        cols="12"
        md="8"
        :class="$vuetify.breakpoint.smAndDown ? 'my-10' : 'pr-7'"
      >
        <mew-sheet class="pa-15">
          <div class="mew-heading-2 textDark--text mb-8">Unstake ETH</div>

          <!-- ======================================================================================= -->
          <!-- Stake direction information -->
          <!-- ======================================================================================= -->
          <div ref="input" class="d-flex align-center text-center">
            <div
              class="border-radius--8px bgWalletBlockDark flex-grow-1 pa-5 d-flex flex-column align-center"
              style="width: 30%"
            >
              <div
                class="mew-caption textMedium--text font-weight-regular mb-2"
              >
                You give
              </div>
              <div class="stake-icon">
                <img
                  src="@/assets/images/icons/icon-eth-gray.svg"
                  alt="Stakewise"
                />
              </div>
              <div class="font-weight-bold mt-2">MEWcbETH</div>
            </div>
            <div class="px-5">
              <v-icon color="greenPrimary">mdi-arrow-right</v-icon>
            </div>
            <div
              class="border-radius--8px bgWalletBlockDark flex-grow-1 pa-5 d-flex flex-column align-center"
              style="width: 30%"
            >
              <div
                class="mew-caption textMedium--text font-weight-regular mb-2"
              >
                You will get
              </div>
              <div class="stake-icon">
                <img src="@/assets/images/icons/icon-eth-gray.svg" alt="Eth" />
              </div>
              <div class="font-weight-bold mt-2">{{ currencyName }}</div>
            </div>
          </div>

          <!-- ======================================================================================= -->
          <!-- Amount to stake -->
          <!-- ======================================================================================= -->
          <div class="position--relative mt-15">
            <app-button-balance :loading="false" :balance="stakedBalance" />
            <mew-input
              type="number"
              :max-btn-obj="{
                title: 'Max',
                method: setMax
              }"
              :image="iconEth"
              label="Amount to unstake"
              placeholder="Enter amount"
              :value="unstakeAmount"
              :error-messages="errorMessages"
              :buy-more-str="buyMoreStr"
              @buyMore="openBuySell"
              @input="setAmount"
            />
          </div>

          <!-- ======================================================================================= -->
          <!-- Stake status -->
          <!-- ======================================================================================= -->
          <div class="stake-status">
            <div class="d-flex justify-space-between">
              <div>
                <div class="mew-body">
                  Network Fee
                  <span
                    class="ml-2 greenPrimary--text cursor--pointer"
                    @click="openSettings"
                  >
                    Edit
                  </span>
                </div>
              </div>
              <div class="text-right">
                <div class="">{{ ethTotalFee }} {{ currencyName }}</div>
                <div v-show="isEthNetwork" class="mew-body textLight--text">
                  {{ gasPriceFiat }}
                </div>
              </div>
            </div>
          </div>

          <!-- ======================================================================================= -->
          <!-- Divier -->
          <!-- ======================================================================================= -->
          <v-divider class="mt-8" />

          <!-- ======================================================================================= -->
          <!-- How stake works -->
          <!-- ======================================================================================= -->
          <div class="mt-6">
            <div class="font-weight-bold mb-2">How unstaking works</div>
            <ul class="textMedium--text">
              <li class="mb-2">
                Request to unstake the desired amount of
                {{ currencyName }}.
              </li>
              <li class="mb-2">
                Wait for {{ currencyName }} to become ready to claim. Status
                updates daily at 1pm UTC.
              </li>
              <li class="mb-2">
                Claim available {{ currencyName }} to unstake. Unstaked
                {{ currencyName }} will be deposited to your wallet.
              </li>
            </ul>

            <div class="mt-6">
              <a
                href="https://help.myetherwallet.com/en/articles/8843926-stake-eth-with-coinbase-in-mew-web"
                target="_blank"
              >
                <div class="greenPrimary--text">
                  View the ETH Staking guide<v-icon
                    color="greenPrimary"
                    small
                    class="ml-2"
                  >
                    mdi-open-in-new
                  </v-icon>
                </div>
              </a>
            </div>
          </div>

          <!-- ======================================================================================= -->
          <!-- Divier -->
          <!-- ======================================================================================= -->
          <v-divider class="mt-9 mb-8" />

          <!-- ======================================================================================= -->
          <!-- Start staking -->
          <!-- ======================================================================================= -->
          <div class="d-flex flex-column align-center">
            <mew-button
              class="mt-8"
              title="Unstake"
              btn-size="xlarge"
              :loading="loading"
              :disabled="!isValid"
              @click.native="unstake"
            />
          </div>
        </mew-sheet>
      </v-col>
      <v-col cols="12" md="4">
        <coinbase-staking-summary ref="summary" class="mb-4" />
      </v-col>
    </v-row>
  </div>
</template>

<script>
import handlerAnalytics from '@/modules/analytics-opt-in/handlers/handlerAnalytics.mixin';
import { fromWei } from 'web3-utils';
import { mapGetters, mapState } from 'vuex';
import BigNumber from 'bignumber.js';
import { debounce, isEmpty } from 'lodash';

import buyMore from '@/core/mixins/buyMore.mixin.js';
import { formatFloatingPointValue } from '@/core/helpers/numberFormatHelper';
import { ERROR, SUCCESS, Toast } from '@/modules/toast/handler/handlerToast';
import { EventBus } from '@/core/plugins/eventBus';
import hasValidDecimals from '@/core/helpers/hasValidDecimals';
import { toBase, fromBase } from '@/core/helpers/unit';
import { API, CB_TRACKING } from '@/dapps/coinbase-staking/configs.js';

const MIN_GAS_LIMIT = 400000;
export default {
  name: 'ModuleCoinbaseUnstaking',
  components: {
    CoinbaseStakingSummary: () => import('../components/CoinbaseStakingSummary')
  },
  mixins: [buyMore, handlerAnalytics],
  data() {
    return {
      iconEth: require('@/assets/images/icons/icon-eth-gray.svg'),
      unstakeAmount: '0',
      locGasPrice: '0',
      gasLimit: '21000',
      estimateGasError: false,
      loading: false
    };
  },
  computed: {
    ...mapGetters('wallet', ['balanceInETH']),
    ...mapGetters('global', [
      'network',
      'isEthNetwork',
      'gasPriceByType',
      'getFiatValue'
    ]),
    ...mapGetters('external', ['fiatValue']),
    ...mapState('global', ['gasPriceType']),
    ...mapState('wallet', ['web3', 'address', 'instance']),
    ...mapState('coinbaseStaking', ['fetchedDetails']),
    details() {
      return this.fetchedDetails[this.network.type.name];
    },
    hasDetails() {
      return !isEmpty(this.details);
    },
    stakedBalance() {
      return this.hasDetails
        ? fromBase(this.details.integratorShareUnderlyingBalance.value, 18)
        : 0;
    },
    currencyName() {
      return this.network.type.currencyName;
    },
    ethTotalFee() {
      const gasPrice = BigNumber(this.locGasPrice).gt(0)
        ? BigNumber(this.locGasPrice)
        : BigNumber(this.gasPriceByType(this.gasPriceType));
      const gasLimit = BigNumber(this.gasLimit).gt('21000')
        ? this.gasLimit
        : MIN_GAS_LIMIT;
      const ethFee = fromWei(BigNumber(gasPrice).times(gasLimit).toFixed());
      return formatFloatingPointValue(ethFee).value;
    },
    gasPriceFiat() {
      const gasPrice = BigNumber(this.ethTotalFee);
      return gasPrice.gt(0)
        ? this.getFiatValue(gasPrice.times(this.fiatValue).toFixed())
        : '0';
    },
    hasEnoughBalanceToStake() {
      return BigNumber(this.ethTotalFee).lte(this.balanceInETH);
    },
    isValid() {
      return (
        BigNumber(this.unstakeAmount).gt(0) &&
        this.hasEnoughBalanceToStake &&
        this.errorMessages === ''
      );
    },
    errorMessages() {
      if (!this.hasEnoughBalanceToStake) {
        return 'Not enough ETH.';
      }

      if (this.estimateGasError) {
        return !this.hasEnoughBalanceToStake
          ? 'Issue with gas estimation. Please check if you have enough balance!'
          : '';
      }
      if (BigNumber(this.stakedBalance).lte(0)) {
        return 'User has no ETH staked.';
      }
      if (BigNumber(this.unstakeAmount).lt(0)) {
        return 'Value cannot be negative';
      }
      if (BigNumber(this.unstakeAmount).gt(this.stakedBalance)) {
        return `Balance too low! User only has ${this.stakedBalance}.`;
      }
      if (
        BigNumber(this.unstakeAmount).gt(0) &&
        !hasValidDecimals(BigNumber(this.unstakeAmount).toFixed(), 18)
      ) {
        return 'Invalid decimals. ETH can only have 18 decimals';
      }
      return '';
    },
    buyMoreStr() {
      return this.isEthNetwork && !this.hasEnoughBalanceToStake
        ? this.network.type.canBuy
          ? 'Buy more.'
          : ''
        : null;
    }
  },
  watch: {
    gasPriceType() {
      this.locGasPrice = this.gasPriceByType(this.gasPriceType);
    }
  },
  mounted() {
    this.locGasPrice = this.gasPriceByType(this.gasPriceType);
  },
  methods: {
    reset() {
      this.setAmount(0);
      this.loading = false;
    },
    async unstake() {
      this.trackDapp(CB_TRACKING.CLICK_UNSTAKE);
      window.scrollTo(0, 0);
      this.loading = true;
      const { gasLimit, to, data, value, error } = await fetch(
        `${API}?address=${this.address}&action=unstake&networkId=${
          this.network.type.chainID
        }&amount=${toBase(this.unstakeAmount, 18)}`
      ).then(res => res.json());
      if (error) {
        const message = isEmpty(error)
          ? 'Something went wrong! Please try again!'
          : error.message;
        Toast(message, {}, ERROR);
        this.loading = false;
        this.trackDapp(CB_TRACKING.UNSTAKE_FAIL);
        return;
      }
      const txObj = {
        gasLimit: gasLimit,
        to: to,
        from: this.address,
        data: data,
        value: value
      };
      this.web3.eth
        .sendTransaction(txObj)
        .once('receipt', () => {
          EventBus.$emit('fetchSummary');
        })
        .then(() => {
          this.reset();
          EventBus.$emit('fetchSummary');
          Toast(
            'Successfully unstaked! Account will reflect once pool refreshes.',
            {},
            SUCCESS
          );
          this.trackDapp(CB_TRACKING.UNSTAKE_SUCCESS);
        })
        .catch(e => {
          this.reset();
          this.instance.errorHandler(e);
          this.trackDapp(CB_TRACKING.UNSTAKE_FAIL);
        });
    },
    setAmount: debounce(function (val) {
      const value = val ? val : 0;
      this.unstakeAmount = BigNumber(value).toFixed();
    }, 500),
    setMax() {
      if (this.hasEnoughBalanceToStake) {
        const max = BigNumber(this.stakedBalance);
        this.setAmount(max.toFixed());
      }
    },
    openSettings() {
      EventBus.$emit('openSettings');
    }
  }
};
</script>

<style lang="scss" scoped>
.stake-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px solid var(--v-borderInput-base) !important;
  border-radius: 50% !important;
  width: 52px;
  height: 52px;
  background-color: var(--v-alwaysWhite-base);

  img {
    height: 30px;
  }
}

ul {
  li {
    list-style: none;
    margin-bottom: 12px;

    &:before {
      font-size: 11px;
      content: '◆';
      margin-left: -23px;
      margin-right: 10px;
      color: var(--v-greenPrimary-base);
    }
  }
}
</style>
