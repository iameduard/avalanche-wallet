<template>
    <div>
        <div class="cols">
            <form @submit.prevent="">
                <transition-group name="fade" mode="out-in">
                    <div v-show="!isConfirm" key="form">
                        <div style="margin: 30px 0;">
                            <h4>{{$t('earn.validate.label_1')}}</h4>
                            <input type="text" v-model="nodeId" style="width: 100%" label="NodeID-XXXXXXX">
                        </div>
                        <div style="margin: 30px 0;">
                            <h4>{{$t('earn.validate.duration.label')}}</h4>
                            <p class="desc">{{$t('earn.validate.duration.desc')}}</p>
                            <div class="dates">
                                <div>
                                    <label>{{$t('earn.validate.duration.start')}}</label>
                                    <datetime v-model="startDate" type="datetime" :min-datetime="startDateMin" :max-datetime="startDateMax"></datetime>
                                </div>
                                <div>
                                    <label>{{$t('earn.validate.duration.end')}} <span @click="maxoutEndDate">Max</span></label>
                                    <datetime v-model="endDate" type="datetime" :min-datetime="endDateMin" :max-datetime="endDateMax"></datetime>
                                </div>
                            </div>
                        </div>
                        <div style="margin: 30px 0;">
                            <h4>{{$t('earn.validate.amount.label')}}</h4>
                            <p class="desc">{{$t('earn.validate.amount.desc')}}</p>
                            <AvaxInput v-model="stakeAmt" :max="maxAmt" class="amt_in"></AvaxInput>
                        </div>
                        <div style="margin: 30px 0;">
                            <h4>{{$t('earn.validate.fee.label')}}</h4>
                            <p class="desc">{{$t('earn.validate.fee.desc')}}</p>
                            <input type="number" :min="minFee" max="100" step="0.01" v-model="delegationFee">
                        </div>
                        <div class="reward_in" style="margin: 30px 0;" :type="rewardDestination">
                            <h4>{{$t('earn.validate.reward.label')}}</h4>
                            <p class="desc">{{$t('earn.validate.reward.desc')}}</p>
                            <v-chip-group mandatory @change="rewardSelect">
                                <v-chip small value="local">{{$t('earn.validate.reward.chip_1')}}</v-chip>
                                <v-chip small value="custom">{{$t('earn.validate.reward.chip_2')}}</v-chip>
                            </v-chip-group>
                            <QrInput style="height: 40px; border-radius: 2px;" v-model="rewardIn" placeholder="Reward Address" class="reward_addr_in"></QrInput>
                        </div>
                    </div>
                    <ConfirmPage key="confirm" v-show="isConfirm" :node-i-d="nodeId" :start="formStart" :end="formEnd" :amount="formAmt" :delegation-fee="delegationFee" :reward-address="rewardIn" :reward-destination="rewardDestination"></ConfirmPage>
                </transition-group>
                <div>
                    <div class="summary" v-if="!isSuccess">
                        <div>
                            <label>{{$t('earn.validate.summary.max_del')}} <Tooltip style="display: inline-block" :text="$t('earn.validate.summary.max_del_tooltip')"><fa icon="question-circle"></fa></Tooltip></label>
                            <p>{{maxDelegationText}} AVAX</p>
                        </div>
                        <div>
                            <label>{{$t('earn.validate.summary.duration')}} *</label>
                            <p>{{durationText}}</p>
                        </div>
                        <div>
                            <label>{{$t('earn.validate.summary.rewards')}}</label>
                            <p>{{estimatedReward}} AVAX</p>
                        </div>
                        <div class="submit_box">
                            <label style="margin: 8px 0 !important;">* {{$t('earn.validate.summary.warn')}}</label>
                            <p class="err">{{err}}</p>
                            <v-btn v-if="!isConfirm" @click="confirm" class="button_secondary" depressed :loading="isLoading" :disabled="!canSubmit" block>{{$t('earn.validate.confirm')}}</v-btn>
                            <template v-else>
                                <v-btn @click="submit" class="button_secondary" depressed :loading="isLoading" block>{{$t('earn.validate.submit')}}</v-btn>
                                <v-btn text @click="cancelConfirm" block style="color:var(--primary-color); margin-top: 20px;">{{$t('earn.validate.cancel')}}</v-btn>
                            </template>
                        </div>
                    </div>
                    <div class="success_cont" v-else>
                        <p class="check"><fa icon="check-circle"></fa></p>
                        <h2>{{$t('earn.validate.success.title')}}</h2>
                        <p>{{$t('earn.validate.success.desc')}}</p>
                        <p class="tx_id">Tx ID: {{txId}}</p>
                        <label>{{$t('earn.validate.success.status')}}</label>
                        <p v-if="!txStatus">Waiting..</p>
                        <p v-else>{{txStatus}}</p>
                    </div>
                </div>
            </form>
        </div>


    </div>
</template>
<script lang="ts">
import "reflect-metadata";
import {Vue, Component, Prop, Watch} from "vue-property-decorator";
//@ts-ignore
import AvaxInput from "@/components/misc/AvaxInput.vue";
import {BN} from "avalanche";
import Big from 'big.js';
//@ts-ignore
import { QrInput } from "@avalabs/vue_components";
import {bintools, pChain} from "@/AVA";
import AvaHdWallet from "@/js/wallets/AvaHdWallet";
import ConfirmPage from "@/components/wallet/earn/Validate/ConfirmPage.vue";
import moment from "moment";
import {bnToBig, calculateStakingReward} from "@/helpers/helper";
import {ONEAVAX} from "avalanche/dist/utils";
import Tooltip from "@/components/misc/Tooltip.vue";

const MIN_MS = 60000;
const HOUR_MS = MIN_MS * 60;
const DAY_MS = HOUR_MS * 24;

@Component({
    name: "add_validator",
    components: {
        Tooltip,
        AvaxInput,
        QrInput,
        ConfirmPage
    }
})
export default class AddValidator extends Vue{
    startDate: string = (new Date()).toISOString();
    endDate: string = "";
    delegationFee: string = '2.0';
    nodeId = "";
    rewardIn: string = "";
    rewardDestination = 'local'; // local || custom
    isLoading = false;
    isConfirm = false;
    err:string = "";
    stakeAmt: BN = new BN(0);

    minFee = 2;

    formNodeId = "";
    formAmt: BN = new BN(0);
    formStart: Date = new Date(this.startDateMin);
    formEnd: Date = new Date(this.endDateMax);
    formFee: number = 0;
    formRewardAddr = "";

    txId = "";
    txStatus: string|null = null;
    isSuccess = false;


    @Watch('delegationFee')
    onFeeChange(val: string){
        let num = parseFloat(val);
        if(num < this.minFee){
            this.delegationFee = this.minFee.toString();
        }else if(num>100){
            this.delegationFee = '100';
        }
    }


    created(){
        this.startDate = this.startDateMin;
        this.endDate = this.endDateMin;
    }


    amount_in(val: BN){
        console.log("Stake val: ",val);
    }

    get rewardAddressLocal(){
        let wallet: AvaHdWallet = this.$store.state.activeWallet;
        return wallet.getPlatformRewardAddress();
    }

    rewardSelect(val: 'local'|'custom'){
        if(val==='local'){
            this.rewardIn = this.rewardAddressLocal;
        }else{
            this.rewardIn = "";
        }
        this.rewardDestination = val;
    }

    // 5 minutes from now
    get startDateMin(){
        let now = Date.now();
        let res = now + (60000 * 5);

        return (new Date(res)).toISOString();
    }

    // 2 weeks
    get startDateMax(){
        let startDate = new Date();
        // add 2 weeks
        let endTime = startDate.getTime() + (60000*60*24*14);
        let endDate = new Date(endTime);
        return endDate.toISOString();
    }

    // Start date + 2 weeks
    get endDateMin(){
        let start = this.startDate;
        let startDate = new Date(start);

        let end = startDate.getTime() + (DAY_MS*14);
        let endDate = new Date(end);
        return endDate.toISOString();
    }

    // Start date + 1 year
    get endDateMax(){
        let start = this.startDate;
        let startDate = new Date(start);

        let end = startDate.getTime() + (60000*60*24*365);
        let endDate = new Date(end);
        return endDate.toISOString();
    }


    get stakeDuration(): number{
        let start = new Date(this.startDate);
        let end = new Date(this.endDate);
        let diff = end.getTime() - start.getTime();
        return diff;
    }

    get durationText(){
        let d = moment.duration(this.stakeDuration, 'milliseconds');
        let days = Math.floor(d.asDays());
        return `${days} days ${d.hours()} hours ${d.minutes()} minutes`;
    }

    // get stakeAmtText(){
    //     let amt = this.stakeAmt;
    //     let big = Big(amt.toString()).div(Math.pow(10,9));
    //     return big.toLocaleString(2);
    // }
    //
    // get dateMax(){
    //     let dateMs = Date.now() + DAY_MS*364;
    //     let date = new Date(dateMs);
    //     console.log(date);
    //     return date.toISOString();
    // }

    get dateMin(){
        let dateMs = Date.now();
        let date = new Date(dateMs);

        return date.toISOString();
    }

    get denomination(){
        return 9;
    }

    get platformUnlocked(): BN{
        return this.$store.getters.walletPlatformBalance;
    }

    get platformLockedStakeable(): BN{
        return this.$store.getters.walletPlatformBalanceLockedStakeable;
    }

    get feeAmt(): BN{
        return pChain.getTxFee();
    }

    get maxAmt(): BN{
        let pAmt = this.platformUnlocked.add(this.platformLockedStakeable);
        // let fee = this.feeAmt;

        // absolute max stake
        let mult = new BN(10).pow(new BN(6+9))
        let absMaxStake = new BN(3).mul(mult);

        // If above stake limit
        if(pAmt.gt(absMaxStake)){
            return absMaxStake;
        }

        // let res = pAmt.sub(fee);
        const ZERO = new BN('0');
        if(pAmt.gt(ZERO)){
            return pAmt;
        }else{
            return ZERO;
        }


    }

    get maxDelegationAmt(): BN{
        let stakeAmt = this.stakeAmt;

        let maxRelative = stakeAmt.mul(new BN(5));

        // absolute max stake
        let mult = new BN(10).pow(new BN(6+9))
        let absMaxStake = new BN(3).mul(mult);


        let res;
        if(maxRelative.lt(absMaxStake)){
            res = maxRelative.sub(stakeAmt);
        }else{
            res = absMaxStake.sub(stakeAmt);
        }

        return BN.max(res,new BN(0));
    }

    get maxDelegationText(){
        return bnToBig(this.maxDelegationAmt,9).toLocaleString(9);
    }


    get estimatedReward(): string{
        let start = new Date(this.startDate);
        let end = new Date(this.endDate);
        let duration = end.getTime() - start.getTime(); // in ms
        // let durationYears = duration / (60000*60*24*365);
        //
        // let inflationRate = this.inflation;
        // let stakeAmt = Big(this.stakeAmt.toString()).div(Math.pow(10,9));

        let currentSupply = this.$store.state.Platform.currentSupply;
        let estimation = calculateStakingReward(this.stakeAmt,duration/1000, currentSupply);
        let res = Big(estimation.toString()).div(Math.pow(10,9));

        // let value = stakeAmt.times( Math.pow(inflationRate, durationYears));
        // value = value.sub(stakeAmt);

        return res.toLocaleString(2);
    }

    updateFormData(){
        this.formNodeId = this.nodeId.trim();
        this.formAmt = this.stakeAmt;
        this.formStart = new Date(this.startDate);
        this.formEnd = new Date(this.endDate);
        this.formRewardAddr = this.rewardIn;
        this.formFee = parseFloat(this.delegationFee);
    }

    confirm(){
        if(!this.formCheck()) return;
        this.updateFormData();
        this.isConfirm = true;
    }
    cancelConfirm(){
        this.isConfirm = false;
    }

    get canSubmit(){
        if(!this.nodeId){
            return false;
        }

        if(this.stakeAmt.isZero()){
            return false;
        }

        if(!this.rewardIn){
            return false;
        }

        return true;
    }

    formCheck(): boolean{
        this.err = "";


        // Reward Address
        if(this.rewardDestination!=='local'){
            let rewardAddr = this.rewardIn;

            try{
                bintools.parseAddress(rewardAddr, 'P')
            }catch(e){
                this.err = this.$t('earn.validate.errs.address') as string;
                return false;
            }
        }

        // Stake amount
        if(this.stakeAmt.lt(this.minStakeAmt)){
            let big = Big(this.minStakeAmt.toString()).div(Math.pow(10,9));
            this.err = this.$t('earn.validate.errs.amount', [big.toLocaleString()]) as string;
            return false;
        }

        return true;
    }

    async submit(){
        if(!this.formCheck()) return;
        let wallet: AvaHdWallet = this.$store.state.activeWallet;

        try{
            this.isLoading = true;
            this.err = "";
            let txId = await wallet.validate(this.formNodeId,this.formAmt,this.formStart,this.formEnd,this.formFee,this.formRewardAddr);
            this.isLoading = false;
            this.onsuccess(txId);
        }catch(err){
            this.isLoading = false;
            this.onerror(err);
        }
    }

    maxoutEndDate(){
        this.endDate = this.endDateMax;
    }

    onsuccess(txId: string){
        this.txId = txId;
        this.isSuccess = true;
        this.$store.dispatch('Notifications/add', {
            type: 'success',
            title: 'Validator Transaction Sent',
            message: 'If accepted, your tokens will be locked to validate the network and earn rewards.'
        });
        this.updateTxStatus(txId);
    }

    async updateTxStatus(txId: string){
        let status = await pChain.getTxStatus(txId);
        if(!status || status==='Processing'){
            setTimeout(() => {
                this.updateTxStatus(txId);
            }, 5000);
        }else{
            this.txStatus = status;
        }
    }

    get minStakeAmt(): BN{
        return this.$store.state.Platform.minStake;
    }

    onerror(err: any){
        let msg:string = err.message;
        console.error(err);

        if(msg.includes('startTime')){
            this.err = this.$t('earn.validate.errs.date') as string;
        }else if(msg.includes('must be at least')){
            let minAmt = this.minStakeAmt;
            let big = Big(minAmt.toString()).div(Math.pow(10,9));
            this.err = this.$t('earn.validate.errs.amount', [big.toLocaleString()]) as string;
        }else if(msg.includes('nodeID')){
            this.err = this.$t('earn.validate.errs.id') as string;
        }else if(msg.includes('address format')){
            this.err = this.$t('earn.validate.errs.address') as string;
        }else{
            this.err = err.message;
        }

        this.$store.dispatch('Notifications/add', {
            type: 'error',
            title: 'Validation Failed',
            message: 'Failed to add validator.'
        })
    }

}
</script>
<style scoped lang="scss">
@use "../../../../main";
.cols{
    /*display: grid;*/
    /*grid-template-columns: 1fr 1fr;*/
}


form{
    display: grid;
    grid-template-columns: 1fr 340px;
    column-gap: 90px;
}

.amt{
    width: 100%;
    display: flex;
    flex-direction: row;
    align-items: center;
    border: 1px solid #999;
    padding: 4px 14px;
}
.bigIn{
    flex-grow: 1;

}

input{
    color: var(--primary-color);
    background-color: var(--bg-light);
    padding: 6px 14px;
}

.desc{
    font-size: 13px;
    margin-bottom: 8px !important;
    color: var(--primary-color-light);
}

h4{
    margin: 14px 0px 4px;
    font-weight: bold;
}

label{
    margin-top: 6px;
    color: var(--primary-color-light);
    font-size: 14px;
    margin-bottom: 3px;
}

.dates{
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-gap: 15px;

    label > span{
        float: right;
        opacity: 0.4;
        cursor: pointer;
        &:hover{
            opacity: 1;
        }
    }
}

.submit_box{
    .v-btn{
        margin-top: 14px;
    }
}

.summary{
    border-left: 2px solid var(--bg-light);
    padding-left: 30px;
    > div{
        margin: 14px 0;
        p{
            font-size: 24px;
        }
    }

    .err{
        margin: 14px 0 !important;
        font-size: 14px;
    }
}

.success_cont{
    .check{
        font-size: 4em;
        color: var(--success);
    }

    .tx_id{
        font-size: 13px;
        color: var(--primary-color-light);
        word-break: break-all;
        margin: 14px 0 !important;
        font-weight: bold;
    }
}

.amt_in{
    width: max-content;
}

.reward_in{
    transition-duration: 0.2s;
    &[type="local"]{
        .reward_addr_in{
            opacity: 0.3;
            user-select: none;
            pointer-events: none;
        }
    }
}

@include main.mobile-device{
    form{
        grid-template-columns: 1fr;
    }

    .dates{
        grid-template-columns: 1fr;
    }

    .amt_in{
        width: 100%;
    }

    .summary{
        border-left: none;
        border-top: 2px solid var(--bg-light);
        padding-left: 0;
        padding-top: 30px;
    }
}
</style>
