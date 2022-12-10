<script setup>
import {ref, onMounted} from 'vue'

const textInput = ref(null);
const textData = ref("Freiberger Syntax")
const canvas = ref(null);
const wrapper = ref(null)

onMounted(() => {

  const ctx = canvas.value.getContext('2d');

  canvas.value.width = wrapper.value.clientWidth;
  canvas.value.height = wrapper.value.clientHeight;

  class Particle {
    constructor(effect, x, y, color) {
      this.effect = effect;
      this.x = Math.random() * this.effect.canvasWidth;
      // this.x = 0;
      this.y = Math.random() * this.effect.canvasHeight;
      this.color = color;
      this.originX = x;
      this.originY = y;
      this.size = this.effect.gap;
      this.dx = 1;
      this.dy = 0;
      this.vx = 0;
      this.vy = 0;
      this.force = 0;
      this.angle = 0;
      this.distance = 0;
      this.friction = Math.random() * 0.2 + 0.215;
      this.ease = Math.random() * 0.1 + 0.005;
    }
    draw() {
      this.effect.context.fillStyle = this.color;
      this.effect.context.fillRect(this.x, this.y, this.size, this.size)
    }
    update() {
      this.dx = this.effect.mouse.x - this.x;
      this.dy = this.effect.mouse.y - this.y;
      this.distance = this.dx * this.dx + this.dy * this.dy;
      this.force = -this.effect.mouse.radius / this.distance;

      if (this.distance < this.effect.mouse.radius) {
        this.angle = Math.atan2(this.dy, this.dx);
        this.vx += this.force * Math.cos(this.angle);
        this.vy += this.force * Math.sin(this.angle);
      }

      this.x += (this.vx *= this.friction) + (this.originX - this.x) * this.ease;
      this.y += (this.vy *= this.friction) + (this.originY - this.y) * this.ease;
    }
  }

  class Effect {
    constructor(context, canvasWidth, canvasHeight, textInput) {
      this.context = context;
      this.canvasWidth = canvasWidth + 22;
      this.canvasHeight = canvasHeight;
      this.textX = this.canvasWidth / 2;
      this.textY = this.canvasHeight / 2;
      this.fontSize = 100;
      this.lineHeight = this.fontSize * 0.8;
      this.maxTextWidth = this.canvasWidth * 0.8;
      this.textInput = textInput;
      // this.textInput.addEventListener('keyup', (e) => {
      //     if (e.key !== ' ') {
      //         this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      //         this.wrapText(e.target.value);
      //     }
      // })
      this.textInput.addEventListener('keyup', this.debounce((e) => {
        if (e.key !== ' ') {
          this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
          this.wrapText(e.target.value);
        }
      }, 500))

      this.particles = [];
      this.gap = 1;
      this.mouse = {
        radius: 10000,
        x: 0,
        y: 0
      }

      window.addEventListener('mousemove', (e) => {
        this.mouse.x = e.x;
        this.mouse.y = e.y;
      })
    }
    debounce(callback, wait) {
      let timeout;
      return (...args) => {
        clearTimeout(timeout);
        timeout = setTimeout(function () {
          callback.apply(this, args);
        }, wait);
      };
    }
    wrapText(text) {
      const gradient = this.context.createLinearGradient(0, 0, this.canvasWidth, this.canvasHeight);
      gradient.addColorStop(0.3, 'lightseagreen');
      gradient.addColorStop(0.5, 'orange');
      // gradient.addColorStop(0.6, 'darkorange');
      gradient.addColorStop(0.7, 'lightseagreen');
      this.context.fillStyle = gradient;
      this.context.lineWidth = "1px";
      this.context.strokeStyle = "black"
      this.context.textAlign = "center"
      this.context.textBaseline = 'middle';
      this.context.font = `${this.fontSize}px sans-serif`;
      this.context.fillText(text, this.textX, this.textY);
      this.context.strokeText(text, this.textX, this.textY);

      this.context.clearRect(0, 0, canvas.value.width, canvas.value.height)
      let linesArray = [];
      let lineCounter = 0;
      let line = '';
      let words = text.split(' ');
      for (let i = 0; i < words.length; i++) {
        let testLine = line + words[i] + ' ';
        if (this.context.measureText(testLine).width > this.maxTextWidth) {
          line = words[i] + ' ';
          lineCounter++;
        } else {
          line = testLine;
        }
        linesArray[lineCounter] = line;
      }
      let textHeight = this.lineHeight * lineCounter
      let textY = this.canvasHeight / 2 - textHeight / 2;
      linesArray.forEach((el, index) => {
        this.context.fillText(el, this.canvasWidth / 2, textY + (index * this.lineHeight));
        this.context.strokeText(el, this.canvasWidth / 2, textY + (index * this.lineHeight));
      })
      this.convertToParticles();
    }
    convertToParticles() {
      this.particles = [];
      const pixels = this.context.getImageData(0, 0, this.canvasWidth, this.canvasHeight).data;
      this.context.clearRect(0, 0, this.canvasWidth, this.canvasHeight)
      for (let y = 0; y < this.canvasWidth; y += this.gap) {
        for (let x = 0; x < this.canvasWidth; x += this.gap) {
          const index = (y * this.canvasWidth + x) * 4;
          const alpha = pixels[index + 3];
          if (alpha > 0) {
            const red = pixels[index];
            const green = pixels[index + 1];
            const blue = pixels[index + 2];
            const color = `rgb(${red}, ${green}, ${blue})`;
            this.particles.push(new Particle(this, x, y, color));
          }
        }
      }
    }
    render() {
      this.particles.forEach(particle => {
        particle.update();
        particle.draw();
      })
    }
    resize(width, height) {
      this.canvasWidth = width;
      this.canvasHeight = height;
      this.textX = this.canvasWidth / 2;
      this.textY = this.canvasHeight / 2;
      this.maxTextWidth = this.canvasWidth * 0.8;

    }
  }
  const effect = new Effect(ctx, canvas.value.width, canvas.value.height, textInput.value);
  effect.wrapText(textData.value);
  function animate() {

    ctx.clearRect(0, 0, canvas.value.width, canvas.value.height)

    effect.render()
    requestAnimationFrame(animate);

  }
  animate()

  window.addEventListener('resize', () => {
    canvas.value.width = wrapper.value.clientWidth;
    canvas.value.height = wrapper.value.clientHeight;
    effect.resize(canvas.value.width, canvas.value.height);
    effect.wrapText(textData.value);

  })
  // wrapText(textInput.value === null ? "Namstê!" : textInput.value)
})
// watchEffect(()=> {
//     if(textInput.value !== null) {
//         wrapText(textInput.value);
//         // console.log(textInput.value)
//     }
// })

</script>
<template>
  <input type="text" hidden ref="textInput" v-model="textData" placeholder="blabla">
  <div class="wrapper h-96" ref="wrapper">
    <canvas id="textCanvas" class="h-full w-full" ref="canvas"></canvas>
  </div>
</template>

<style >
#app {
  background: black;
  height: 100vh;
  width: 100vw;
  display: grid;
  place-items: center;
}

.wrapper {
  position: absolute;
  inset: 0;
}

canvas {
  position: absolute;
  inset: 0;
}
</style>
