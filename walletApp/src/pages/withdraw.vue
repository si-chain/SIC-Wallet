<template>
  <div>
    <x-header :left-options="{backText: ''}">{{$t('withdraw.extract')}}</x-header>
    <group>
      <!-- <x-input :title="$t('withdraw.to')" label-width="4.5em" required :placeholder="$t('withdraw.to_placeholder')" :is-type="validateAccount" v-model="ToAccount"></x-input> -->
      <x-input :title="$t('withdraw.amount')" type="number" label-width="4.5em" required :placeholder="$t('withdraw.amount_placeholder')" :is-type="validateAmount" v-model="amount"></x-input>
      <x-input :title="$t('withdraw.memo')" label-width="4.5em" :placeholder="$t('withdraw.memo')" v-model="memo"></x-input>
    </group>
    <box gap="30px 15px">
      <x-button type="primary" @click.native="next" :show-loading="loading">{{btnTitle}}</x-button>
    </box>
    <!-- <toast v-model="show" :type="isSuccess">{{error}}</toast> -->
    <alert v-model="show" :title="$t('index.notice')" :content="error"></alert>
    <password-confirm v-if="isUnlock" ref="confirm" @setUnlock="setUnlock" @unlocking="unlocking"></password-confirm>
  </div>
</template>
<script>
import Eos from 'eosjs'
import config from '../libs/env'
import passwordConfirm from '../components/PasswordConfirm'
import { XHeader, Group, XInput, Box, Toast, XButton, Alert } from 'vux'
export default {
  components: {
    Group, XInput, XHeader, Box, Toast, XButton, passwordConfirm, Alert
  },
  data () {
    return {
      // ToAccount: '',
      amount: '',
      memo: '',
      show: false,
      loading: false,
      isUnlock: false,
      btnTitle: this.$t('withdraw.next'),
      pwd: '',
      isSuccess: 'warn',
      error: '',
      validateAmount: function (value) {
        return {
          valid: value > 0.0001,
          msg: this.$t('withdraw.error.amount.minimum')
        }
      }
    }
  },
  methods: {
    setUnlock (val) {
      this.isUnlock = val
    },
    unlocking (pwd) {
      this.pwd = pwd
      this.loading = true
      this.isUnlock = false
      this.btnTitle = this.$t('withdraw.sending')
      this.withdraw()
    },
    withdraw () {
      if (this.amount > 0.0001) {
        let _this = this
        try {
          let accountStr
          this.$common.getStore('account').map(item => {
            if (item.account === _this.$store.state.account) {
              accountStr = item
            }
          })
          let accountData = JSON.parse(this.$common.decryptActive(accountStr.encryption, this.pwd))
          let activeKey = this.$common.decryptActive(accountData.active, this.pwd)
          config.keyProvider = activeKey
          let eos = Eos.Localnet(config)
          // 合约，固定
          const contractName = 'sic.token'
          const contractPromise = eos.contract(contractName)
          try {
            contractPromise.then(contract => {
              contract.withdraw({
                from: accountStr.account,
                quantity: _this.amount + ' SIC',
                memo: _this.memo + Date.now()
              }, {authorization: accountStr.account}).then(res => {
                _this.isSuccess = 'success'
                _this.show = true
                _this.error = _this.$t('withdraw.success.title')
                _this.loading = false
                _this.btnTitle = _this.$t('withdraw.success.title')
                setTimeout(() => {
                  _this.$router.push(`/withdraw?account=${this.$route.query.account}`)
                }, 500)
              }).catch((req) => {
                let data = JSON.parse(req)
                _this.btnTitle = _this.$t('withdraw.next')
                _this.loading = false
                _this.show = true
                _this.isSuccess = 'warn'
                _this.error = data.error.details[0].message
                _this.ToAccount = ''
                _this.amount = ''
                _this.memo = ''
              })
            })
          } catch (error) {
            _this.loading = false
            _this.show = true
            _this.isSuccess = 'warn'
            _this.error = _this.$t('withdraw.tip')
            _this.amount = ''
            _this.memo = ''
          }
        } catch (error) {
          _this.loading = false
          _this.show = true
          _this.isSuccess = 'warn'
          _this.error = _this.$t('unlock.error.invalid_password')
          _this.btnTitle = _this.$t('withdraw.next')
        }
      } else {
        this.loading = false
        this.show = true
        this.isSuccess = 'warn'
        this.error = this.$t('withdraw.tip')
        this.amount = ''
        this.memo = ''
      }
    },
    next () {
      this.isUnlock = true
    }
  }
}
</script>
<style lang="less" scoped>

</style>
