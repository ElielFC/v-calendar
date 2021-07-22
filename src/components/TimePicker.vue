<template>
  <div
    class="vc-time-picker"
    :class="[{ 'vc-disabled': isDisabled, 'vc-bordered': showBorder }]"
  >
    <div>
      <svg
        fill="none"
        stroke-linecap="round"
        stroke-linejoin="round"
        stroke-width="2"
        viewBox="0 0 24 24"
        class="vc-time-icon"
        stroke="currentColor"
      >
        <path d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
      </svg>
    </div>
    <div class="vc-date-time">
      <div v-if="date" class="vc-date">
        <span class="vc-weekday">
          {{ locale.format(date, 'WWW') }}
        </span>
        <span class="vc-month">
          {{ locale.format(date, 'MMM') }}
        </span>
        <span class="vc-day">
          {{ locale.format(date, 'D') }}
        </span>
        <span class="vc-year">
          {{ locale.format(date, 'YYYY') }}
        </span>
      </div>
      <div class="vc-time">
        <time-select v-model.number="hours" :options="hourOptions" />
        <span style="margin: 0 4px;">:</span>
        <time-select v-model.number="minutes" :options="minuteOptions" :disabled="!isValidHour" />
        <div
          v-if="!is24hr"
          class="vc-am-pm"
          :class="{ 'vc-disabled': !(hours >= 0) }"
        >
          <button
            :class="{ active: isAM }"
            @click.prevent="isAM = true"
            type="button"
            :disabled="!selectAm"
          >
            AM
          </button>
          <button
            :class="{ active: !isAM }"
            @click.prevent="isAM = false"
            type="button"
            :disabled="!selectPm"
          >
            PM
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import TimeSelect from './TimeSelect';
import { pad } from '../utils/helpers';

export default {
  name: 'TimePicker',
  components: { TimeSelect },
  props: {
    value: { type: Object, required: true },
    locale: { type: Object, required: true },
    theme: { type: Object, required: true },
    is24hr: { type: Boolean, default: true },
    minuteIncrement: { type: Number, default: 1 },
    showBorder: Boolean,
    isDisabled: Boolean,
    minTime: { type: String, default: null },
    maxTime: { type: String, default: null },
  },
  data() {
    return {
      hours: 0,
      minutes: 0,
      isAM: true,
    };
  },
  computed: {
    date() {
      let date = this.locale.normalizeDate(this.value);
      if (this.value.hours === 24) {
        date = new Date(date.getTime() - 1);
      }
      return date;
    },
    maxHour() {
      return this.locale.parse(this.maxTime || '23:59', 'HH:mm').getHours();
    },
    minHour() {
      return this.locale.parse(this.minTime || '00:00', 'HH:mm').getHours();
    },
    isValidHour() {
      return this.hours >= this.minHourByFormat() && this.hours <= this.maxHourByFormat();
    },
    selectAm() {
      return this.minHour <= 12;
    },
    selectPm() {
      return this.maxHour >= 12;
    },
    maxMinute() {
      return this.locale.parse(this.maxTime || '23:59', 'HH:mm').getMinutes();
    },
    minMinute() {
      return this.locale.parse(this.minTime || '00:00', 'HH:mm').getMinutes();
    },
    hourOptions() {
      const options = [];
      const minHour = this.minHourByFormat();
      const maxHour = this.maxHourByFormat();

      if (this.hours < minHour) {
        options.push({
          value: this.hours,
          label: pad(this.hours === 0 && !this.is24hr ? 12 : this.hours, 2),
          disabled: true,
        });
      }

      for (let i = minHour; i <= maxHour; i++) {
        options.push({
          value: i,
          label: pad(i === 0 && !this.is24hr ? 12 : i, 2, 0)
        });
      }

      if (this.hours > maxHour) {
        options.push({
          value: this.hours,
          label: pad(this.hours === 0 && !this.is24hr ? 12 : this.hours, 2),
          disabled: true,
        });
      }

      return options;
    },
    minuteOptions() {
      const options = [];
      let m = 0;
      let added = false;
      let minutes = 59;

      if (this.hours === this.minHourByFormat()) {
        m = this.minMinute;
        if (m > this.minutes) {
          added = true;
          options.push({
            value: this.minutes,
            label: pad(this.minutes, 2),
            disabled: true,
          });
        }
      } else if (this.hours === this.maxHourByFormat()) {
        minutes = this.maxMinute;
      }

      while (m <= minutes) {
        options.push({
          value: m,
          label: pad(m, 2),
        });
        added = added || m === this.minutes;
        m += this.minuteIncrement;
        // Add disabled option if interval has skipped it
        if (!added && m > this.minutes) {
          added = true;
          options.push({
            value: this.minutes,
            label: pad(this.minutes, 2),
            disabled: true,
          });
        }
      }

      if (this.hours === this.maxHourByFormat()) {
        if (minutes < this.minutes) {
          options.push({
            value: this.minutes,
            label: pad(this.minutes, 2),
            disabled: true,
          });
        }
      }

      return options;
    },
  },
  watch: {
    value() {
      this.setup();
    },
    hours() {
      this.updateValue();
    },
    minutes() {
      this.updateValue();
    },
    isAM() {
      this.updateValue();
    },
  },
  created() {
    this.setup();
  },
  methods: {
    protected(fn) {
      if (this.busy) return;
      this.busy = true;
      fn();
      this.$nextTick(() => (this.busy = false));
    },
    setup() {
      this.protected(() => {
        let { hours } = this.value;
        if (hours === 24) hours = 0;
        let isAM = true;
        if (!this.is24hr && hours >= 12) {
          hours -= 12;
          isAM = false;
        }
        this.hours = hours;
        this.minutes = this.value.minutes;
        this.isAM = isAM;
      });
    },
    updateValue() {
      this.protected(() => {
        let hours = this.hours;
        if (!this.is24hr && !this.isAM) {
          hours += 12;
        }
        this.$emit('input', {
          ...this.value,
          hours,
          minutes: this.minutes,
          seconds: 0,
          milliseconds: 0,
        });
      });
    },
    maxHourByFormat() {
      if (this.is24hr) {
        return this.maxHour;
      }
      if (this.isAM) {
        return this.maxHour >= 12 ? 11 : this.maxHour;
      }
      return this.maxHour >= 12 ? this.maxHour - 12 : 0;
    },
    minHourByFormat() {
      if (this.is24hr) {
        return this.minHour;
      }
      if (this.isAM) {
        return this.minHour >= 12 ? 0 : this.minHour;
      }
      return this.minHour >= 12 ? this.minHour - 12 : 0;
    }
  },
};
</script>

<style lang="postcss" scoped>
.vc-time-picker {
  display: flex;
  align-items: center;
  padding: 8px;
  &.vc-invalid {
    pointer-events: none;
    opacity: 0.5;
  }
  &.vc-bordered {
    border-top: 1px solid var(--gray-400);
  }
}

.vc-date-time {
  margin-left: 8px;
}

.vc-disabled {
  pointer-events: none;
  opacity: 0.5;
}

.vc-time-icon {
  width: 16px;
  height: 16px;
  color: var(--gray-600);
}

.vc-date {
  display: flex;
  align-items: center;
  font-size: var(--text-sm);
  font-weight: var(--font-semibold);
  text-transform: uppercase;
  padding: 0 0 4px 4px;
  margin-top: -4px;
  & .vc-weekday {
    color: var(--gray-700);
    letter-spacing: var(--tracking-wide);
  }
  & .vc-month {
    color: var(--accent-600);
    margin-left: 8px;
  }
  & .vc-day {
    color: var(--accent-600);
    margin-left: 4px;
  }
  & .vc-year {
    color: var(--gray-500);
    margin-left: 8px;
  }
}

.vc-time {
  display: flex;
  align-items: center;
}

.vc-am-pm {
  display: flex;
  align-items: center;
  background: var(--gray-200);
  margin-left: 8px;
  padding: 4px;
  border-radius: var(--rounded);
  height: 30px;
  & button {
    color: var(--gray-900);
    font-size: var(--text-sm);
    font-weight: var(--font-medium);
    padding: 0 4px;
    background: transparent;
    border: 2px solid transparent;
    border-radius: var(--rounded);
    line-height: var(--leading-snug);
    &:hover {
      color: var(--gray-600);
    }
    &:focus {
      border-color: var(--accent-400);
    }
    &.active {
      background: var(--accent-600);
      color: var(--white);
      &:hover {
        background: var(--accent-500);
      }
      &:focus {
        border-color: var(--accent-400);
      }
    }
  }
}

.vc-is-dark {
  & .vc-time-picker {
    border-color: var(--gray-700);
  }
  & .vc-time-icon {
    color: var(--gray-400);
  }
  & .vc-weekday {
    color: var(--gray-400);
  }
  & .vc-month {
    color: var(--accent-400);
  }
  & .vc-day {
    color: var(--accent-400);
  }
  & .vc-year {
    color: var(--gray-500);
  }
  & .vc-am-pm {
    background: var(--gray-700);
    &:focus {
      border-color: var(--accent-500);
    }
    & button {
      color: var(--gray-100);
      &:hover {
        color: var(--gray-400);
      }
      &:focus {
        border-color: var(--accent-500);
      }
      &.active {
        background: var(--accent-500);
        color: var(--white);
        &:hover {
          background: var(--accent-600);
        }
        &:focus {
          border-color: var(--accent-500);
        }
      }
    }
  }
}
</style>
