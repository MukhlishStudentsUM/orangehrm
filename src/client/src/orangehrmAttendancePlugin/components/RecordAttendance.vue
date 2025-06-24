<!--
/**
 * OrangeHRM is a comprehensive Human Resource Management (HRM) System that captures
 * all the essential functionalities required for any enterprise.
 * Copyright (C) 2006 OrangeHRM Inc., http://www.orangehrm.com
 *
 * OrangeHRM is free software: you can redistribute it and/or modify it under the terms of
 * the GNU General Public License as published by the Free Software Foundation, either
 * version 3 of the License, or (at your option) any later version.
 *
 * OrangeHRM is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY;
 * without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 * See the GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License along with OrangeHRM.
 * If not, see <https://www.gnu.org/licenses/>.
 */
 -->

<template>
  <oxd-form :loading="isLoading" @submit-valid="onSave">
    <oxd-form-row v-if="attendanceRecord.previousRecord">
      <oxd-grid :cols="4" class="orangehrm-full-width-grid">
        <oxd-grid-item :class="!attendanceRecord.previousRecord.note ? '--span-column-2' : ''">
          <oxd-input-group :label="$t('attendance.punched_in_time')">
            <oxd-text type="subtitle-2">
              {{ previousAttendanceRecordDate }} -
              {{ previousAttendanceRecordTime }}
              <oxd-text tag="span" class="orangehrm-attendance-punchedIn-timezone">
                {{ `(GMT ${previousRecordTimezone})` }}
              </oxd-text>
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
        <oxd-grid-item v-if="attendanceRecord.previousRecord.note">
          <oxd-input-group :label="$t('attendance.punched_in_note')">
            <oxd-text type="subtitle-2">
              {{ attendanceRecord.previousRecord.note }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>

    <!-- Date -->
    <oxd-form-row>
      <oxd-grid :cols="4" class="orangehrm-full-width-grid">
        <oxd-grid-item class="--span-column-2">
          <oxd-grid :cols="2">
            <oxd-grid-item>
              <date-input :key="attendanceRecord.time" v-model="attendanceRecord.date" :label="$t('general.date')"
                :rules="rules.date" :disabled="!isEditable" required />
            </oxd-grid-item>

            <!-- Time -->
            <oxd-grid-item>
              <oxd-input-field v-model="attendanceRecord.time" :label="$t('general.time')" :disabled="!isEditable"
                :rules="rules.time" type="time" :placeholder="$t('attendance.hh_mm')" required />
            </oxd-grid-item>

            <oxd-grid-item v-if="isTimezoneEditable" class="--span-column-2">
              <timezone-dropdown v-model="attendanceRecord.timezone" required />
            </oxd-grid-item>

            <!-- Note -->
            <oxd-grid-item class="--span-column-2">
              <oxd-input-field style="height: 200px; width: 100%;" v-model="attendanceRecord.note" :rules="rules.note"
                :label="$t('general.note')" :placeholder="$t('general.type_here')" type="textarea" />
            </oxd-grid-item>
          </oxd-grid>
        </oxd-grid-item>

        <!-- Location -->
        <oxd-grid-item class="--span-column-2">
          <oxd-input-group :label="$t('general.location')">
            <div id="map" style="height: 300px; width: 100%"></div>
            <oxd-text v-if="
              attendanceRecord.address &&
              ![
                'Map not loaded',
                'Unable to retrieve location',
                'Geolocation not supported',
                'Address not found',
                'Error retrieving address',
                'Failed to load map',
              ].includes(attendanceRecord.address)
            " type="subtitle-2" class="orangehrm-map-address">
              {{ attendanceRecord.address }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>

    <oxd-divider />
    <oxd-form-actions>
      <required-text />
      <submit-button :label="!attendanceRecordId ? $t('attendance.in') : $t('attendance.out')
        " />
    </oxd-form-actions>
  </oxd-form>
</template>

<script>
import {
  required,
  validDateFormat,
  shouldNotExceedCharLength,
} from '@/core/util/validation/rules';
import {
  parseTime,
  parseDate,
  formatTime,
  formatDate,
  guessTimezone,
  setClockInterval,
  getStandardTimezone,
} from '@/core/util/helper/datefns';
import { promiseDebounce } from '@ohrm/oxd';
import useLocale from '@/core/util/composable/useLocale';
import { APIService } from '@ohrm/core/util/services/api.service';
import useDateFormat from '@/core/util/composable/useDateFormat';
import { reloadPage, navigate } from '@/core/util/helper/navigation';
import TimezoneDropdown from '@/orangehrmAttendancePlugin/components/TimezoneDropdown.vue';

const attendanceRecordModal = {
  date: null,
  time: null,
  note: null,
  timezone: null,
  previousRecord: null,
  address: null, // PENAMBAHAN: Properti untuk menyimpan alamat lokasi
};

export default {
  name: 'RecordAttendance',
  components: {
    'timezone-dropdown': TimezoneDropdown,
  },
  props: {
    isEditable: {
      type: Boolean,
      default: false,
    },
    isTimezoneEditable: {
      type: Boolean,
      default: false,
    },
    attendanceRecordId: {
      type: Number,
      default: null,
    },
    employeeId: {
      type: Number,
      default: null,
    },
    date: {
      type: String,
      default: null,
    },
  },
  setup(props) {
    const apiPath = props.employeeId
      ? // PERUBAHAN: Menggunakan backtick untuk template literal
      `/api/v2/attendance/employees/${props.employeeId}/records`
      : '/api/v2/attendance/records';
    const http = new APIService(window.appGlobal.baseUrl, apiPath);
    const { jsDateFormat, userDateFormat, timeFormat, jsTimeFormat } =
      useDateFormat();
    const { locale } = useLocale();
    return {
      http,
      locale,
      timeFormat,
      jsTimeFormat,
      jsDateFormat,
      userDateFormat,
    };
  },
  data() {
    return {
      isLoading: false,
      attendanceRecord: { ...attendanceRecordModal },
      rules: {
        date: [
          required,
          validDateFormat(this.userDateFormat),
          promiseDebounce(this.validateDate, 500),
        ],
        time: [required, promiseDebounce(this.validateDate, 500)],
        note: [shouldNotExceedCharLength(250)],
      },
      previousRecordTimezone: null,
      map: null, // PENAMBAHAN: Variabel untuk instance peta Leaflet
      marker: null, // PENAMBAHAN: Variabel untuk marker di peta
      leafletLoaded: false, // PENAMBAHAN: Status apakah Leaflet sudah dimuat
    };
  },
  computed: {
    previousAttendanceRecordDate() {
      if (!this.attendanceRecord?.previousRecord) return null;
      return formatDate(
        parseDate(this.attendanceRecord.previousRecord.userDate),
        this.jsDateFormat,
        { locale: this.locale },
      );
    },
    previousAttendanceRecordTime() {
      if (!this.attendanceRecord?.previousRecord) return null;
      return formatTime(
        parseTime(
          this.attendanceRecord.previousRecord.userTime,
          this.timeFormat,
        ),
        this.jsTimeFormat,
      );
    },
  },
  // PENAMBAHAN: Lifecycle hook mounted untuk memuat Leaflet dan peta
  mounted() {
    this.loadLeaflet();
  },
  beforeMount() {
    this.isLoading = true;
    // set default timezone
    if (this.isTimezoneEditable) {
      const tz = guessTimezone();
      this.attendanceRecord.timezone = {
        id: tz.name,
        label: tz.label,
        _name: tz.name,
        _offset: tz.offset,
      };
    }

    // fetch and set attendance record on initial load
    this.setCurrentDateTime()
      .then(() => {
        // then set record date/time every minute
        !this.date &&
          !this.isEditable &&
          setClockInterval(this.setCurrentDateTime, 60000);
        let url = '/api/v2/attendance/records/latest';
        if (this.employeeId) {
          // PERUBAHAN: Menggunakan backtick untuk template literal
          url = `/api/v2/attendance/records/latest?empNumber=${this.employeeId}`;
        }
        return this.attendanceRecordId
          ? this.http.request({ method: 'GET', url })
          : null;
      })

      .then((response) => {
        if (response) {
          const { data } = response.data;
          this.attendanceRecord.previousRecord = data.punchIn;
        }
      })
      .then(() => {
        this.previousRecordTimezone = getStandardTimezone(
          this.attendanceRecord.previousRecord?.offset,
        );
      })
      .finally(() => {
        this.isLoading = false;
      });
  },
  methods: {
    // PENAMBAHAN: Metode untuk memuat library Leaflet secara dinamis
    loadLeaflet() {
      // Cek apakah Leaflet sudah ada di window
      if (window.L) {
        this.leafletLoaded = true;
        this.initMap();
        this.getUserLocation();
        return;
      }
      // Jika belum, buat elemen script dan muat Leaflet
      const script = document.createElement('script');
      script.src = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js';
      script.async = true;
      script.onload = () => {
        this.leafletLoaded = true;
        this.initMap();
        this.getUserLocation();
      };
      // Tangani error jika gagal memuat Leaflet
      script.onerror = () => {
        console.error('Failed to load Leaflet.js');
        this.attendanceRecord.address = 'Failed to load map';
      };
      document.head.appendChild(script);
    },
    // PENAMBAHAN: Metode untuk menginisialisasi peta
    initMap() {
      // Pastikan Leaflet sudah dimuat sebelum inisialisasi peta
      if (!this.leafletLoaded || !window.L) return;
      // Inisialisasi peta dan atur tampilan awal
      this.map = window.L.map('map').setView([0, 0], 13);
      // Tambahkan tile layer OpenStreetMap
      window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution:
          '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      }).addTo(this.map);
    },
    // PENAMBAHAN: Metode untuk mendapatkan lokasi pengguna
    getUserLocation() {
      if (!this.leafletLoaded || !window.L) {
        this.attendanceRecord.address = 'Map not loaded';
        return;
      }
      // Cek apakah Geolocation API didukung browser
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          (position) => {
            const { latitude, longitude } = position.coords;
            // Atur tampilan peta ke lokasi pengguna
            this.map.setView([latitude, longitude], 13);
            // Tambahkan marker di lokasi pengguna
            this.marker = window.L.marker([latitude, longitude]).addTo(
              this.map,
            ).bindPopup(`<b>You Are Here</b>`).openPopup();
            // Lakukan reverse geocoding untuk mendapatkan alamat
            this.reverseGeocode(latitude, longitude);
          },
          (error) => {
            console.error('Geolocation error:', error);
            this.attendanceRecord.address = 'Unable to retrieve location';
          },
        );
      } else {
        this.attendanceRecord.address = 'Geolocation not supported';
      }
    },
    // PENAMBAHAN: Metode untuk melakukan reverse geocoding (koordinat ke alamat)
    reverseGeocode(lat, lon) {
      // PERUBAHAN: Menggunakan backtick untuk template literal
      fetch(
        `https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lon}`,
      )
        .then((response) => response.json())
        .then((data) => {
          if (data && data.display_name) {
            this.attendanceRecord.address = data.display_name;
          } else {
            this.attendanceRecord.address = 'Address not found';
          }
        })
        .catch((error) => {
          console.error('Reverse geocoding error:', error);
          this.attendanceRecord.address = 'Error retrieving address';
        });
    },
    onSave() {
      this.isLoading = true;
      const timezone = guessTimezone();
      this.http
        .request({
          method: this.attendanceRecordId ? 'PUT' : 'POST',
          data: {
            date: this.attendanceRecord.date,
            time: this.attendanceRecord.time,
            note: this.attendanceRecord.note,
            address: this.attendanceRecord.address, // PENAMBAHAN: Mengirim data alamat ke backend
            timezoneOffset:
              this.attendanceRecord.timezone?._offset ?? timezone.offset,
            timezoneName: this.attendanceRecord.timezone?.id ?? timezone.name,
          },
        })
        .then(() => {
          return this.$toast.saveSuccess();
        })
        .then(() => {
          this.employeeId
            ? navigate('/attendance/viewAttendanceRecord', undefined, {
              employeeId: this.employeeId,
              date: this.date,
            })
            : reloadPage();
        });
    },
    setCurrentDateTime() {
      return new Promise((resolve, reject) => {
        this.http
          .request({ method: 'GET', url: '/api/v2/attendance/current-datetime' })
          .then((res) => {
            const { utcDate, utcTime } = res.data.data;
            // PERUBAHAN: Menggunakan backtick untuk template literal
            const currentDate = parseDate(
              `${utcDate} ${utcTime} +00:00`,
              'yyyy-MM-dd HH:mm xxx',
            );
            this.attendanceRecord.date =
              this.date ?? formatDate(currentDate, 'yyyy-MM-dd');
            this.attendanceRecord.time = formatDate(currentDate, 'HH:mm');
            resolve();
          })
          .catch((error) => reject(error));
      });
    },
    validateDate() {
      if (!this.attendanceRecord.date || !this.attendanceRecord.time) {
        return true;
      }
      if (parseDate(this.attendanceRecord.date) === null) {
        return true;
      }
      const tzOffset = (new Date().getTimezoneOffset() / 60) * -1;
      return new Promise((resolve) => {
        this.http
          .request({
            method: 'GET',
            url: `/api/v2/attendance/${this.attendanceRecordId ? 'punch-out' : 'punch-in'
              }/overlaps`,
            params: {
              date: this.attendanceRecord.date,
              time: this.attendanceRecord.time,
              timezoneOffset:
                this.attendanceRecord.timezone?._offset ?? tzOffset,
              empNumber: this.employeeId,
            },
            // Prevent triggering response interceptor on 400
            validateStatus: (status) => {
              return (status >= 200 && status < 300) || status == 400;
            },
          })
          .then((res) => {
            const { data, error } = res.data;
            if (error) {
              return resolve(error.message);
            }
            return data.valid === true
              ? resolve(true)
              : resolve(this.$t('attendance.overlapping_records_found'));
          });
      });
    },
  },
};
</script>

<style src="./record-attendance.scss" lang="scss" scoped></style>
<style>
@import 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';

.orangehrm-map-address {
  margin-top: 8px;
}
</style>