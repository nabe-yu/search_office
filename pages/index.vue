/* eslint-disable no-console */
<template>
  <section class="section">
    <div class="content box">
      <h2 class="title is-5">
        ファイルをアップロードしましょう
      </h2>
      <p class="subtitle is-6">
        JSON形式に対応しています。
      </p>
      <b-field class="file is-primary" :class="{ 'has-name': !!staff_data_file }">
        <b-upload v-model="staff_data_file" class="file-label">
          <span class="file-cta">
            <b-icon class="file-icon" icon="upload" />
            <span class="file-label">個人情報</span>
          </span>
          <span v-if="staff_data_file" class="file-name">
            {{ staff_data_file.name }}
          </span>
        </b-upload>
      </b-field>
      <p v-if="staffs.length > 0">
        {{ staffs.length }} 件
      </p>
      <b-field class="file is-primary" :class="{ 'has-name': !!office_data_file }">
        <b-upload v-model="office_data_file" class="file-label">
          <span class="file-cta">
            <b-icon class="file-icon" icon="upload" />
            <span class="file-label">オフィス情報</span>
          </span>
          <span v-if="office_data_file" class="file-name">
            {{ office_data_file.name }}
          </span>
        </b-upload>
      </b-field>
      <p v-if="offices.length > 0">
        {{ offices.length }} 件
      </p>
    </div>
    <div class="columns">
      <div class="column">
        <div v-if="office_data_file && staff_data_file">
          <b-button class="is-large" type="is-info" expanded outlined @click="upload()">
            ファイルを読み込む
          </b-button>
        </div>
      </div>
      <div class="column">
        <b-button
          v-if="isUpload"
          class="is-large"
          type="is-danger"
          expanded
          outlined
          @click="btn()"
        >
          計算する ( {{ data.length }} 件)
        </b-button>
      </div>
      <div class="column">
        <b-button
          v-if="data.length > 0 && data.length >= staffs.length"
          class="is-large"
          type="is-dark"
          expanded
          outlined
          @click="json2csv(data)"
        >
          結果をCSVで表示する
        </b-button>
      </div>
    </div>
    <div v-if="csvData" class="box" style="white-space:pre-wrap;">
      {{ csvData }}
    </div>
    <b-table v-if="data" :data="data" :columns="columns" />
  </section>
</template>

<script>
import distance from '@turf/distance'
import { point } from '@turf/helpers'
export default {
  data() {
    return {
      csvData: null,
      offices: [],
      staffs: [],
      data: [],
      isUpload: '',
      staff_data_file: null,
      office_data_file: null,
      columns: [
        {
          field: 'id',
          label: 'ID'
        },
        {
          field: 'home_address',
          label: '自宅'
        },
        {
          field: 'current_office_name',
          label: '現在の勤務地'
        },
        {
          field: 'current_office_distance',
          label: '現在の勤務地までの距離'
        },
        {
          field: 'satellite_office_name',
          label: '最寄りオフィス'
        },
        {
          field: 'satellite_office_distance',
          label: '最寄りオフィスまでの距離'
        },
        {
          field: 'reduced_distance',
          label: '削減された距離'
        }
      ]
    }
  },
  computed: {},
  methods: {
    json2csv(json) {
      let ret = Object.keys(json[1]).join(',') + '\n'
      ret += json.map(function(d) {
        return Object.keys(d).map(function(key) {
          return d[key]
        }).join(',')
      }).join('\n')
      this.csvData = ret
      return ret
    },
    upload() {
      this.uploadStaffData()
      this.uploadOfficeData()
      this.isUpload = true
    },
    async btn() {
      for (let index = 0; index < this.offices.length; index++) {
        this.offices[index].coordinates = await this.getCoordinates(this.offices[index].office_address)
      }
      this.data = []
      for (const staff of this.staffs) {
        const currentOffice = this.offices.find(office => office.id === staff.office_id)
        const homeCoordinates = await this.getCoordinates(staff.home_address)
        const currentOfficeDistance = this.getDistance(homeCoordinates, currentOffice.coordinates)
        const resltList = []
        for (const office of this.offices) {
          resltList.push({
            name: office.office_name,
            dis: this.getDistance(homeCoordinates, office.coordinates)
          })
        }
        // eslint-disable-next-line array-callback-return
        const res = resltList.sort((a, b) => {
          if (a.dis < b.dis) {
            return -1
          };
        })
        const resobj = {
          id: staff.id,
          home_address: staff.home_address,
          current_office_name: currentOffice.office_name,
          current_office_distance: currentOfficeDistance,
          satellite_office_name: res[0].name,
          satellite_office_distance: res[0].dis,
          reduced_distance: (Math.round((currentOfficeDistance - res[0].dis) * 100)) / 100
        }
        this.data.push(resobj)
      }
    },
    uploadOfficeData() {
      this.offices = ''
      const officeJson = this.office_data_file
      const reader = new FileReader()
      const loadJson = () => {
        this.offices = JSON.parse(reader.result).offices
      }
      reader.onload = loadJson
      return reader.readAsText(officeJson)
    },
    uploadStaffData() {
      this.staffs = ''
      const staffJson = this.staff_data_file
      const reader = new FileReader()
      const loadJson = () => {
        this.staffs = JSON.parse(reader.result).staffs
      }
      reader.onload = loadJson
      return reader.readAsText(staffJson)
    },
    getDistance(point1, point2) {
      const result = distance(point(point1), point(point2), { units: 'kilometers' })
      const roundResult = (Math.round(result * 100)) / 100
      return roundResult
    },
    async getCoordinates(address) {
      const url = `https://msearch.gsi.go.jp/address-search/AddressSearch?q=${address}`
      const response = await this.$axios.$get(url)
      return response[0].geometry.coordinates
    }
  }
}
</script>
