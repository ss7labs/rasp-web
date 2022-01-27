<template>
  <div id="app">
    <div class="container">
      <ValidationObserver v-slot="{ valid }">
      <form class="form-inline bottom-buffer" @submit.prevent="searchPhone">
        <div class="form-group mr-2">
          <label class="sr-only " for="inputPhone">Email</label>
          <ValidationProvider :rules="{ required: true, integer:true, regex: /^\d{6}$/  }">
          <input v-model.number="numa" type="text" class="form-control" id="inputPhone" placeholder="Telephone">
          </ValidationProvider>
        </div>

        <b-input-group class="mb-2 mr-sm-2 mb-sm-0">
          <b-form-datepicker id="datepicker-start" v-model="startDate" placeholder="Start date"></b-form-datepicker>
        </b-input-group>
        <b-input-group class="mb-2 mr-sm-2 mb-sm-0">
          <b-form-datepicker id="datepicker-end" v-model="endDate" placeholder="End date"></b-form-datepicker>
        </b-input-group>
        <button type="submit" class="btn btn-primary" :disabled="!valid">Search</button>
        <button v-if="finded" @click="print" type="button" class="btn ml-2 btn-success">Print</button>
      </form>
      </ValidationObserver>

      <div id="detSpacer" v-if="finded"><p class="font-weight-bold">Детализация звонков для:</p></div>

      <div id="details" v-if="finded">
        <div  id="hd1" class="row  text-left">
          <div class="col-1">Абонент:</div>
          <div class="col-2">{{this.numa}}</div>
          <div class="col">{{name}}</div>
          <div class="col-3 text-right">С {{ startDate }} До {{ endDate}}</div>
        </div>
        <div id="hd2" class="row  text-left">
          <div  class="col-1">Адрес:</div>
          <div  class="col">{{ street }} {{settl}}</div>
          <div  class="col-2">дом {{building}}</div>
          <div  class="col">кв. {{flat}}</div>
        </div>
        <table class="table table-sm table-bordered">
          <tbody>
          <tr v-if="callsView">
            <td>Дата</td>
            <td>Направление исх.зв.</td>
            <td>Номер исх.зв.</td>
            <td>Длительность(мин)</td>
            <td>Стоимость</td>
          </tr>
          <tr v-for="(item) in calls" :key="item.id">
            <td>{{item.time}}</td><td>{{item.direction}} <span v-if="item.iskz">(Ком.зал)</span></td><td>{{item.numb}}</td><td>{{item.duration}}</td><td>{{item.cost}}</td>
          </tr>
          <tr v-if="show_exceeded">
            <td colspan="3" class="text-left">Свеpхлимитные переговоры по гоpоду</td>
            <td>{{localMin}}</td>
            <td>{{localCost}}</td>
          </tr>
          <tr v-if="extTotalView">
            <td  colspan="4" class="text-left">Сумма за меж/гор. меж/нар.</td>
            <td>2</td>
          </tr>
          <tr class="text">
            <td colspan="4" class="text-left">Итого к оплате</td>
            <td>{{ totalCost }}</td>
          </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import Util from "@/util";
import { ValidationProvider, ValidationObserver, extend } from 'vee-validate';
import { required, integer, regex } from 'vee-validate/dist/rules';
extend('required', required);
extend('integer', integer);
extend('regex', regex);

export default {
  data() {
    const now = new Date()
    let month = Util.make2digit(now.getMonth()+1)
    let todayStr = now.getFullYear()+"-"+month+"-"+Util.make2digit(now.getDate())
    let firstDay = now.getFullYear()+"-"+month+"-01"
    return {
      name: '',
      street: '',
      settl: '',
      building: '',
      flat: '',
      totalCost: 0,
      localMin: 0,
      localCost: 0,
      callsView: true,
      extTotalView: false,
      show_exceeded: true,
      finded: false,
      startDate: firstDay,
      endDate: todayStr,
      paramStart:'',
      paramEnd:'',
      numa: '',
      calls: [
        {id: 1, date: "21-05-2020 15:16", numa: 490036, numb: 5858861, duration: "3", cost: 0.24},
        {id: 2, date: "21-05-2020 15:23", numa: 490036, numb: 5858861, duration: "1", cost: 0.08},
      ],
    }
  },
  name: 'App',
  components: {
    ValidationProvider,
    ValidationObserver
  },
  methods: {
    print () {
      // Pass the element id here
      this.$htmlToPaper('details', null, () => {
        console.warn('done');
      });
    },
    searchPhone (){
      this.paramStart = this.startDate.replace(/(\d+)-(\d+)-(\d+)/,"$1$2$3");
      this.paramEnd = this.endDate.replace(/(\d+)-(\d+)-(\d+)/,"$1$2$3");
      console.log("start date",this.paramStart);
      console.log("end date",this.paramEnd);
      console.log("numa",this.numa);
      this.loadInfo();
    },
    loadInfo() {
      this.$http.get("/api-kv/"+this.numa+"/"+this.paramStart+"/"+this.paramEnd).then(success => {
        let data = success.body;
        console.log(data);
        let notFound = data.notfound;
        if(notFound || !data.iskv) {
          this.finded=false;
        } else {
          this.finded=true;
        }
        this.calls = data.calls;
        this.localCost = data.locals.total;

        if(data.total < 0.01) {
          this.totalCost = 0;
        } else {
          this.totalCost = data.total;
        }

        this.localMin = data.locals.duration;
        this.street = data.info.street;
        this.settl = data.info.settl;
        this.building = data.info.building;
        this.flat = data.info.flat;
        this.name = data.info.name;
      }, errResp => {
        console.log(errResp);
        this.finded=false;
      });
    },
  }
}
</script>

<style>
.bottom-buffer { margin-botom:120px; }
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
