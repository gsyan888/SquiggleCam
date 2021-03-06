<template>
  <video ref="video"
         :width="width"
         :height="height"
         :src="source"
         :autoplay="autoplay"
         :playsinline="playsinline"/>
</template>

<script>
  export default {
    name: "WebCam",
    props: {
      width: {
        type: [Number, String],
        default: "100%"
      },
      height: {
        type: [Number, String],
        default: 200
      },
      autoplay: {
        type: Boolean,
        default: true
      },
      screenshotFormat: {
        type: String,
        default: "image/jpeg"
      },
      deviceId: {
        type: String,
        default: null
      },
      playsinline: {
        type: Boolean,
        default: true
      }
    },
    data() {
      return {
        source: null,
        canvas: null,
        camerasListEmitted: false,
        cameras: []
      };
    },
    watch: {
      deviceId: function(id) {
        this.changeCamera(id);
      }
    },
    mounted() {

      this.setupMedia();
    },
    beforeDestroy() {
      //this.camerasListEmitted = false;
      //this.cameras = [];
      this.stop();
    },
    methods: {
      legacyGetUserMediaSupport() {
        return constraints => {
          // First get a hold of the legacy getUserMedia, if present
          let getUserMedia =
            navigator.getUserMedia ||
            navigator.webkitGetUserMedia ||
            navigator.mozGetUserMedia ||
            navigator.msGetUserMedia ||
            navigator.oGetUserMedia;

          // Some browsers just don't implement it - return a rejected promise with an error
          // to keep a consistent interface
          if (!getUserMedia) {
            return Promise.reject(
              new Error("getUserMedia is not implemented in this browser")
            );
          }

          // Otherwise, wrap the call to the old navigator.getUserMedia with a Promise
          return new Promise(function(resolve, reject) {
            getUserMedia.call(navigator, constraints, resolve, reject);
          });
        };
      },
      setupMedia() {
        if (navigator.mediaDevices === undefined) {
          navigator.mediaDevices = {};
        }

        if (navigator.mediaDevices.getUserMedia === undefined) {
          navigator.mediaDevices.getUserMedia = this.legacyGetUserMediaSupport();
        }

        this.testMediaAccess();
      },
      loadCameras() {
        navigator.mediaDevices
          .enumerateDevices()
          .then(deviceInfos => {
            for (let i = 0; i !== deviceInfos.length; ++i) {
              let deviceInfo = deviceInfos[i];
              if (deviceInfo.kind === "videoinput") {
                this.cameras.push(deviceInfo);
              }
            }
          })
          .then(() => {
            if (!this.camerasListEmitted) {
              this.$emit("cameras", this.cameras);
              this.camerasListEmitted = true;
            }
          })
          .catch(error => this.$emit("notsupported", error));
      },
      /**
       * change to a different camera stream, like front and back camera on phones
       */
      changeCamera(deviceId) {
        this.stop();
        this.$emit("camera-change", deviceId);
        this.loadCamera(deviceId);
      },
      /**
       * load the stream to the
       */
      loadSrcStream(stream) {
        if ("srcObject" in this.$refs.video) {
          // new browsers api
          this.$refs.video.srcObject = stream;

          //this.$refs.video.setAttribute('width', this.width);
          //this.$refs.video.setAttribute('height', this.height);
        } else {
          // old broswers
          this.source = window.HTMLMediaElement.srcObject(stream);
        }
        // Emit video start/live event
        this.$refs.video.onloadedmetadata = () => {
          this.$emit("video-live", stream);
        };

        this.$emit("started", stream);
      },
      /**
       * stop the selected streamed video to change camera
       */
      stopStreamedVideo(videoElem) {
        let stream = videoElem.srcObject;
        let tracks = stream.getTracks();
        tracks.forEach(track => {
          // stops the video track
          track.stop();
          this.$emit("stopped", stream);

          this.$refs.video.srcObject = null;
          this.source = null;
        });
      },
      // Stop the video
      stop() {
        if (this.$refs.video !== null && this.$refs.video.srcObject) {
          this.stopStreamedVideo(this.$refs.video);
        }
      },
      // Start the video
      start() {
        if (this.deviceId) {
          this.loadCamera(this.deviceId);
        }
      },
      /**
       * test access
       */
      testMediaAccess() {

        navigator.mediaDevices
          .getUserMedia({ video: {
              width: this.$props.width,
              aspectRatio: {
                exact: this.$props.width / this.$props.height
              }
            }, audio: false})
          .then(stream => this.loadCameras())
          .catch(error => this.$emit("error", error));
      },
      /**
       * load the Camera passed as index!
       */
      loadCamera(device) {
        navigator.mediaDevices
          .getUserMedia({
            video: {
              deviceId: { exact: device },
              width: this.width,
              aspectRatio: {
                exact: this.width / this.height
              }
            },
            audio: false
          })
          .then(stream => this.loadSrcStream(stream))
          .catch(error => this.$emit("error", error));
      },
      capture() {
        return this.getCanvas().toDataURL(this.screenshotFormat);
      },
      getCanvasRaw() {
        return this.ctx.getImageData(0,0,this.canvas.width, this.canvas.height);
      },
      getCanvas() {
        let video = this.$refs.video;
        if (!this.ctx) {
          let canvas = document.createElement("canvas");
          canvas.height = video.height;
          canvas.width = video.width;
          this.canvas = canvas;
          this.ctx = canvas.getContext("2d");
          this.ctx.translate(canvas.width, 0);
          this.ctx.scale(-1, 1);
        }

        const { ctx, canvas } = this;
        ctx.clearRect(0, 0, this.width, this.height);

        ctx.drawImage(video, 0, 0, this.width, this.height);


        return canvas;
      }
    }
  };
</script>

<style scoped>
  video {
    width:100%;
    height: auto;
    /* Flip the video horizontally*/
    -webkit-transform: scaleX(-1);
    transform: scaleX(-1);
    border: 1px solid #6F6F6F;
  }
</style>
