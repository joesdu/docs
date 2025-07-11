```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Particle Animation</title>
    <style>
      canvas {
        position: fixed;
        top: 0;
        left: 0;
        z-index: -1;
      }
    </style>
  </head>
  <body>
    <canvas id="particleCanvas"></canvas>
    <script>
      class ParticleAnimation {
        constructor() {
          this.canvas = document.getElementById("particleCanvas");
          this.ctx = this.canvas.getContext("2d");
          this.settings = this.getSettings();
          this.particles = [];
          this.mouse = { x: null, y: null, maxDistance: 20000 };
          this.grid = new Map();
          this.gridSize = 100; // Grid cell size for spatial partitioning

          this.initializeCanvas();
          this.initializeParticles();
          this.setupEventListeners();
          this.animate();
        }

        getSettings() {
          const script = document.getElementsByTagName("script");
          const lastScript = script[script.length - 1];
          return {
            zIndex: lastScript.getAttribute("zIndex") || -1,
            opacity: parseFloat(lastScript.getAttribute("opacity") || 0.5),
            color: lastScript.getAttribute("color") || "0,0,0",
            count: parseInt(lastScript.getAttribute("count") || 99, 10),
          };
        }

        initializeCanvas() {
          this.canvas.style.zIndex = this.settings.zIndex;
          this.canvas.style.opacity = this.settings.opacity;
          this.resizeCanvas();
        }

        resizeCanvas() {
          this.canvas.width =
            window.innerWidth ||
            document.documentElement.clientWidth ||
            document.body.clientWidth;
          this.canvas.height =
            window.innerHeight ||
            document.documentElement.clientHeight ||
            document.body.clientHeight;
          this.updateGrid();
        }

        initializeParticles() {
          this.particles = Array.from({ length: this.settings.count }, () => ({
            x: Math.random() * this.canvas.width,
            y: Math.random() * this.canvas.height,
            vx: 2 * Math.random() - 1,
            vy: 2 * Math.random() - 1,
            maxDistance: 6000,
          }));
          this.particles.push(this.mouse);
          this.updateGrid();
        }

        updateGrid() {
          this.grid.clear();
          this.particles.forEach((particle, index) => {
            if (particle.x === null || particle.y === null) return;
            const gridX = Math.floor(particle.x / this.gridSize);
            const gridY = Math.floor(particle.y / this.gridSize);
            const key = `${gridX},${gridY}`;
            if (!this.grid.has(key)) {
              this.grid.set(key, []);
            }
            this.grid.get(key).push({ particle, index });
          });
        }

        updateParticles() {
          this.particles.forEach((particle) => {
            if (particle.x === null || particle.y === null) return;

            particle.x += particle.vx;
            particle.y += particle.vy;

            // Bounce off canvas edges
            if (particle.x > this.canvas.width || particle.x < 0) {
              particle.vx *= -1;
            }
            if (particle.y > this.canvas.height || particle.y < 0) {
              particle.vy *= -1;
            }
          });
          this.updateGrid();
        }

        draw() {
          this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

          this.particles.forEach((particle, index) => {
            if (particle.x === null || particle.y === null) return;

            // Draw particle
            this.ctx.fillStyle = `rgb(${this.settings.color})`;
            this.ctx.fillRect(particle.x - 0.5, particle.y - 0.5, 1, 1);

            // Draw lines to nearby particles using spatial grid
            const gridX = Math.floor(particle.x / this.gridSize);
            const gridY = Math.floor(particle.y / this.gridSize);
            for (let dx = -1; dx <= 1; dx++) {
              for (let dy = -1; dy <= 1; dy++) {
                const key = `${gridX + dx},${gridY + dy}`;
                const neighbors = this.grid.get(key) || [];
                neighbors.forEach(
                  ({ particle: neighbor, index: neighborIndex }) => {
                    if (neighborIndex <= index) return;
                    const dx = particle.x - neighbor.x;
                    const dy = particle.y - neighbor.y;
                    const distanceSquared = dx * dx + dy * dy;
                    const maxDistance =
                      neighbor === this.mouse
                        ? neighbor.maxDistance
                        : particle.maxDistance;

                    if (distanceSquared < maxDistance) {
                      if (
                        neighbor === this.mouse &&
                        distanceSquared >= maxDistance / 2
                      ) {
                        particle.x -= 0.03 * dx;
                        particle.y -= 0.03 * dy;
                      }
                      const opacity =
                        (maxDistance - distanceSquared) / maxDistance;
                      this.ctx.beginPath();
                      this.ctx.lineWidth = opacity / 2;
                      this.ctx.strokeStyle = `rgba(${this.settings.color},${
                        opacity + 0.2
                      })`;
                      this.ctx.moveTo(particle.x, particle.y);
                      this.ctx.lineTo(neighbor.x, neighbor.y);
                      this.ctx.stroke();
                    }
                  }
                );
              }
            }
          });
        }

        animate() {
          this.updateParticles();
          this.draw();
          requestAnimationFrame(() => this.animate());
        }

        setupEventListeners() {
          let resizeTimeout;
          window.addEventListener("resize", () => {
            clearTimeout(resizeTimeout);
            resizeTimeout = setTimeout(() => this.resizeCanvas(), 100);
          });

          window.addEventListener("mousemove", (event) => {
            this.mouse.x = event.clientX;
            this.mouse.y = event.clientY;
            this.updateGrid();
          });

          window.addEventListener("mouseout", () => {
            this.mouse.x = null;
            this.mouse.y = null;
            this.updateGrid();
          });
        }
      }

      // Initialize the animation
      new ParticleAnimation();
    </script>
  </body>
</html>
```

原版代码:

```javascript
!(function () {
  function n(n, e, t) {
    return n.getAttribute(e) || t;
  }
  function e(n) {
    return document.getElementsByTagName(n);
  }
  function t() {
    var t = e("script"),
      o = t.length,
      i = t[o - 1];
    return {
      l: o,
      z: n(i, "zIndex", -1),
      o: n(i, "opacity", 0.5),
      c: n(i, "color", "0,0,0"),
      n: n(i, "count", 99),
    };
  }
  function o() {
    (a = m.width =
      window.innerWidth ||
      document.documentElement.clientWidth ||
      document.body.clientWidth),
      (c = m.height =
        window.innerHeight ||
        document.documentElement.clientHeight ||
        document.body.clientHeight);
  }
  function i() {
    r.clearRect(0, 0, a, c);
    var n, e, t, o, m, l;
    s.forEach(function (i, x) {
      for (
        i.x += i.xa,
          i.y += i.ya,
          i.xa *= i.x > a || i.x < 0 ? -1 : 1,
          i.ya *= i.y > c || i.y < 0 ? -1 : 1,
          r.fillRect(i.x - 0.5, i.y - 0.5, 1, 1),
          e = x + 1;
        e < u.length;
        e++
      )
        (n = u[e]),
          null !== n.x &&
            null !== n.y &&
            ((o = i.x - n.x),
            (m = i.y - n.y),
            (l = o * o + m * m),
            l < n.max &&
              (n === y &&
                l >= n.max / 2 &&
                ((i.x -= 0.03 * o), (i.y -= 0.03 * m)),
              (t = (n.max - l) / n.max),
              r.beginPath(),
              (r.lineWidth = t / 2),
              (r.strokeStyle = "rgba(" + d.c + "," + (t + 0.2) + ")"),
              r.moveTo(i.x, i.y),
              r.lineTo(n.x, n.y),
              r.stroke()));
    }),
      x(i);
  }
  var a,
    c,
    u,
    m = document.createElement("canvas"),
    d = t(),
    l = "c_n" + d.l,
    r = m.getContext("2d"),
    x =
      window.requestAnimationFrame ||
      window.webkitRequestAnimationFrame ||
      window.mozRequestAnimationFrame ||
      window.oRequestAnimationFrame ||
      window.msRequestAnimationFrame ||
      function (n) {
        window.setTimeout(n, 1e3 / 45);
      },
    w = Math.random,
    y = { x: null, y: null, max: 2e4 };
  (m.id = l),
    (m.style.cssText =
      "position:fixed;top:0;left:0;z-index:" + d.z + ";opacity:" + d.o),
    e("body")[0].appendChild(m),
    o(),
    (window.onresize = o),
    (window.onmousemove = function (n) {
      (n = n || window.event), (y.x = n.clientX), (y.y = n.clientY);
    }),
    (window.onmouseout = function () {
      (y.x = null), (y.y = null);
    });
  for (var s = [], f = 0; d.n > f; f++) {
    var h = w() * a,
      g = w() * c,
      v = 2 * w() - 1,
      p = 2 * w() - 1;
    s.push({ x: h, y: g, xa: v, ya: p, max: 6e3 });
  }
  (u = s.concat([y])),
    setTimeout(function () {
      i();
    }, 100);
})();
```
