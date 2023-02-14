<template>
    <b-card style="max-width: 35rem" class="mb-2">
        <!-- <img src="../assets/logoVC.png" alt="Logo" class="card-img-top"> -->
        <template #header>
            <h4 class="my-2 text-center">Validator Claims Check</h4>
        </template>
        <b-card-text>
            Just enter your public address and click Submit. The app will check your
            transactions and display the results. If the amount received does not
            match the amount it should be (based on the days lapsed from the last
            claim) it will display the days missing (or extra in case you were
            compensated).
            <br /><br />
            Your theoretical rewards are also calculated, but this assumes your first
            claim was correct, and therefore it marks the first day your validator
            started producing. If you're unlucky and it wasn't, sorry, I don't know
            when it was.
            <br />
            <br />
            As a personal recommendation, do not fill a form with every single error.
            Wait around a month and then make your request. It will facilitate the
            work of devs and we'll probaly get faster responses.
        </b-card-text>
        <b-link href="https://forms.gle/GPrVobR7jMv5dQGn6" class="card-link">Link to Form</b-link>
        <b-form-input v-model="address" placeholder="Enter your address" class="mt-4"></b-form-input>
        <b-button class="mt-2" v-on:click="submit" variant="primary">Analyze Claims</b-button>
        <br />

        <b-list-group class="mt-2">
            <b-list-group-item v-for="item in redlc" :key="item.dateTime">
                {{ item.dateTime }} -> {{ item.redlc }} {{ item.good }}
            </b-list-group-item>
        </b-list-group>
        
        <div v-if="end" class="mt-2">
            <h3>Total Claimed: {{ total }}</h3>
            <h6>
                Should be: {{ totalRewards }} ({{ differenceInDays }} days)
            </h6>
        </div>

        <Datepicker class="mt-4" v-model="date" range />
        <b-button class="mt-2"
            :to="'https://beta.redlightscan.finance/api/addresses/' + address + '/transactions/export?address=' + address + '&startTime=' + Date.parse(date[0])/1000 + '&endTime=' + Date.parse(date[1])/1000"
            variant="primary">
            Download All Txns
        </b-button>
        <template #footer>
            <em>For any question you can contact me in our Telegram group: Neomorph</em>
        </template>
    </b-card>
</template>

<script>
import Datepicker from '@vuepic/vue-datepicker';
import '@vuepic/vue-datepicker/dist/main.css'

export default {
    components: { Datepicker },
    data() {
        return {
            address: "",
            redlc: [],
            total: 0,
            totalRewards: 0,
            end: false,
            date: ''
        };
    },

    mounted() {
        this.address = localStorage.getItem("address") || "";
        const startDate = new Date('August 1, 2022 00:00:00');
        const endDate = new Date(Date.now());
        this.date = [startDate, endDate];
    },

    computed: {
        differenceInDays() {
            const diff = (this.total - this.totalRewards) / 8
            return diff > 0 ? `+ ${diff}` : diff;
        }
    },

    methods: {
        submit() {
            this.redlc = [];
            this.total = 0;
            this.end = false;
            localStorage.setItem("address", this.address);
            this.getTxData();
        },

        async getTxData() {
            const now = Date.now();
            const response = await fetch(
                `https://beta.redlightscan.finance/api/addresses/${this.address}/transactions/export?startTime=1661990400&endTime=${now}`
            );
            const data = await response.text();
            this.getTx(data);
        },

        async getTx(csv) {
            // console.log(csv)
            let prevTx = "";
            let prevDay = "";
            let firstDay = "";
            let firstClaim = 0;

            const lines = csv.trim().split("\n");
            for (var i = 1; i < lines.length; i = i + 1) {
                const splitLine = lines[i].split(",");
                const method = splitLine[12].trim();
                const tx = splitLine[0];
                const dateTime = splitLine[3];
                const status = splitLine[10]

                // Get internal transactions only from Claim actions
                // Some tx comes repeated multiple times so verify this tx is different
                if ((method === "0xae169a50") & (tx !== prevTx) & (status === 'success')) {
                    const response = await fetch(
                        `https://beta.redlightscan.finance/api/transactions/${tx}/internalTransactions`
                    );
                    const data = await response.json();
                    const value =
                        data.internalTransactionCallList[0].value / 1000000000000000000;
                    const correctAmount = this.getDaysDiff(dateTime, prevDay) * 8;
                    const errorDays = (value - correctAmount) / 8;
                    const errorText =
                        errorDays < 0
                            ? `${Math.abs(errorDays)} days missing`
                            : `${Math.abs(errorDays)} days extra`;
                    const obj = {
                        dateTime: dateTime,
                        redlc: value,
                        good:
                            correctAmount === value || prevDay === ""
                                ? ""
                                : `  -  NO OK (${errorText})`,
                    };
                    this.redlc.push(obj);
                    this.total = this.total + value;

                    // Save first claim and value
                    if (firstDay === "") {
                        firstDay = dateTime;
                        firstClaim = value;
                    }

                    prevDay = dateTime;
                }
                // Save this transaction id to verify next one is different
                prevTx = tx;
            }
            this.end = true;
            this.totalRewards = this.getTotalRewards(firstDay, firstClaim, prevDay);
        },

        getDaysDiff(_today, _prevDay) {
            const day = _today.split(" ")[0];
            const prevDay = _prevDay.split(" ")[0];
            const days = (Date.parse(day) - Date.parse(prevDay)) / (1000 * 3600 * 24);
            // console.log(days);
            return days;
        },

        getTotalRewards(firstClaimDate, firstClaimValue, lastClaimDate) {
            /*  Get the total amount of rewards it should have */
            const _firstClaimDate = firstClaimDate.split(" ")[0];
            const _lastClaimDate = lastClaimDate.split(" ")[0];
            const days =
                (Date.parse(_lastClaimDate) - Date.parse(_firstClaimDate)) /
                (1000 * 3600 * 24);
            return days * 8 + firstClaimValue;
        },
    },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
ul {
    text-align: left;
    color: #2c3e50;
}

li {
    margin: 0 10px;
}

.downloadLink {
    color: var(--bs-link-color);
    text-decoration: underline;
    cursor: pointer;
}
</style>