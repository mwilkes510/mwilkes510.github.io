<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Project Delirium - Dreamwalk</title>
    <style>
      :root {
        --ink: #efe7d6;
        --ink-dim: rgba(239, 231, 214, 0.7);
        --veil: rgba(8, 12, 18, 0.72);
        --glow: rgba(110, 196, 255, 0.5);
        --accent: #f4b9ff;
      }

      * {
        box-sizing: border-box;
      }

      html,
      body {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
        background: radial-gradient(circle at top, #202a3b, #0d1017 55%, #07080c);
        font-family: "Baskerville", "Palatino", "Georgia", serif;
        color: var(--ink);
      }

      #stage {
        display: block;
        width: 100%;
        height: 100%;
      }

      #ui {
        position: fixed;
        inset: 0;
        pointer-events: none;
      }

      #hud {
        position: fixed;
        top: 16px;
        left: 16px;
        font-family: "Courier New", "Andale Mono", monospace;
        font-size: 13px;
        letter-spacing: 0.08em;
        text-transform: uppercase;
        color: var(--ink-dim);
        text-shadow: 0 0 12px rgba(160, 215, 255, 0.35);
        line-height: 1.6;
        white-space: pre;
      }

      #message {
        position: fixed;
        bottom: 20px;
        left: 50%;
        transform: translateX(-50%);
        padding: 10px 18px;
        border-radius: 999px;
        background: rgba(9, 12, 18, 0.6);
        border: 1px solid rgba(255, 255, 255, 0.08);
        font-size: 14px;
        color: var(--ink);
        text-align: center;
        max-width: min(560px, 88vw);
        text-shadow: 0 0 14px rgba(120, 200, 255, 0.35);
      }

      #credits {
        position: fixed;
        right: 18px;
        bottom: 18px;
        font-size: 12px;
        letter-spacing: 0.08em;
        color: rgba(239, 231, 214, 0.45);
        text-transform: uppercase;
      }

      #start {
        position: fixed;
        inset: 0;
        display: flex;
        align-items: center;
        justify-content: center;
        background: radial-gradient(circle at center, rgba(62, 80, 108, 0.85), rgba(10, 12, 20, 0.92));
        transition: opacity 1s ease;
        pointer-events: auto;
      }

      #start.hidden {
        opacity: 0;
        pointer-events: none;
      }

      .card {
        padding: 32px 36px;
        border-radius: 20px;
        border: 1px solid rgba(255, 255, 255, 0.12);
        background: rgba(14, 18, 28, 0.82);
        box-shadow: 0 24px 60px rgba(0, 0, 0, 0.5);
        max-width: 420px;
        text-align: center;
        animation: breathe 6s ease-in-out infinite;
      }

      .card h1 {
        margin: 0 0 12px;
        font-size: 28px;
        letter-spacing: 0.1em;
        text-transform: uppercase;
        color: var(--ink);
      }

      .card p {
        margin: 10px 0;
        font-size: 15px;
        color: var(--ink-dim);
        line-height: 1.5;
      }

      #startButton {
        margin-top: 16px;
        padding: 10px 22px;
        border-radius: 999px;
        border: 1px solid rgba(255, 255, 255, 0.18);
        background: linear-gradient(130deg, #182235, #2a3754);
        color: var(--ink);
        font-size: 14px;
        letter-spacing: 0.12em;
        text-transform: uppercase;
        cursor: pointer;
        box-shadow: 0 0 18px rgba(120, 180, 255, 0.25);
        transition: transform 0.2s ease, box-shadow 0.2s ease;
      }

      #startButton:hover {
        transform: translateY(-1px);
        box-shadow: 0 0 22px rgba(160, 210, 255, 0.35);
      }

      @keyframes breathe {
        0%,
        100% {
          transform: translateY(0);
        }
        50% {
          transform: translateY(-6px);
        }
      }
    </style>
  </head>
  <body>
    <canvas id="stage"></canvas>
    <div id="ui">
      <div id="hud"></div>
      <div id="message"></div>
      <div id="credits">Dreamwalk</div>
    </div>
    <div id="start">
      <div class="card">
        <h1>Project Delirium</h1>
        <p>A quiet room. A soft light. A path that should make sense.</p>
        <p>Move with arrow keys or WASD. Touch and drag works too.</p>
        <p>Collect the lights. Press M to mute. Press R to restart.</p>
        <button id="startButton">Enter</button>
      </div>
    </div>

    <script>
      (() => {
        const canvas = document.getElementById("stage");
        const ctx = canvas.getContext("2d");
        const hud = document.getElementById("hud");
        const messageEl = document.getElementById("message");
        const startOverlay = document.getElementById("start");
        const startButton = document.getElementById("startButton");

        const keys = {};
        const pointer = { active: false, x: 0, y: 0 };
        const orbs = [];
        const pulses = [];
        const whispers = [];

        const state = {
          started: false,
          collected: 0,
          phase: 0,
          targetEnd: 25,
          lastTime: 0,
          startTime: 0,
          message: "Collect the quiet lights.",
          messageUntil: 0,
          ended: false,
          audioMuted: false,
        };

        const player = {
          x: 200,
          y: 200,
          vx: 0,
          vy: 0,
          r: 9,
        };

        const phaseMessages = [
          "You wake in a quiet room.",
          "The walls hum with a soft echo.",
          "The corridor folds into itself.",
          "You misplace the floor.",
          "There is only motion now.",
        ];

        const phaseInstructions = [
          "Collect the quiet lights. Stay awake.",
          "Follow the soft echo. Keep moving.",
          "Find the memory shards. They drift.",
          "Hold to the thread. It frays.",
          "Let the dream carry you.",
        ];

        const whisperPools = [
          [
            "hello",
            "remember",
            "light",
            "breathe",
            "window",
            "quiet",
            "lantern",
            "sigh",
            "warm",
            "paper",
            "linen",
            "dust",
            "gentle",
            "glow",
            "still",
            "blue",
            "hand",
            "chair",
            "slow",
            "morning",
            "cold",
            "flicker",
            "empty",
          ],
          [
            "soft",
            "near",
            "hush",
            "drift",
            "listen",
            "distant",
            "tide",
            "murmur",
            "low",
            "feather",
            "veil",
            "pulse",
            "lull",
            "bloom",
            "glide",
            "float",
            "faint",
            "ripple",
            "glimmer",
            "sway",
            "shadow",
            "tremble",
            "hollow",
          ],
          [
            "bend",
            "loop",
            "fold",
            "echo",
            "skin",
            "spiral",
            "melt",
            "blur",
            "tilt",
            "split",
            "fray",
            "tremor",
            "glitch",
            "ravel",
            "twist",
            "hinge",
            "rift",
            "suture",
            "warp",
            "lattice",
            "scar",
            "stain",
            "needle",
            "static",
            "hushbone",
          ],
          [
            "north",
            "south",
            "left",
            "right",
            "down",
            "up",
            "inside",
            "outside",
            "between",
            "beneath",
            "beyond",
            "under",
            "over",
            "here",
            "there",
            "nearby",
            "elsewhere",
            "forward",
            "backward",
            "sideways",
            "behind",
            "within",
            "underfoot",
            "above you",
            "no exit",
          ],
          [
            "you",
            "not you",
            "awake?",
            "not real",
            "more",
            "less",
            "maybe",
            "repeat",
            "again",
            "never",
            "already",
            "borrowed",
            "unmade",
            "hollow",
            "overfull",
            "drifting",
            "closer",
            "too close",
            "who am i",
            "remember me",
            "no face",
            "watching",
            "dont look",
            "teeth",
            "dark",
            "fade out",
            "i am here",
          ],
        ];

        const audio = {
          ctx: null,
          master: null,
          filter: null,
          baseOsc: null,
          driftOsc: null,
          lfo: null,
          lfoGain: null,
          noise: null,
          noiseGain: null,
        };

        function resize() {
          const dpr = window.devicePixelRatio || 1;
          const width = window.innerWidth;
          const height = window.innerHeight;
          canvas.width = width * dpr;
          canvas.height = height * dpr;
          ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
        }

        function randomBetween(min, max) {
          return min + Math.random() * (max - min);
        }

        function clamp(value, min, max) {
          return Math.max(min, Math.min(max, value));
        }

        function announce(text) {
          state.message = text;
          state.messageUntil = performance.now() + 4200;
        }

        function scrambleText(text, amount) {
          if (amount <= 0) return text;
          const chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789@#$%";
          let result = "";
          for (let i = 0; i < text.length; i += 1) {
            const ch = text[i];
            if (ch !== " " && Math.random() < amount) {
              result += chars[Math.floor(Math.random() * chars.length)];
            } else {
              result += ch;
            }
          }
          return result;
        }

        function spawnOrb() {
          const width = canvas.width / (window.devicePixelRatio || 1);
          const height = canvas.height / (window.devicePixelRatio || 1);
          let tries = 0;
          let x = 0;
          let y = 0;
          do {
            x = randomBetween(40, width - 40);
            y = randomBetween(40, height - 40);
            tries += 1;
          } while (tries < 10 && Math.hypot(x - player.x, y - player.y) < 140);
          orbs.push({
            x,
            y,
            r: randomBetween(7, 13),
            hue: randomBetween(160, 320),
            drift: randomBetween(0.6, 1.6),
          });
        }

        function spawnPulse(x, y) {
          pulses.push({
            x,
            y,
            r: 0,
            life: 1,
          });
        }

        function spawnWhisper(now) {
          const width = canvas.width / (window.devicePixelRatio || 1);
          const height = canvas.height / (window.devicePixelRatio || 1);
          const phase = state.phase;
          const pool = whisperPools[phase] || whisperPools[0];
          const text = pool[Math.floor(Math.random() * pool.length)];
          whispers.push({
            text,
            x: randomBetween(40, width - 40),
            y: randomBetween(40, height - 40),
            life: randomBetween(1.2, 2.6),
            driftX: randomBetween(-18, 18),
            driftY: randomBetween(-12, 12),
            born: now,
          });
        }

        function resetGame() {
          state.collected = 0;
          state.phase = 0;
          state.ended = false;
          state.message = phaseInstructions[0];
          state.messageUntil = 0;
          player.x = canvas.width / (window.devicePixelRatio || 1) / 2;
          player.y = canvas.height / (window.devicePixelRatio || 1) / 2;
          player.vx = 0;
          player.vy = 0;
          orbs.length = 0;
          pulses.length = 0;
          whispers.length = 0;
          for (let i = 0; i < 4; i += 1) {
            spawnOrb();
          }
          announce(phaseMessages[0]);
        }

        function setupAudio() {
          try {
            audio.ctx = new (window.AudioContext || window.webkitAudioContext)();
          } catch (err) {
            audio.ctx = null;
            return;
          }
          audio.master = audio.ctx.createGain();
          audio.master.gain.value = 0.05;
          audio.filter = audio.ctx.createBiquadFilter();
          audio.filter.type = "lowpass";
          audio.filter.frequency.value = 700;

          audio.baseOsc = audio.ctx.createOscillator();
          audio.baseOsc.type = "sine";
          audio.baseOsc.frequency.value = 110;

          audio.driftOsc = audio.ctx.createOscillator();
          audio.driftOsc.type = "triangle";
          audio.driftOsc.frequency.value = 220;

          audio.lfo = audio.ctx.createOscillator();
          audio.lfo.type = "sine";
          audio.lfo.frequency.value = 0.18;

          audio.lfoGain = audio.ctx.createGain();
          audio.lfoGain.gain.value = 28;

          const buffer = audio.ctx.createBuffer(1, audio.ctx.sampleRate * 2, audio.ctx.sampleRate);
          const data = buffer.getChannelData(0);
          for (let i = 0; i < data.length; i += 1) {
            data[i] = Math.random() * 2 - 1;
          }
          audio.noise = audio.ctx.createBufferSource();
          audio.noise.buffer = buffer;
          audio.noise.loop = true;

          audio.noiseGain = audio.ctx.createGain();
          audio.noiseGain.gain.value = 0;

          audio.baseOsc.connect(audio.filter);
          audio.driftOsc.connect(audio.filter);
          audio.noise.connect(audio.noiseGain);
          audio.noiseGain.connect(audio.filter);
          audio.filter.connect(audio.master);
          audio.master.connect(audio.ctx.destination);
          audio.lfo.connect(audio.lfoGain);
          audio.lfoGain.connect(audio.baseOsc.frequency);

          audio.baseOsc.start();
          audio.driftOsc.start();
          audio.lfo.start();
          audio.noise.start();
        }

        function updateAudio(now, madness) {
          if (!audio.ctx || !audio.master) return;
          const baseGain = state.audioMuted ? 0 : 0.035 + madness * 0.05;
          audio.master.gain.setTargetAtTime(baseGain, audio.ctx.currentTime, 0.6);
          const filterFreq = 400 + madness * 1300 + Math.sin(now * 0.0006) * 100;
          audio.filter.frequency.setTargetAtTime(filterFreq, audio.ctx.currentTime, 0.6);
          audio.baseOsc.frequency.setTargetAtTime(100 + madness * 160, audio.ctx.currentTime, 0.6);
          audio.driftOsc.frequency.setTargetAtTime(180 + madness * 260, audio.ctx.currentTime, 0.6);
          audio.noiseGain.gain.setTargetAtTime(state.phase >= 3 ? 0.01 + madness * 0.03 : 0, audio.ctx.currentTime, 0.8);
        }

        function update(dt, now) {
          const width = canvas.width / (window.devicePixelRatio || 1);
          const height = canvas.height / (window.devicePixelRatio || 1);
          const madness = clamp(state.collected / state.targetEnd, 0, 1);
          const nextPhase = Math.min(4, Math.floor(state.collected / 5));
          if (nextPhase !== state.phase) {
            state.phase = nextPhase;
            announce(phaseMessages[nextPhase]);
          }

          if (!state.ended && state.collected >= state.targetEnd) {
            state.ended = true;
            announce("The dream blinks. Press R to drift again.");
          }

          let moveX = 0;
          let moveY = 0;
          if (keys["arrowleft"] || keys["a"]) moveX -= 1;
          if (keys["arrowright"] || keys["d"]) moveX += 1;
          if (keys["arrowup"] || keys["w"]) moveY -= 1;
          if (keys["arrowdown"] || keys["s"]) moveY += 1;

          if (pointer.active) {
            const dx = pointer.x - player.x;
            const dy = pointer.y - player.y;
            const dist = Math.hypot(dx, dy) || 1;
            moveX = dx / dist;
            moveY = dy / dist;
          }

          const speed = 840 + madness * 480;
          const inputMag = Math.hypot(moveX, moveY);
          if (inputMag > 0) {
            let angle = Math.atan2(moveY, moveX);
            const warp = (state.phase >= 2 ? 0.55 : 0.2) * madness;
            angle += Math.sin(now * 0.0014 + state.collected) * warp;
            if (state.phase >= 3 && Math.sin(now * 0.0007) > 0.6) {
              angle += Math.PI;
            }
            player.vx += Math.cos(angle) * speed * dt;
            player.vy += Math.sin(angle) * speed * dt;
          }

          const drift = (Math.sin(now * 0.001 + player.x * 0.01) + Math.cos(now * 0.0012 + player.y * 0.02)) * madness * 20;
          player.vx += Math.cos(now * 0.002 + player.y * 0.01) * drift * dt;
          player.vy += Math.sin(now * 0.002 + player.x * 0.01) * drift * dt;

          player.vx *= 0.86;
          player.vy *= 0.86;
          player.x += player.vx * dt;
          player.y += player.vy * dt;

          if (state.phase < 2) {
            player.x = clamp(player.x, 20, width - 20);
            player.y = clamp(player.y, 20, height - 20);
          } else {
            if (player.x < -20) player.x = width + 20;
            if (player.x > width + 20) player.x = -20;
            if (player.y < -20) player.y = height + 20;
            if (player.y > height + 20) player.y = -20;
          }

          const targetOrbs = state.phase < 4 ? 4 : 6;
          while (orbs.length < targetOrbs) {
            spawnOrb();
          }

          for (let i = orbs.length - 1; i >= 0; i -= 1) {
            const orb = orbs[i];
            const wander = Math.sin(now * 0.0012 + orb.drift) * (8 + madness * 30);
            orb.x += Math.cos(now * 0.0011 + orb.drift) * wander * dt;
            orb.y += Math.sin(now * 0.0014 + orb.drift) * wander * dt;

            if (state.phase >= 3) {
              const dx = player.x - orb.x;
              const dy = player.y - orb.y;
              const dist = Math.hypot(dx, dy) || 1;
              if (dist < 160) {
                orb.x -= (dx / dist) * (20 + madness * 80) * dt;
                orb.y -= (dy / dist) * (20 + madness * 80) * dt;
              }
            }

            if (orb.x < -40) orb.x = width + 40;
            if (orb.x > width + 40) orb.x = -40;
            if (orb.y < -40) orb.y = height + 40;
            if (orb.y > height + 40) orb.y = -40;

            const distToPlayer = Math.hypot(player.x - orb.x, player.y - orb.y);
            if (distToPlayer < player.r + orb.r) {
              orbs.splice(i, 1);
              state.collected += 1;
              spawnPulse(orb.x, orb.y);
              announce(phaseInstructions[state.phase]);
            }
          }

          for (let i = pulses.length - 1; i >= 0; i -= 1) {
            const pulse = pulses[i];
            pulse.r += 140 * dt;
            pulse.life -= dt * 0.9;
            if (pulse.life <= 0) {
              pulses.splice(i, 1);
            }
          }

          if (state.phase >= 1) {
            if (now - state.startTime > 900 && Math.random() < 0.018 + madness * 0.05) {
              spawnWhisper(now);
            }
          }

          for (let i = whispers.length - 1; i >= 0; i -= 1) {
            const whisper = whispers[i];
            whisper.life -= dt;
            whisper.x += whisper.driftX * dt;
            whisper.y += whisper.driftY * dt;
            if (whisper.life <= 0) {
              whispers.splice(i, 1);
            }
          }

          updateAudio(now, madness);
        }

        function drawBackground(now, madness) {
          const dpr = window.devicePixelRatio || 1;
          const width = canvas.width / dpr;
          const height = canvas.height / dpr;
          const baseHue = 205 + madness * 120 + Math.sin(now * 0.00025) * 20;
          const gradient = ctx.createRadialGradient(
            width * 0.5,
            height * 0.4,
            10,
            width * 0.5,
            height * 0.6,
            width * 0.8
          );
          gradient.addColorStop(0, `hsla(${baseHue}, 48%, 26%, 0.9)`);
          gradient.addColorStop(1, `hsla(${baseHue + 30}, 38%, 8%, 0.9)`);
          ctx.globalAlpha = state.phase >= 2 ? 0.35 : 1;
          ctx.fillStyle = gradient;
          ctx.fillRect(0, 0, width, height);
          ctx.globalAlpha = 1;

          if (state.phase >= 1) {
            ctx.strokeStyle = `rgba(160, 200, 255, ${0.06 + madness * 0.08})`;
            ctx.lineWidth = 1;
            for (let i = 0; i < 8; i += 1) {
              ctx.beginPath();
              const offset = i * 40 + Math.sin(now * 0.001 + i) * 20 * madness;
              for (let x = 0; x <= width; x += 30) {
                const y = height * 0.2 + offset + Math.sin(x * 0.015 + now * 0.001 + i) * 12;
                if (x === 0) {
                  ctx.moveTo(x, y);
                } else {
                  ctx.lineTo(x, y);
                }
              }
              ctx.stroke();
            }
          }
        }

        function draw(now) {
          const dpr = window.devicePixelRatio || 1;
          const width = canvas.width / dpr;
          const height = canvas.height / dpr;
          const madness = clamp(state.collected / state.targetEnd, 0, 1);

          if (state.phase >= 3) {
            const jitter = (Math.random() - 0.5) * 10 * madness;
            ctx.setTransform(dpr, 0, 0, dpr, jitter * dpr, jitter * dpr);
          } else {
            ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
          }

          drawBackground(now, madness);

          if (state.phase >= 2) {
            ctx.save();
            ctx.globalAlpha = 0.12 + madness * 0.1;
            ctx.fillStyle = `rgba(60, 120, 200, 0.18)`;
            for (let i = 0; i < 5; i += 1) {
              ctx.beginPath();
              const radius = 80 + i * 90 + Math.sin(now * 0.001 + i) * 40;
              ctx.arc(width * 0.5, height * 0.5, radius, 0, Math.PI * 2);
              ctx.stroke();
            }
            ctx.restore();
          }

          for (const pulse of pulses) {
            ctx.strokeStyle = `rgba(180, 220, 255, ${pulse.life * 0.4})`;
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.arc(pulse.x, pulse.y, pulse.r, 0, Math.PI * 2);
            ctx.stroke();
          }

          for (const orb of orbs) {
            ctx.save();
            ctx.globalCompositeOperation = "lighter";
            ctx.fillStyle = `hsla(${orb.hue}, 80%, 70%, 0.85)`;
            ctx.beginPath();
            ctx.arc(orb.x, orb.y, orb.r, 0, Math.PI * 2);
            ctx.fill();
            ctx.shadowBlur = 18 + madness * 30;
            ctx.shadowColor = `hsla(${orb.hue}, 90%, 70%, 0.9)`;
            ctx.beginPath();
            ctx.arc(orb.x, orb.y, orb.r * 0.6, 0, Math.PI * 2);
            ctx.fill();
            ctx.restore();
          }

          if (state.phase >= 2) {
            ctx.save();
            ctx.globalAlpha = 0.2 + madness * 0.3;
            ctx.fillStyle = "rgba(200, 240, 255, 0.25)";
            ctx.beginPath();
            ctx.arc(player.x + 18, player.y - 12, player.r * 0.85, 0, Math.PI * 2);
            ctx.fill();
            ctx.restore();
          }

          ctx.save();
          ctx.globalCompositeOperation = "lighter";
          ctx.fillStyle = "rgba(220, 246, 255, 0.9)";
          ctx.shadowBlur = 25;
          ctx.shadowColor = "rgba(170, 220, 255, 0.8)";
          ctx.beginPath();
          ctx.arc(player.x, player.y, player.r, 0, Math.PI * 2);
          ctx.fill();
          ctx.restore();

          for (const whisper of whispers) {
            const age = 1 - whisper.life / 2.6;
            ctx.fillStyle = `rgba(255, 245, 230, ${0.2 + age * 0.4})`;
            ctx.font = "14px 'Courier New', monospace";
            ctx.fillText(whisper.text, whisper.x, whisper.y);
          }

          if (state.phase >= 3) {
            ctx.save();
            ctx.globalAlpha = 0.08 + madness * 0.1;
            ctx.fillStyle = "rgba(255, 255, 255, 0.08)";
            for (let i = 0; i < 40; i += 1) {
              const x = Math.random() * width;
              const y = Math.random() * height;
              const w = Math.random() * 40 + 4;
              const h = Math.random() * 2 + 1;
              ctx.fillRect(x, y, w, h);
            }
            ctx.restore();
          }

          if (state.phase >= 4) {
            ctx.save();
            ctx.globalCompositeOperation = "screen";
            ctx.strokeStyle = `rgba(255, 180, 240, ${0.12 + madness * 0.2})`;
            ctx.lineWidth = 1;
            for (let i = 0; i < 16; i += 1) {
              ctx.beginPath();
              const radius = 50 + i * 30 + Math.sin(now * 0.0014 + i) * 20;
              ctx.arc(player.x, player.y, radius, 0, Math.PI * 2);
              ctx.stroke();
            }
            ctx.restore();
          }

          if (state.phase >= 3) {
            const slices = 4 + Math.floor(madness * 6);
            for (let i = 0; i < slices; i += 1) {
              const y = Math.random() * height;
              const sliceH = 6 + Math.random() * 18;
              const dx = (Math.random() - 0.5) * 30 * madness;
              ctx.drawImage(canvas, 0, y * dpr, width * dpr, sliceH * dpr, dx, y, width, sliceH);
            }
          }
        }

        function renderHud(now) {
          const madness = clamp(state.collected / state.targetEnd, 0, 1);
          const phaseText = `phase ${state.phase + 1}/5`;
          const shardText = `shards ${state.collected}/${state.targetEnd}`;
          const lucidityText = `lucidity ${(1 - madness).toFixed(2)}`;
          const soundText = `sound ${state.audioMuted ? "off" : "on"}`;
          const scramble = state.phase >= 2 ? 0.08 + madness * 0.25 : 0;
          hud.textContent = scrambleText(
            `${phaseText}\n${shardText}\n${lucidityText}\n${soundText}`,
            scramble
          );

          const baseMessage =
            state.messageUntil > now ? state.message : phaseInstructions[state.phase] || phaseInstructions[0];
          const messageScramble = state.phase >= 3 ? 0.1 + madness * 0.35 : 0;
          messageEl.textContent = scrambleText(baseMessage, messageScramble);
        }

        function loop(now) {
          if (!state.started) return;
          if (!state.lastTime) state.lastTime = now;
          const dt = Math.min(0.05, (now - state.lastTime) / 1000);
          state.lastTime = now;
          update(dt, now);
          draw(now);
          renderHud(now);
          requestAnimationFrame(loop);
        }

        function startGame() {
          if (state.started) return;
          state.started = true;
          state.startTime = performance.now();
          state.lastTime = state.startTime;
          startOverlay.classList.add("hidden");
          if (!audio.ctx) setupAudio();
          resetGame();
          requestAnimationFrame(loop);
        }

        window.addEventListener("resize", () => {
          resize();
        });

        window.addEventListener("keydown", (event) => {
          const key = event.key.toLowerCase();
          keys[key] = true;
          if (!state.started && (key === "enter" || key === " ")) {
            startGame();
          }
          if (key === "m") {
            state.audioMuted = !state.audioMuted;
          }
          if (key === "r" && state.started) {
            resetGame();
          }
        });

        window.addEventListener("keyup", (event) => {
          keys[event.key.toLowerCase()] = false;
        });

        canvas.addEventListener("pointerdown", (event) => {
          pointer.active = true;
          pointer.x = event.clientX;
          pointer.y = event.clientY;
        });

        canvas.addEventListener("pointermove", (event) => {
          if (!pointer.active) return;
          pointer.x = event.clientX;
          pointer.y = event.clientY;
        });

        window.addEventListener("pointerup", () => {
          pointer.active = false;
        });

        startButton.addEventListener("click", () => {
          startGame();
        });

        startOverlay.addEventListener("click", (event) => {
          if (event.target === startOverlay) {
            startGame();
          }
        });

        resize();
      })();
    </script>
  </body>
</html>
