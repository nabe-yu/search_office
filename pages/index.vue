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
            {{ staff_data_file.name }} ({{ staffs.length }})
          </span>
        </b-upload>
      </b-field>
      <b-field class="file is-primary" :class="{ 'has-name': !!office_data_file }">
        <b-upload v-model="office_data_file" class="file-label">
          <span class="file-cta">
            <b-icon class="file-icon" icon="upload" />
            <span class="file-label">オフィス情報</span>
          </span>
          <span v-if="office_data_file" class="file-name">
            {{ office_data_file.name }} ({{ offices.length }})
          </span>
        </b-upload>
      </b-field>
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
          計算する
        </b-button>
      </div>
      <div class="column">
        <b-button
          v-if="data.length > 0 && data.length >= staffs.length"
          class="is-large"
          type="is-dark"
          expanded
          outlined
          @click="downloadCsv()"
        >
          結果をCSVで保存する
        </b-button>
      </div>
    </div>
    <!-- <b-table v-if="data.length === staffs.length" :data="data" :columns="columns" /> -->
    <b-progress
      v-if="isUpload"
      type="is-info"
      show-value
      size="is-large"
      :value="(data.length / staffs.length) * 100"
      :rounded="false"
      format="percent"
    />
  </section>
</template>

<script>
import distance from '@turf/distance'
import { point } from '@turf/helpers'
export default {
  data() {
    return {
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
          field: 'current_office_address',
          label: '現在の勤務地'
        },
        {
          field: 'satellite_office_name',
          label: '最寄りオフィス'
        },
        {
          field: 'satellite_office_distance',
          label: '最寄りオフィスまでの距離'
        }
      ]
    }
  },
  computed: {},
  methods: {
    downloadCsv() {
      let csv = 'ID,自宅住所,現在の勤務地,最寄りオフィス,自宅から最寄りオフィスまでの距離\n'
      this.data.forEach((el) => {
        const line = `${el.id},${el.home_address},${el.current_office_address},${el.satellite_office_name},${el.satellite_office_distance}\n`
        csv += line
      })
      const blob = new Blob([csv], { type: 'text/csv' })
      const link = document.createElement('a')
      link.href = window.URL.createObjectURL(blob)
      link.download = 'Result.csv'
      link.click()
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
        const homeCoordinates = await this.getCoordinates(staff.home_address)
        if (homeCoordinates.length < 2) {
          const resobj = {
            id: staff.id,
            home_address: staff.home_address,
            current_office_address: staff.current_office_address,
            satellite_office_name: '計算不能',
            satellite_office_distance: -1
          }
          this.data.push(resobj)
          continue
        }
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
          current_office_address: staff.current_office_address,
          satellite_office_name: res[0].name,
          satellite_office_distance: res[0].dis
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
      if (response.length > 0) {
        return response[0].geometry.coordinates
      } else {
        return []
      }
    }
  }
}
</script>
