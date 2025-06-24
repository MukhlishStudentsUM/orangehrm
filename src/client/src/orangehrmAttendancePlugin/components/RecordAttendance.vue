<!--
/**
* OrangeHRM is a comprehensive Human Resource Management (HRM) System that captures
* all the essential functionalities required for any enterprise.
* Copyright (C) 2006 OrangeHRM Inc., http://www.orangehrm.com
*
* This file is part of OrangeHRM.
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
    <!-- PREVIOUS PUNCH IN INFORMATION SECTION -->
    <oxd-form-row v-if="attendanceRecord.previousRecord">
      <!-- DIUBAH: Grid utama diubah menjadi 2 kolom agar sejajar seperti form di bawahnya -->
      <oxd-grid :cols="2" class="orangehrm-full-width-grid">
        <!-- KOLOM KIRI: Berisi info Waktu dan Catatan Punch In -->
        <oxd-grid-item>
          <oxd-input-group :label="$t('attendance.punched_in_time')">
            <oxd-text type="subtitle-2">
              {{ previousAttendanceRecordDate }} -
              {{ previousAttendanceRecordTime }}
              <oxd-text
                tag="span"
                class="orangehrm-attendance-punchedIn-timezone"
              >
                {{ `(GMT ${previousRecordTimezone})` }}
              </oxd-text>
            </oxd-text>
          </oxd-input-group>
          <br />
          <!-- Memberi sedikit spasi -->
          <oxd-input-group
            v-if="attendanceRecord.previousRecord.note"
            :label="$t('attendance.punched_in_note')"
          >
            <oxd-text type="subtitle-2">
              {{ attendanceRecord.previousRecord.note }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>

        <!-- KOLOM KANAN: Berisi Peta Lokasi Punch In -->
        <oxd-grid-item>
          <oxd-input-group :label="$t('Punch In Location')">
            <div
              v-if="hasValidPunchInAddress"
              id="punchInMap"
              style="height: 200px; width: 100%"
            ></div>
            <oxd-text
              v-else
              class="orangehrm-map-address orangehrm-map-placeholder"
            >
              Location data not available for this punch in.
            </oxd-text>
            <oxd-text
              v-if="attendanceRecord.previousRecord.address"
              type="subtitle-2"
              class="orangehrm-map-address"
            >
              {{ attendanceRecord.previousRecord.address }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>

    <oxd-divider v-if="attendanceRecord.previousRecord" />

    <!-- MAIN FORM SECTION (Date, Time, Current Location) -->
    <oxd-form-row>
      <oxd-grid :cols="4" class="orangehrm-full-width-grid">
        <oxd-grid-item class="--span-column-2">
          <oxd-grid :cols="2">
            <!-- Date, Time, Note fields... -->
            <oxd-grid-item>
              <date-input
                :key="attendanceRecord.time"
                v-model="attendanceRecord.date"
                :label="$t('general.date')"
                :rules="rules.date"
                :disabled="!isEditable"
                required
              />
            </oxd-grid-item>
            <oxd-grid-item>
              <oxd-input-field
                v-model="attendanceRecord.time"
                :label="$t('general.time')"
                :disabled="!isEditable"
                :rules="rules.time"
                type="time"
                :placeholder="$t('attendance.hh_mm')"
                required
              />
            </oxd-grid-item>
            <oxd-grid-item v-if="isTimezoneEditable" class="--span-column-2">
              <timezone-dropdown v-model="attendanceRecord.timezone" required />
            </oxd-grid-item>
            <oxd-grid-item class="--span-column-2">
              <oxd-input-field
                v-model="attendanceRecord.note"
                :rules="rules.note"
                :label="$t('general.note')"
                :placeholder="$t('general.type_here')"
                type="textarea"
              />
            </oxd-grid-item>
          </oxd-grid>
        </oxd-grid-item>

        <!-- Map for Current Location (Punch Out) -->
        <oxd-grid-item class="--span-column-2">
          <oxd-input-group
            :label="
              !attendanceRecordId
                ? $t('general.location')
                : $t('Punch Out Location (Current)')
            "
          >
            <div id="punchOutMap" style="height: 200px; width: 100%"></div>
            <oxd-text
              v-if="attendanceRecord.address"
              type="subtitle-2"
              class="orangehrm-map-address"
            >
              {{ attendanceRecord.address }}
            </oxd-text>
          </oxd-input-group>
        </oxd-grid-item>
      </oxd-grid>
    </oxd-form-row>

    <oxd-divider />
    <oxd-form-actions>
      <required-text />
      <submit-button
        :label="
          !attendanceRecordId ? $t('attendance.in') : $t('attendance.out')
        "
      />
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
import {promiseDebounce} from '@ohrm/oxd';
import useLocale from '@/core/util/composable/useLocale';
import {APIService} from '@ohrm/core/util/services/api.service';
import useDateFormat from '@/core/util/composable/useDateFormat';
import {reloadPage, navigate} from '@/core/util/helper/navigation';
import TimezoneDropdown from '@/orangehrmAttendancePlugin/components/TimezoneDropdown.vue';

const attendanceRecordModal = {
  date: null,
  time: null,
  note: null,
  timezone: null,
  previousRecord: null,
  address: null,
};

export default {
  name: 'RecordAttendance',
  components: {
    'timezone-dropdown': TimezoneDropdown,
  },
  props: {
    isEditable: {type: Boolean, default: false},
    isTimezoneEditable: {type: Boolean, default: false},
    attendanceRecordId: {type: Number, default: null},
    employeeId: {type: Number, default: null},
    date: {type: String, default: null},
  },
  setup(props) {
    const apiPath = props.employeeId
      ? `/api/v2/attendance/employees/${props.employeeId}/records`
      : '/api/v2/attendance/records';
    const http = new APIService(window.appGlobal.baseUrl, apiPath);
    const {jsDateFormat, userDateFormat, timeFormat, jsTimeFormat} =
      useDateFormat();
    const {locale} = useLocale();
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
      attendanceRecord: {...attendanceRecordModal},
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
      leafletLoaded: false,
      punchInMap: null,
      punchOutMap: null,
    };
  },
  computed: {
    previousAttendanceRecordDate() {
      if (!this.attendanceRecord?.previousRecord) return null;
      return formatDate(
        parseDate(this.attendanceRecord.previousRecord.userDate),
        this.jsDateFormat,
        {locale: this.locale},
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
    // CRITICAL FIX: Computed property to check if the punch-in address is valid
    hasValidPunchInAddress() {
      const address = this.attendanceRecord.previousRecord?.address;
      if (!address) {
        return false;
      }
      // Check for known invalid address strings from the backend or geocoding failures
      const invalidAddresses = [
        'Address not found',
        'Error retrieving address',
        'Map not loaded',
        'Unable to retrieve location',
        'Geolocation not supported',
        'Failed to load map',
      ];
      return !invalidAddresses.includes(address);
    },
  },
  watch: {
    // CRITICAL FIX: Watch the result of the computed property
    hasValidPunchInAddress(isNowValid) {
      if (isNowValid && this.attendanceRecordId) {
        // Wait for the DOM to update with the new v-if condition
        this.$nextTick(() => {
          this.displayPunchInLocation();
        });
      }
    },
  },
  mounted() {
    this.loadLeaflet().then(() => {
      // Always display the current location map on mount
      this.$nextTick(() => {
        this.displayCurrentLocation();
      });
    });
  },
  beforeMount() {
    this.isLoading = true;
    if (this.isTimezoneEditable) {
      const tz = guessTimezone();
      this.attendanceRecord.timezone = {
        id: tz.name,
        label: tz.label,
        _name: tz.name,
        _offset: tz.offset,
      };
    }
    this.setCurrentDateTime()
      .then(() => {
        if (!this.date && !this.isEditable) {
          setClockInterval(this.setCurrentDateTime, 60000);
        }
        let url = '/api/v2/attendance/records/latest';
        if (this.employeeId) {
          url = `/api/v2/attendance/records/latest?empNumber=${this.employeeId}`;
        }
        return this.attendanceRecordId
          ? this.http.request({method: 'GET', url})
          : null;
      })
      .then((response) => {
        if (response) {
          const {data} = response.data;
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
    loadLeaflet() {
      return new Promise((resolve, reject) => {
        if (window.L) {
          this.leafletLoaded = true;
          resolve();
          return;
        }
        const script = document.createElement('script');
        script.src = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js';
        script.async = true;
        script.onload = () => {
          this.leafletLoaded = true;
          resolve();
        };
        script.onerror = () => {
          console.error('Failed to load Leaflet.js');
          this.attendanceRecord.address = 'Failed to load map';
          reject();
        };
        document.head.appendChild(script);
      });
    },
    displayPunchInLocation() {
      if (this.punchInMap) return; // Don't re-initialize
      if (!this.leafletLoaded) return;

      const punchInAddress = this.attendanceRecord.previousRecord.address;
      console.log('Attempting to geocode Punch In address:', punchInAddress);

      fetch(
        `https://nominatim.openstreetmap.org/search?q=${encodeURIComponent(
          punchInAddress,
        )}&format=json&limit=1`,
      )
        .then((response) => response.json())
        .then((data) => {
          if (data && data.length > 0) {
            const {lat, lon} = data[0];
            const coords = [parseFloat(lat), parseFloat(lon)];

            // Make sure the div exists before trying to create a map
            const mapContainer = document.getElementById('punchInMap');
            if (mapContainer) {
              this.punchInMap = window.L.map('punchInMap').setView(coords, 15);
              window.L.tileLayer(
                'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
              ).addTo(this.punchInMap);
              window.L.marker(coords)
                .addTo(this.punchInMap)
                .bindPopup(`<b>You Punched In Here</b>`)
                .openPopup();
              console.log('Punch In map rendered successfully.');
            } else {
              console.error(
                "Punch In map container 'punchInMap' not found in DOM.",
              );
            }
          } else {
            console.error(
              'Geocoding failed for Punch In: Address not found via API.',
              punchInAddress,
            );
          }
        })
        .catch((error) =>
          console.error('Error during Punch In Geocoding:', error),
        );
    },

    displayCurrentLocation() {
      if (this.punchOutMap) return;
      if (!this.leafletLoaded) return;

      const mapContainer = document.getElementById('punchOutMap');
      if (mapContainer) {
        this.punchOutMap = window.L.map('punchOutMap').setView([0, 0], 2);
        window.L.tileLayer(
          'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        ).addTo(this.punchOutMap);

        if (navigator.geolocation) {
          navigator.geolocation.getCurrentPosition(
            (position) => {
              const {latitude, longitude} = position.coords;
              const coords = [latitude, longitude];
              this.punchOutMap.setView(coords, 13);
              window.L.marker(coords)
                .addTo(this.punchOutMap)
                .bindPopup('<b>Your Current Location</b>')
                .openPopup();
              this.reverseGeocode(latitude, longitude);
            },
            (error) => {
              console.error('Geolocation error:', error);
              // Handle error display
            },
          );
        }
      }
    },
    reverseGeocode(lat, lon) {
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
            address: this.attendanceRecord.address,
            timezoneOffset:
              this.attendanceRecord.timezone?._offset ?? timezone.offset,
            timezoneName: this.attendanceRecord.timezone?.id ?? timezone.name,
          },
        })
        .then(() => this.$toast.saveSuccess())
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
          .request({method: 'GET', url: '/api/v2/attendance/current-datetime'})
          .then((res) => {
            const {utcDate, utcTime} = res.data.data;
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
            url: `/api/v2/attendance/${
              this.attendanceRecordId ? 'punch-out' : 'punch-in'
            }/overlaps`,
            params: {
              date: this.attendanceRecord.date,
              time: this.attendanceRecord.time,
              timezoneOffset:
                this.attendanceRecord.timezone?._offset ?? tzOffset,
              empNumber: this.employeeId,
            },
            validateStatus: (status) =>
              (status >= 200 && status < 300) || status == 400,
          })
          .then((res) => {
            const {data, error} = res.data;
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
/* CSS UNTUK IMPORT LEAFLET */
@import 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';

/* CSS LOKAL UNTUK TAMPILAN */
.orangehrm-map-address {
  margin-top: 8px;
}

.orangehrm-map-placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200px;
  width: 100%;
  border-radius: 8px;
  background-color: #f0f0f0;
  color: #666;
  font-style: italic;
}

/*
 * Z-INDEX FIX V4: Solusi Definitif.
 * Memberikan `position: relative` dan `z-index: 0` pada div container peta.
 * Ini menciptakan "stacking context" baru yang akan "mengurung" semua elemen
 * di dalam peta (termasuk popup dan tombol zoom dengan z-index tinggi)
 * agar tidak menimpa elemen UI utama seperti header.
*/
#punchInMap,
#punchOutMap {
  position: relative;
  z-index: 0;
}
</style>
