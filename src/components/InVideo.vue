<template>
  <div>
    <div class="out1">
      <div class="out2">
        <div style="position: relative;">
          <div class="title">カメラ映像</div>
          <video ref="inVideo" autoplay @mousedown="long_touch_start"></video>
        </div>
        <div style="position: relative;">
          <table>
            <tr><th>送信側</th><td></td></tr>
            <tr><th>幅@フレーム数</th><td>{{ in_width }}@{{ in_fps }}fps</td></tr>
            <tr><th>CODEC</th><td>{{ in_codec }}</td></tr>
          </table>
        </div>
      </div>
      <div class="out2">
        <div style="position: relative;">
          <div class="title">エンコード映像</div>
          <video ref="outVideo" autoplay @click="fullscreen($refs.outVideo)"></video>
        </div>
        <div style="position: relative;">
          <table>
            <tr><th>着信側(kbps)</th><td>{{ recv_rate }}kbps</td></tr>
            <tr><th>幅@フレーム数</th><td>{{ out_width }}@{{ out_fps }}fps</td></tr>
            <tr><th>CODEC</th><td>{{ out_codec }}</td></tr>
          </table>
        </div>
      </div>
    </div>
    <select-cam @selected_cam="selected_cam" v-if="showed_selectCam" :x="x" :y="y"></select-cam>
    <div class="param_area">
      <span>カメラパラメータ</span>
      <input type="text" v-model="const_width" placeholder="幅">
      <span>px</span>
      <input type="text" v-model="const_framerate" placeholder="フレーム数">
      <span>fps</span>
      <select v-model="selected_mimeType">
        <option v-for="mimeType_name in mimeTypes" :value="mimeType_name" :key="mimeType_name">
          {{ mimeType_name }}
        </option>
      </select>
      <button @click="restart_video">RESTART</button>
    </div>
    <div class="param_area">
      <span>符号化パラメータ</span>
      <input type="text" v-model="enc_rate" placeholder="指定速度 " @change="enc_change">
      <span>kbps</span>
      <input type="text" v-model="enc_framerate" placeholder="指定FPS " @change="enc_change">
      <span>fps</span>
    </div>
    <!-- <span>Selected: {{ selected_mimeType }}</span> -->
  </div>
</template>

<script>
import SelectCam from "./SelectCam.vue";
export default {
  components: { selectCam: SelectCam },
  data() {
    return {
      pc1: null,
      pc2: null,
      clearId: {}, 
      stream: null,
      const_width: 720,
      const_framerate: 30,
      enc_rate: "",
      enc_framerate: "",
      capabilities: [],
      mimeTypes: [],
      selected_mimeType: "",
      in_width: "",
      in_fps: "",
      in_codec: "",
      out_width: "",
      out_fps: "",
      out_codec: "",
      recv_rate: "",
      showed_selectCam: false,
      x: 0,
      y: 0,
      devId: null,
      long_touch_id: null
    }
  },
  async mounted() {
    this.capabilities = RTCRtpSender.getCapabilities("video");
    this.mimeTypes = ["", ...new Set(this.capabilities.codecs.map(codec => { return codec.mimeType }))];
    await this.start_video();
  },
  methods: {
    long_touch_start(ev) {
      document.addEventListener("mouseup", this.long_touch_stop, false);
      this.x = ev.pageX;
      this.y = ev.pageY;
      this.long_touch_id = setTimeout(() => {
        document.removeEventListener("mouseup", this.long_touch_stop, false);
        this.showed_selectCam = true;
        this.long_touch_id = null;
      }, 1000);
    },
    long_touch_stop() {
      clearTimeout(this.long_touch_id);
      document.removeEventListener("mouseup", this.long_touch_stop, false);
      if (this.long_touch_id ) {
        this.fullscreen(this.$refs.inVideo)
      }
      this.long_touch_id = null;
    },
    // show_selectCam(ev) {
    //   this.x = ev.pageX;
    //   this.y = ev.pageY;
    //   this.showed_selectCam = true;
    // },
    selected_cam(id) {
      console.log(id);
      this.devId = id;
      this.restart_video();
      this.showed_selectCam = false;
    },
    async enc_change() {
      const [sender] = this.pc1.getSenders();
      const params = sender.getParameters();
      if (this.enc_rate != "") {
        params.encodings[0].maxBitrate = this.enc_rate * 1000;
      } else {
        delete params.encodings[0].maxBitrate;
      }
      if (this.enc_framerate != "") {
        params.encodings[0].maxFramerate = this.enc_framerate;
      } else {
        delete params.encodings[0].maxFramerate;
      }
      console.dir(params);
      await sender.setParameters(params);
    },
    async fullscreen(obj) {
      obj.requestFullscreen();
      await obj.play();
    },
    get_cap() {
      const supports = navigator.mediaDevices.getSupportedConstraints();
      console.dir(supports);
    },
    async get_stats (obj, stats_types, res_stats) {
      const _stats = await obj.getStats();
      Object.keys(stats_types).forEach(type => {
        const report_names = stats_types[type];
        res_stats[type] = {};
        _stats.forEach(report => {
          report_names.forEach(report_name => {
            if (report_name in report && report.type == type) {
              if (!res_stats[type][report_name]) {
                res_stats[type].timestamp = report.timestamp;
                res_stats[type][report_name] = report[report_name];
              } else if (res_stats[type][report_name] < report[report_name]) {
                res_stats[type][report_name] = report[report_name];
              }
            }
          });
        });
      });
    },
    async recv_stats() {
      const [recv] = this.pc2.getReceivers();
      this.clearId["recv"] = setInterval(async () => {
        const out_types = {
          "codec": ["codecType", "mimeType"],
          "inbound-rtp": ["frameWidth", "framesPerSecond", "headerBytesReceived", "bytesReceived"]
        };
        const res_out_stats = {}
        await this.get_stats(recv, out_types, res_out_stats);
        this.out_codec = res_out_stats.codec.mimeType;
        // FPS
        this.out_fps = res_out_stats["inbound-rtp"].framesPerSecond;
        // WIDTH
        this.out_width = res_out_stats["inbound-rtp"].frameWidth;
        // Rate(Mbps)
        const header_bytes = res_out_stats["inbound-rtp"].headerBytesReceived;
        const recv_bytes = res_out_stats["inbound-rtp"].bytesReceived;
        const timestamp = res_out_stats["inbound-rtp"].timestamp;
        if (this.last_timestamp) {
          const _rate = (header_bytes - this.last_header_bytes + recv_bytes - this.last_recv_bytes) * 8.0 / (timestamp - this.last_timestamp);
          this.recv_rate = _rate.toLocaleString(undefined, { maximumFractionDigits: 2 })
        }
        this.last_header_bytes = header_bytes;
        this.last_recv_bytes = recv_bytes
        this.last_timestamp = timestamp; 
      }, 500)
    },
    async restart_video() {
      this.pc1.close();
      this.pc2.close();
      Object.keys(this.clearId).forEach(id => {
        clearInterval(this.clearId[id]);
      });
      this.last_timestamp = null;
      this.in_width = "";
      this.in_fps = "";
      this.in_codec = "";
      this.out_width = "";
      this.out_fps = "";
      this.out_codec = "";
      this.recv_rate = "";
      this.enc_framerate = "";
      this.enc_rate = "";

      this.const_width = this.const_width == "" ? 720 : this.const_width;
      this.const_framerate = this.const_framerate == "" ? 30 : this.const_framerate;    

      let codecs;
      if (this.selected_mimeType == "") {
        codecs = this.capabilities.codecs;
      } else {
        codecs = this.capabilities.codecs.filter(codec => { return codec.mimeType == this.selected_mimeType })
      }
      await this.start_video(codecs);
    },
    async start_video(codecs) {
      this.get_cap();
      const constrain = {
        video: {
          width: this.const_width,
          frameRate: this.const_framerate
        },
        audio: false
      };
      if (this.devId) {
        constrain.video.deviceId = this.devId;
      } else {
        delete constrain.video.deviceId;
      }
      const stream = await navigator.mediaDevices.getUserMedia(constrain);
      console.log("GET STREAM")
      this.$refs.inVideo.srcObject = stream;

      this.pc1 = new RTCPeerConnection();
      this.pc2 = new RTCPeerConnection();

      this.pc1.onicecandidate = ev => {
        console.dir(ev);
        this.pc2.addIceCandidate(ev.candidate)
      };
      this.pc2.onicecandidate = ev => {
        console.dir(ev);
        this.pc1.addIceCandidate(ev.candidate)
      };
      this.pc2.ontrack = ev => {
        console.log("THIS IS PC2");
        console.dir(ev);
        this.recv_stats();
        this.$refs.outVideo.srcObject = new MediaStream([ev.track])
      };
      // const track = stream.getVideoTracks()[0];
      // console.dir(track);
      // pc1.addTransceiver(track, {
      //   streams: [ stream ],
      //   direction: "sendonly"
      // });
      
      // pc1.addTrack(track, stream);
      // pc2.addTrack(track, stream);

      const track = stream.getTracks()[0];
      const cap = track.getSettings();
      console.dir(cap);
      this.in_width = cap.width;
      this.in_fps = cap.frameRate;
      
      this.pc1.addTrack(track, stream);
      if (codecs) {
        const [tr] = this.pc1.getTransceivers();
        tr.setCodecPreferences(codecs);
      }
      const offer = await this.pc1.createOffer();
      console.dir(offer);
      await this.pc1.setLocalDescription(offer);
      console.log("pc1.setLocalDescription")
      await this.pc2.setRemoteDescription(offer);
      console.log("pc2.setRemoteDescription")
      const answer = await this.pc2.createAnswer();
      console.dir(answer);
      await this.pc2.setLocalDescription(answer);
      await this.pc1.setRemoteDescription(answer);

      const [sender] = this.pc1.getSenders();
      const in_types = {"codec": ["codecType", "mimeType"]};
      const res_in_stats = {};
      this.clearId["send"] = setInterval(() => {
        this.get_stats(sender, in_types, res_in_stats);
        this.in_codec = res_in_stats.codec && res_in_stats.codec.mimeType;
      }, 500);
    }
  },
  beforeDestroy() {
    Object.keys(this.clearId).forEach(id => {
      clearInterval(this.clearId[id]);
    })
  }
}
</script>

<style>
.out1 {
  display: flex;
}

.out2 {
  width: 50%;
  position: relative;
}
.title {
  top: 0px;
  left: 0px;
  position: absolute;
  color: white;
  background-color: black;
}
video {
  width: 100%;
}

input[type="text"],
select,
textarea {
  resize: none;
  padding: 10px 0;
  /* padding: 0.1em; */
  outline: none;
  border: 1px solid #DDD;
  -webkit-border-radius: 3px;
  -moz-border-radius: 3px;
  border-radius: 3px;
  font-size: 18px;
  width: 120px;
}
input[type="text"]:focus,
textarea:focus {
  box-shadow: 0 0 7px #3498db;
  /* border: 1px solid #3498db; */
  border-bottom: 2px solid #3498db;
}
span,
input[type="text"] {
  /* font: 15px/24px sans-serif; */
	box-sizing: border-box;
	letter-spacing: 1px;
  padding: 10px 0;
	/* padding: 0.3em; */
	border: 0;
}

input[type="text"] {
  text-align: right;
	padding-left: 40px;
  border-bottom: 1px solid #1b2538;
}
span {
  padding: 10px 20px 10px 0;
}

.param_area {
  display: flex;
  padding: 10px 0;
}

table{
  width: 100%;
  border-spacing: 0;
  /* position: absolute; */
  top: 0px;
  left: 0px;
  width: 100%;
}

table th{
  border-bottom: solid 2px #fb5144;
  padding: 10px 0;
}

table td{
  border-bottom: solid 2px #ddd;
  text-align: center;
  padding: 10px 0;
}
</style>