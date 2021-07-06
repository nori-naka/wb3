<template>
  <div class="select_cam" ref="select_cam" >
    <select v-model="selected_cam" @change="selected">
      <option v-for="devId in devIds" :value="devId.deviceId" :key="devId.deviceId" class="options">
        {{ devId.label }}
      </option>
    </select>    
  </div>

</template>

<script>
export default {
  props: ["x", "y"],
  data() {
    return {
      devIds: "",
      selected_cam: ""
    }
  },
  async mounted() {
    this.$refs.select_cam.style.top = this.y + "px";
    this.$refs.select_cam.style.left = this.x + "px";
    const _devs = await navigator.mediaDevices.enumerateDevices();
    this.devIds = _devs.filter(dev => { return dev.kind == "videoinput" });
    // this.$refs.select_cam.onchange = () => { alert(this.selected_cam)};
  },
  methods: {
    selected() {
      this.$emit("selected_cam", this.selected_cam);
    }
  }
}
</script>

<style>
.select_cam {
  position: absolute;
}
.options {
  padding: 10px, 0;
  /* box-shadow: 0 0 7px #3498db;
  border-bottom: 2px solid #3498db; */
}
.options::selection {
  box-shadow: 0 0 7px #3498db;
  border-bottom: 2px solid #3498db;
}
</style>