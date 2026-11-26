<template>
  <Generic
    :item="item"
    :subtitle="lastScan"
    :status="status"
    :badges="badges"
  />
</template>

<script>
import service from "@/mixins/service.js";

const hex = (text) =>
  Array.from(new TextEncoder().encode(text), (byte) =>
    byte.toString(16).padStart(2, "0"),
  ).join("");

export default {
  name: "Navidrome",
  mixins: [service],
  data: () => ({
    tracks: null,
    scan: null,
    serverError: null,
  }),
  computed: {
    status() {
      if (this.serverError || !this.scan?.scanning) {
        return this.reachabilityStatus();
      }
      return { state: "busy", label: "scanning" };
    },
    lastScan() {
      const time = Date.parse(this.scan?.lastScan);
      if (this.serverError || !(time > 0)) {
        return null;
      }
      const date = new Date(time);
      const when = date.toLocaleString(undefined, {
        year:
          date.getFullYear() === new Date().getFullYear()
            ? undefined
            : "numeric",
        day: "2-digit",
        month: "2-digit",
        hour: "2-digit",
        minute: "2-digit",
      });
      return `Last scan: ${when}`;
    },
    badges() {
      return [
        {
          key: "tracks",
          label: "Tracks",
          value: this.serverError ? null : this.tracks,
          tone: "info",
          cap: false,
        },
      ];
    },
  },
  methods: {
    fetchData: function () {
      if (!this.requireConfig("username", "password")) {
        return;
      }
      const query = new URLSearchParams({
        u: this.item.username,
        p: `enc:${hex(this.item.password)}`,
        v: "1.16.1",
        c: "homer",
        f: "json",
      });
      return this.load(
        this.fetch(`/rest/getScanStatus?${query}`).then((body) => {
          const response = body?.["subsonic-response"];
          if (response?.status !== "ok") {
            throw new Error(
              response?.error?.message ?? "Unexpected answer from Navidrome",
            );
          }
          this.scan = response.scanStatus ?? null;
          if (this.scan && !this.scan.scanning) {
            this.tracks = this.scan.count;
          }
        }),
      );
    },
  },
};
</script>
