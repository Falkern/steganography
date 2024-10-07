<template>
  <div id="app" class="app">
    <h1 class="title">Image Steganography Tool</h1>

    <input
      type="file"
      @change="onImageUpload"
      accept="image/png, image/bmp"
      class="file-input"
    />

    <textarea
      v-model="message"
      placeholder="Enter your message here"
      class="message-input"
    ></textarea>

    <div class="button-container">
      <button @click="embedMessage" class="embed-button">
        Embed Message
      </button>
      <button @click="extractMessage" class="extract-button">
        Extract Message
      </button>
    </div>

    <canvas ref="canvas" style="display:none;"></canvas>

    <a v-if="downloadLink" :href="downloadLink" download="stego_image.png" class="download-link">
      Download Image
    </a>

    <p v-if="extractedMessage" class="extracted-message">Extracted Message: {{ extractedMessage }}</p>
    <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      image: null,
      message: '',
      downloadLink: '',
      extractedMessage: '',
      errorMessage: '',
    };
  },
  methods: {
    onImageUpload(event) {
      this.errorMessage = '';
      const file = event.target.files[0];
      const reader = new FileReader();

      reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
          this.$refs.canvas.width = img.width;
          this.$refs.canvas.height = img.height;
          const ctx = this.$refs.canvas.getContext('2d');
          ctx.drawImage(img, 0, 0);
          this.image = ctx.getImageData(0, 0, img.width, img.height);
        };
        img.src = e.target.result;
      };
      reader.readAsDataURL(file);
    },
    embedMessage() {
      this.errorMessage = ''; 
      if (!this.image) {
        this.errorMessage = 'Please upload an image first.';
        return;
      }
      const messageLength = this.message.length;
      if (messageLength === 0) {
        this.errorMessage = 'Message cannot be empty.';
        return;
      }
      if (messageLength * 8 > this.image.data.length / 4) {
        this.errorMessage = 'Message is too long for the selected image.';
        return;
      }

      // convert message to binary and append null terminator
      const binaryMessage = this.message
        .split('')
        .map((char) => char.charCodeAt(0).toString(2).padStart(8, '0'))
        .join('') + '00000000'; 
      const binaryLength = binaryMessage.length;

      const imgData = this.image;
      for (let i = 0; i < binaryLength; i++) {
        imgData.data[i * 4] = (imgData.data[i * 4] & ~1) | binaryMessage[i];
      }

      const ctx = this.$refs.canvas.getContext('2d');
      ctx.putImageData(imgData, 0, 0);
      this.downloadLink = this.$refs.canvas.toDataURL('image/png');
    },
    extractMessage() {
      if (!this.image) {
        this.errorMessage = 'Please upload an image first.';
        return;
      }

      const imgData = this.image;
      let binaryMessage = '';
      for (let i = 0; i < imgData.data.length; i += 4) {
        binaryMessage += (imgData.data[i] & 1).toString(); 
      }

      // convert binary to text
      const messageArray = binaryMessage.match(/.{1,8}/g);
      if (messageArray) {
        const nullCharIndex = messageArray.indexOf('00000000');
        if (nullCharIndex !== -1) {
          messageArray.length = nullCharIndex; 
        }
        this.extractedMessage = messageArray
          .map((byte) => String.fromCharCode(parseInt(byte, 2)))
          .join('');
      } else {
        this.extractedMessage = '';
      }
    },
  },
};
</script>

<style>
</style>
