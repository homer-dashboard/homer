<template>
  <Generic :item="item" :badges="badges">
    <template v-if="metrics.length" #subtitle>
      <div class="evcc-metrics">
        <span v-for="(metric, index) in metrics" :key="metric.key" :title="metric.label">
          <span v-if="index" class="mx-1"> | </span>
          <i :class="metric.icon"></i> {{ metric.value }}
        </span>
      </div>
    </template>
  </Generic>
</template>

<script>
import service from "@/mixins/service.js";

const BATTERY_LEVELS = ["empty", "quarter", "half", "three-quarters", "full"];

const METRICS = [
  {
    key: "pv",
    path: ".pvPower",
    label: "Solar power",
    icon: "fa-solid fa-solar-panel",
    unit: " W",
  },
  {
    key: "home",
    path: ".homePower",
    label: "Home power",
    icon: "fa-solid fa-house",
    unit: " W",
  },
  {
    key: "grid",
    path: ".grid.power",
    label: "Grid power",
    icon: "fa-solid fa-plug",
    unit: " W",
  },
  {
    key: "soc",
    path: ".battery.soc",
    label: "Battery charge",
    icon: (soc) =>
      `fa-solid fa-battery-${BATTERY_LEVELS[Math.min(Math.round(soc / 25), 4)]}`,
    unit: "%",
  },
  {
    key: "battery",
    path: ".battery.power",
    label: "Battery power",
    icon: "fa-solid fa-car-battery",
    unit: " W",
  },
  {
    key: "co2",
    path: ".tariffCo2",
    label: "Grid CO₂ intensity",
    icon: "fa-solid fa-leaf",
    unit: " g/kWh",
  },
];

const STATE_QUERY = `{${METRICS.map(({ key, path }) => `${key}:(${path}?//null)`).join(",")}}`;

export default {
  name: "Evcc",
  mixins: [service],
  data: () => ({
    state: null,
    serverError: null,
  }),
  computed: {
    metrics() {
      if (this.serverError || !this.state) {
        return [];
      }
      return METRICS.flatMap(({ key, label, icon, unit }) => {
        const value = this.state[key];
        if (!this.isValueShown(key) || typeof value !== "number") {
          return [];
        }
        return {
          key,
          label,
          icon: typeof icon === "function" ? icon(value) : icon,
          value: `${this.createValueString(value, unit)}`,
        };
      });
    },
    badges() {
      return [this.connectionBadge()];
    },
  },
  methods: {
    fetchData: function () {
      return this.load(
        this.fetch(`/api/state?jq=${encodeURIComponent(STATE_QUERY)}`).then(
          (state) => {
            if (!Object.values(state).some((v) => typeof v === "number")) {
              throw new Error("No EVCC power data in the response");
            }
            this.state = state;
          },
        ),
      );
    },
    // Converts the value from W to kW if it is bigger than 500W and adds the unit
    createValueString: function (value, unit) {
      // Remove all leading whitespaces from the unit
      unit = unit.trimStart();

      const kilowattConversionThreshold = Number.isInteger(this.item.kilowattConversionThreshold) ? this.item.kilowattConversionThreshold : 500;

      if (unit === "W" && Math.abs(value) >= kilowattConversionThreshold) {
        // Convert value to kW
        value = value / 1000;

        // Round value
        value = Math.round(value * 10) / 10;

        unit = "k" + unit;
      } else {
        value = Math.round(value);
      }
      return value + " " + unit;
    }
  },
};
</script>

<style scoped>
.evcc-metrics {
  display: flex;
  flex-wrap: wrap;
  white-space: normal;
  overflow: visible;
}

.evcc-metrics>span {
  white-space: nowrap;
}
</style>
