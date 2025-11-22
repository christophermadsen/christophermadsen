<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Funky Perlin Noise</title>
  <style>
    html, body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      background: #000;
      height: 100%;
      width: 100%;
      font-family: system-ui, sans-serif;
    }

    canvas {
      display: block;
      width: 100vw;
      height: 100vh;
      image-rendering: pixelated; /* makes low-res canvas look nice and retro */
    }

    .label {
      position: fixed;
      left: 16px;
      bottom: 16px;
      color: rgba(255, 255, 255, 0.8);
      background: rgba(0, 0, 0, 0.4);
      padding: 6px 10px;
      border-radius: 999px;
      font-size: 12px;
      backdrop-filter: blur(4px);
      pointer-events: none;
    }
  </style>
</head>
<body>
  <canvas id="noiseCanvas"></canvas>
  <div class="label">Christopher Buch Madsen • stopherbm at gmail.com</div>

  <script>
    // --- Perlin Noise Implementation (3D) ---
    class Perlin {
      constructor() {
        this.p = new Uint8Array(512);
        this.permutation = new Uint8Array(256);

        // Fill permutation array with 0..255 and shuffle
        for (let i = 0; i < 256; i++) {
          this.permutation[i] = i;
        }
        for (let i = 255; i > 0; i--) {
          const j = (Math.random() * (i + 1)) | 0;
          [this.permutation[i], this.permutation[j]] = [this.permutation[j], this.permutation[i]];
        }

        // Duplicate into p[512]
        for (let i = 0; i < 512; i++) {
          this.p[i] = this.permutation[i & 255];
        }
      }

      fade(t) {
        return t * t * t * (t * (t * 6 - 15) + 10);
      }

      lerp(t, a, b) {
        return a + t * (b - a);
      }

      grad(hash, x, y, z) {
        const h = hash & 15;
        const u = h < 8 ? x : y;
        const v = h < 4 ? y : h === 12 || h === 14 ? x : z;
        return ((h & 1) === 0 ? u : -u) + ((h & 2) === 0 ? v : -v);
      }

      noise3(x, y, z) {
        const X = Math.floor(x) & 255;
        const Y = Math.floor(y) & 255;
        const Z = Math.floor(z) & 255;

        x -= Math.floor(x);
        y -= Math.floor(y);
        z -= Math.floor(z);

        const u = this.fade(x);
        const v = this.fade(y);
        const w = this.fade(z);

        const A = this.p[X] + Y;
        const AA = this.p[A] + Z;
        const AB = this.p[A + 1] + Z;
        const B = this.p[X + 1] + Y;
        const BA = this.p[B] + Z;
        const BB = this.p[B + 1] + Z;

        const g1 = this.grad(this.p[AA], x, y, z);
        const g2 = this.grad(this.p[BA], x - 1, y, z);
        const g3 = this.grad(this.p[AB], x, y - 1, z);
        const g4 = this.grad(this.p[BB], x - 1, y - 1, z);
        const g5 = this.grad(this.p[AA + 1], x, y, z - 1);
        const g6 = this.grad(this.p[BA + 1], x - 1, y, z - 1);
        const g7 = this.grad(this.p[AB + 1], x, y - 1, z - 1);
        const g8 = this.grad(this.p[BB + 1], x - 1, y - 1, z - 1);

        const lerpX1 = this.lerp(u, g1, g2);
        const lerpX2 = this.lerp(u, g3, g4);
        const lerpX3 = this.lerp(u, g5, g6);
        const lerpX4 = this.lerp(u, g7, g8);

        const lerpY1 = this.lerp(v, lerpX1, lerpX2);
        const lerpY2 = this.lerp(v, lerpX3, lerpX4);

        return this.lerp(w, lerpY1, lerpY2); // range approx [-1, 1]
      }
    }

    // --- HSL to RGB helper ---
    function hslToRgb(h, s, l) {
      h = h % 360;
      if (h < 0) h += 360;
      s /= 100;
      l /= 100;

      const c = (1 - Math.abs(2 * l - 1)) * s;
      const x = c * (1 - Math.abs((h / 60) % 2 - 1));
      const m = l - c / 2;

      let r1, g1, b1;
      if (h < 60) {
        [r1, g1, b1] = [c, x, 0];
      } else if (h < 120) {
        [r1, g1, b1] = [x, c, 0];
      } else if (h < 180) {
        [r1, g1, b1] = [0, c, x];
      } else if (h < 240) {
        [r1, g1, b1] = [0, x, c];
      } else if (h < 300) {
        [r1, g1, b1] = [x, 0, c];
      } else {
        [r1, g1, b1] = [c, 0, x];
      }

      const r = Math.round((r1 + m) * 255);
      const g = Math.round((g1 + m) * 255);
      const b = Math.round((b1 + m) * 255);

      return [r, g, b];
    }

    // --- Main animation setup ---
    const canvas = document.getElementById("noiseCanvas");
    const ctx = canvas.getContext("2d");
    const noise = new Perlin();

    let width, height, imageData;

    // Render at a slightly lower resolution for performance and upscale via CSS
    const RES_SCALE = 0.6;
    const NOISE_SCALE = 0.007; // spatial scale of the noise
    const SPEED = 0.00015;     // temporal speed

    function resize() {
      width = Math.floor(window.innerWidth * RES_SCALE);
      height = Math.floor(window.innerHeight * RES_SCALE);
      canvas.width = width;
      canvas.height = height;
      imageData = ctx.createImageData(width, height);
    }

    window.addEventListener("resize", resize);
    resize();

    function draw(timestamp) {
      const t = timestamp * SPEED;

      const data = imageData.data;
      let idx = 0;

      for (let y = 0; y < height; y++) {
        for (let x = 0; x < width; x++) {
          // Sample the 3D Perlin field (x, y, time)
          const nx = x * NOISE_SCALE;
          const ny = y * NOISE_SCALE;

          const n = noise.noise3(nx, ny, t); // [-1, 1]
          const n01 = (n + 1) * 0.5;         // -> [0, 1]

          // Funky colour mapping:
          // - hue varies with noise + time -> shifting rainbow
          // - saturation and lightness depend on noise for contrast
          const hue = (n01 * 360 + t * 200) % 360;
          const sat = 60 + 40 * Math.sin(n01 * Math.PI * 2);
          const light = 30 + 50 * n01;

          const [r, g, b] = hslToRgb(hue, sat, light);

          data[idx++] = r;
          data[idx++] = g;
          data[idx++] = b;
          data[idx++] = 255; // alpha
        }
      }

      ctx.putImageData(imageData, 0, 0);
      requestAnimationFrame(draw);
    }

    requestAnimationFrame(draw);
  </script>
</body>
</html>
