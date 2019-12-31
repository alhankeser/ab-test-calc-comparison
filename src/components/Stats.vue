<template>
  <div>
    <div class="options">
      Baseline: 
      <select v-model="baselineIndex">
        <option v-for="v in variations" :value="v.id" :key="v.$index">{{v.name}}</option>
      </select>
      Confidence: 
      <select v-model="confidence">
        <option :value="0.99">0.99</option>
        <option :value="0.95">0.95</option>
        <option :value="0.90">0.90</option>
        <option :value="0.85">0.85</option>
        <option :value="0.80">0.80</option>
      </select>
      Hypothesis to Refute: 
      <select v-model="hypothesis">
        <option :value="-1">Variation is Worse than Baseline</option>
        <option :value="0">Variation is Equal to Baseline</option>
      </select>
    </div>
    <table>
       <tr>
         <th>Variation</th>
         <th>Unique Visitors</th>
         <th>Unique Conversions</th>
         <th>Conversion Rate</th>
         <th>Lift</th>
         <th>Min-Max</th>
         <th>Standard Deviation</th>
         <th>Var</th>
         <th>Standard Deviation Mean</th>
         <th>Diff of Means</th>
         <th>Diff Var</th>
         <th>Diff Df</th>
         <th>Diff Mean Std</th>
         <th>pValue</th>
       </tr>
       <tr v-for="v in variations" :key="v.$index" :class="v.status">
         <td>{{v.name}}<div class="bl" v-if="v === baseline">(baseline)</div></td>
         <td><input type="number" v-model="v.visitors" @change="calculate()"></td>
         <td><input type="number" v-model="v.conversions" @change="calculate()"></td>
         <td>{{format_number(v.cr * 100, 2)}} %</td>
         <td>{{format_number((v.cr-baseline.cr)/baseline.cr * 100, 2)}}%</td>
         <td>{{format_number(v.ci[0], 2)}} - {{format_number(v.ci[1], 2)}}</td>
         <td>{{format_number(v.std, 2)}}</td>
         <td>{{format_number(v.vr, 2)}}</td>
         <td>{{format_number(v.mean_std, 2)}}</td>
         <td>{{format_number(v.diff_mean, 2)}}</td>
         <td>{{format_number(v.diff_var, 2)}}</td>
         <td>{{format_number(v.diff_df, 2)}}</td>
         <td>{{format_number(v.diff_mean_std, 2)}}</td>
         <td>{{format_number(v.pval, 2)}}</td>
       </tr>
    </table>
  </div>
</template>

<script>
var { jStat } = require('jstat')

export default {
  name: 'Stats',
  data() {
    return {
      confidence: 0.95,
      baselineIndex: 0,
      hypothesis: 0,
      variations: [
        {id: 0, name: 'Var 1', visitors: 10, conversions: 4, cr: 0, ci:false, std: 0, vr:0, mean_std:0, diff_mean:0, diff_var:0, diff_df:0, diff_mean_std:0, pval: 0, status:false},
        {id: 1, name: 'Var 2', visitors: 10, conversions: 9, cr: 0, ci:false, std: 0, vr:0, mean_std:0, diff_mean:0, diff_var:0, diff_df:0, diff_mean_std:0, pval: 0, status:false},
      ]
    }
  },
  computed: {
    baseline() {
      return this.variations[this.baselineIndex];
    }
  },
  watch: {
    baselineIndex() {
      this.calculate();
    },
    confidence() {
      this.calculate();
    },
    hypothesis() {
      this.calculate();
    },
    variations() {
      this.calculate();
    }
  },
  methods: {
    custom_stdev(visitors, conversions, cr) {
      var result = 0;
      var cr_diff = 1-cr
      var noncr_diff = 0-cr
      result = conversions * (cr_diff * cr_diff) + (visitors-conversions) * (noncr_diff * noncr_diff)
      return Math.sqrt(result / (visitors - 1))
    },
    calculate() {
      this.variations.forEach((v) => {
        v.samples = [].concat(jStat.ones(1,v.conversions)[0], jStat.zeros(1,v.visitors-v.conversions)[0])
        v.cr = v.conversions/v.visitors
        v.std = this.custom_stdev(v.visitors, v.conversions, v.cr)
        v.vr = v.std * v.std
        v.mean_std = Math.sqrt(v.vr / v.visitors)
      }, this)
      this.variations.forEach((v) => {
        v.diff_mean = v.cr - this.baseline.cr
        v.diff_var = this.baseline.vr / this.baseline.visitors + v.vr / v.visitors
        v.diff_mean_std = Math.sqrt(v.diff_var)
        v.diff_df = v.diff_var * v.diff_var / ((v.vr / v.visitors) * (v.vr / v.visitors) / (v.visitors - 1) + (this.baseline.vr / this.baseline.visitors) * (this.baseline.vr / this.baseline.visitors) / (this.baseline.visitors - 1));
        v.cix_1 = this.confidence_interval(v.cr, v.mean_std, 0.999, v.visitors-1);
        v.cix_2 = this.confidence_interval(this.baseline.cr, this.baseline.mean_std, 0.999, this.baseline.visitors-1);
        v.cix_3 = this.confidence_interval(v.cr, v.diff_mean_std, 0.999, v.diff_df);
        v.min = Math.min(v.cix_1[0], v.cix_2[0], v.cix_3[0]);
        v.max = Math.max(v.cix_1[1], v.cix_2[1], v.cix_3[1]);
        v.ci = this.confidence_interval(v.cr, v.mean_std, this.confidence, v.visitors-1);
        v.t = v.diff_mean / v.diff_mean_std;
        v.x = v.diff_df / (v.diff_df + v.t * v.t);
        v.pval = jStat.ibeta(v.x, v.diff_df/2, 0.5);
        if (this.hypothesis != 0) {
            if ((v.t < 0) == (this.hypothesis < 0)) {
                v.pval = 1 - 0.5 * v.pval;
            } else {
                v.pval = 0.5 * v.pval;
            }
        }
        if (v.pval < (1 - this.confidence)) { //&& v.visitors > 100 && v.conversions > 20
            if (v.cr > this.baseline.cr) { //&& (v.ci[0] > this.baseline.ci[1])
              v.status = 'winning'
            }
            // else if (v.ci[1] < this.baseline.ci[0]) {
              
            // }
            else {
              v.status = 'losing'
              // v.status = 'no-diff'
            }
        }
        else {
          v.status = 'no-diff'
        }
            
        
        // window.console.log(v.pval)
      }, this)
    },
    confidence_interval(mean, mean_std, level, nu) {
      var invBeta = jStat.ibetainv(1-level, 0.5 * nu, 0.5);
      var t = Math.sqrt(nu / invBeta - nu);
      return [ mean - t * mean_std, mean + t * mean_std ];
    },
    round_to_places(num, decimals) {
        var extra_places = 0;
        if (Math.abs(num) < 0.1 && num != 0.0) {
            extra_places = Math.abs(1 + Math.floor(Math.log(Math.abs(num)) / Math.LN10));
        }
        if (extra_places > 6) {
            extra_places = 6;
        }
        return (Math.round(num * Math.pow(10, decimals + extra_places)) / 
                Math.pow(10, decimals + extra_places));
    },
    format_number(num, decimals) {
        var sign = "";
        if (num < 0) {
            sign = "−";
        }
        return sign + Math.abs(this.round_to_places(num, decimals));
    }
  },
  created() {
    this.calculate();
  }
}
</script>

<style scoped>

th, td {
  border-right: 1px solid #ccc;
  border-bottom: 1px solid #ccc;
  padding: 6px;
  text-align: center;
}
th:last-child, td:last-child {
  border-right: 0;
}
.bl {
  font-size: 70%;
}
.options {
  margin-bottom: 20px;
}
.winning {
  background-color: greenyellow;
}
.losing {
  background-color: orchid;
}
</style>
