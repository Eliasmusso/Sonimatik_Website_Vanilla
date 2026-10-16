<script lang="ts">
  let canvas: HTMLCanvasElement;

  const angle = -24;
  const lines = 190;
  const opacity = 0.9;
  const fadeStart = 0.25;
  const fadeEnd = 0.85;
  const fadedOpacity = 0.16;

  const TAU = Math.PI * 2;

  const STOPS: ReadonlyArray<readonly [number, number, number, number]> = [
    [0.0, 29, 46, 77],
    [0.34, 92, 115, 153],
    [0.68, 82, 153, 181],
    [1.0, 170, 207, 223],
  ];

  const BUCKETS = 16;

  function paletteAt(t: number): [number, number, number] {
    const x = Math.min(1, Math.max(0, t));
    for (let i = 0; i < STOPS.length - 1; i++) {
      const a = STOPS[i]!;
      const b = STOPS[i + 1]!;
      if (x <= b[0]) {
        const k = (x - a[0]) / (b[0] - a[0] || 1);
        return [
          Math.round(a[1] + (b[1] - a[1]) * k),
          Math.round(a[2] + (b[2] - a[2]) * k),
          Math.round(a[3] + (b[3] - a[3]) * k),
        ];
      }
    }
    const last = STOPS[STOPS.length - 1]!;
    return [last[1], last[2], last[3]];
  }

  $effect(() => {
    const ctx = canvas.getContext('2d', { alpha: true });
    if (!ctx) return;

    const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)');

    let cssW = 0;
    let cssH = 0;
    let spanU = 0;
    let spanV = 0;
    let count = 0;
    let gapV = 0;
    let step = 0;
    let samples = 0;
    let bufA = new Float32Array(0);
    let bufB = new Float32Array(0);

    const theta = (angle * Math.PI) / 180;
    const cos = Math.abs(Math.cos(theta));
    const sin = Math.abs(Math.sin(theta));
    const cos1 = Math.cos(theta);
    const sin1 = Math.sin(theta);

    const strokeColors: string[] = [];
    const strokeWidths: number[] = [];
    for (let b = 0; b < BUCKETS; b++) {
      const t = b / (BUCKETS - 1);
      const [r, g, bl] = paletteAt(t);
      const a = 0.04 + 0.66 * Math.pow(t, 1.35);
      strokeColors.push(`rgba(${r},${g},${bl},${(a * opacity).toFixed(3)})`);
      strokeWidths.push(0.5 + 1.0 * t);
    }

    let scrollPhase = 0;
    let fade = 0;
    let introStart = 0;

    let ptrTargetX = -1e9;
    let ptrTargetY = -1e9;
    let ptrU = 0;
    let ptrV = 0;
    let ptrStrength = 0;
    let ptrSeen = false;

    interface Ripple {
      u: number;
      v: number;
      born: number;
    }
    let ripples: Ripple[] = [];

    const RIPPLE_LIFE = 1.5;
    let glowRadius = 1;
    let rippleSpeed = 1;

    let warpAmp = 0;
    let amp1 = 0;
    let amp2 = 0;
    let amp3 = 0;

    function disp(un: number, vn: number, t: number, sp: number): number {
      const warp =
        warpAmp *
        Math.sin(TAU * (1.15 * un + 0.35 * vn) + t * 0.11 + sp * 0.9) *
        Math.cos(TAU * (0.62 * vn - 0.28 * un) + t * 0.07);

      const s1 = amp1 * Math.sin(TAU * (2.3 * un + 0.85 * vn) + t * 0.2 + sp * 0.4);
      const s2 = amp2 * Math.sin(TAU * (3.7 * un - 1.42 * vn) + t * 0.14 + 1.7);
      const s3 = amp3 * Math.sin(TAU * (6.1 * un + 2.35 * vn) + t * 0.26 + 4.1);

      return warp + s1 + s2 + s3;
    }

    function resize() {
      cssW = window.innerWidth;
      cssH = window.innerHeight;

      const dpr = Math.min(window.devicePixelRatio || 1, 2);
      canvas.width = Math.round(cssW * dpr);
      canvas.height = Math.round(cssH * dpr);
      canvas.style.width = `${cssW}px`;
      canvas.style.height = `${cssH}px`;
      ctx!.setTransform(dpr, 0, 0, dpr, 0, 0);

      spanU = cssW * cos + cssH * sin;
      spanV = cssW * sin + cssH * cos;

      count = cssW < 640 ? Math.round(lines * 0.55) : lines;
      gapV = spanV / count;

      step = cssW < 640 ? 18 : 13;
      samples = Math.max(2, Math.ceil(spanU / step) + 1);

      if (bufA.length !== samples) {
        bufA = new Float32Array(samples);
        bufB = new Float32Array(samples);
      }

      warpAmp = spanV * 0.042;
      amp1 = spanV * 0.024;
      amp2 = spanV * 0.013;
      amp3 = spanV * 0.006;

      glowRadius = Math.min(cssW, cssH) * 0.26;
      rippleSpeed = Math.max(cssW, cssH) * 0.85;
    }

    function fillDisp(target: Float32Array, v: number, t: number, sp: number) {
      const vn = v / spanV;
      for (let j = 0; j < samples; j++) {
        const un = (j * step) / spanU - 0.5;
        target[j] = disp(un, vn, t, sp);
      }
    }

    function draw(t: number) {
      const sp = scrollPhase;

      if (ptrSeen) {
        const dx = ptrTargetX - cssW / 2;
        const dy = ptrTargetY - cssH / 2;
        const tu = dx * cos1 + dy * sin1;
        const tv = -dx * sin1 + dy * cos1;
        ptrU += (tu - ptrU) * 0.22;
        ptrV += (tv - ptrV) * 0.22;
      }

      const live = ripples.filter((r) => t - r.born < RIPPLE_LIFE);
      ripples = live;

      const hasGlow = ptrStrength > 0.01;
      const hasRipple = live.length > 0;

      const rip = live.map((r) => {
        const age = (t - r.born) / RIPPLE_LIFE;
        return {
          u: r.u,
          v: r.v,
          radius: age * RIPPLE_LIFE * rippleSpeed,
          width: glowRadius * 0.34,
          amp: (1 - age) * (1 - age) * 0.95,
        };
      });

      const glowR2 = glowRadius * glowRadius;

      ctx!.clearRect(0, 0, cssW, cssH);
      ctx!.save();
      ctx!.translate(cssW / 2, cssH / 2);
      ctx!.rotate(theta);

      const paths: Path2D[] = [];
      for (let b = 0; b < BUCKETS; b++) paths.push(new Path2D());

      const ridgeV = spanV * 0.4 * Math.sin(t * 0.045 + sp * 0.55);
      const ridgeW = spanV * 0.3;

      const v0 = -spanV / 2;
      let cur = bufA;
      let next = bufB;
      fillDisp(cur, v0, t, sp);

      for (let i = 0; i < count; i++) {
        const v = v0 + i * gapV;
        fillDisp(next, v + gapV, t, sp);

        const grad = 0.58 + 0.42 * (count > 1 ? i / (count - 1) : 0.5);

        let bucket = -1;
        let px = 0;
        let py = 0;

        for (let j = 0; j < samples; j++) {
          const u = j * step - spanU / 2;
          const un = (j * step) / spanU - 0.5;
          const d = cur[j]!;
          const y = v + d;

          const local = gapV + next[j]! - d;
          const comp = gapV / Math.max(Math.abs(local), 0.0001);
          const cNorm = Math.min(1, Math.max(0, (comp - 0.7) / 1.1));

          const rv = ridgeV + spanV * 0.1 * Math.sin(TAU * 0.9 * un + t * 0.09);
          const dr = (v - rv) / ridgeW;
          const ridge = Math.exp(-dr * dr);

          let inten = 0.34 + (0.44 + 0.56 * ridge) * cNorm * grad;

          if (hasGlow || hasRipple) {
            const du = u - ptrU;
            const dv = y - ptrV;

            if (hasGlow) {
              const d2 = du * du + dv * dv;
              if (d2 < glowR2 * 4) {
                const g = Math.exp(-d2 / (2 * glowR2)) * ptrStrength;
                inten += g * 0.6 * (0.35 + 0.65 * cNorm);
              }
            }

            for (let k = 0; k < rip.length; k++) {
              const r = rip[k]!;
              const ru = u - r.u;
              const rv = y - r.v;
              const dist = Math.sqrt(ru * ru + rv * rv);
              const off = (dist - r.radius) / r.width;
              if (off > -3 && off < 3) {
                inten += Math.exp(-off * off) * r.amp * (0.4 + 0.6 * cNorm);
              }
            }
          }

          if (inten > 1) inten = 1;
          const b = Math.min(BUCKETS - 1, (inten * BUCKETS) | 0);

          if (j === 0) {
            bucket = b;
            paths[b]!.moveTo(u, y);
          } else if (b === bucket) {
            paths[b]!.lineTo(u, y);
          } else {
            paths[b]!.moveTo(px, py);
            paths[b]!.lineTo(u, y);
            bucket = b;
          }
          px = u;
          py = y;
        }

        const swap = cur;
        cur = next;
        next = swap;
      }

      for (let b = 0; b < BUCKETS; b++) {
        ctx!.strokeStyle = strokeColors[b]!;
        ctx!.lineWidth = strokeWidths[b]!;
        ctx!.stroke(paths[b]!);
      }

      ctx!.restore();

      const veil = ctx!.createLinearGradient(0, 0, 0, cssH);
      veil.addColorStop(0, 'rgba(8,14,28,0.10)');
      veil.addColorStop(0.18, 'rgba(8,14,28,0.18)');
      veil.addColorStop(0.36, 'rgba(8,14,28,0.52)');
      veil.addColorStop(0.62, 'rgba(8,14,28,0.66)');
      veil.addColorStop(1, 'rgba(8,14,28,0.74)');
      ctx!.fillStyle = veil;
      ctx!.fillRect(0, 0, cssW, cssH);
    }

    let raf = 0;
    let last = 0;
    let running = false;

    const FRAME_IDLE = 1000 / 30;
    const FRAME_ACTIVE = 1000 / 60;

    function loop(now: number) {
      raf = requestAnimationFrame(loop);

      const interactive = ptrStrength > 0.01 || ripples.length > 0;
      if (now - last < (interactive ? FRAME_ACTIVE : FRAME_IDLE)) return;
      const dt = last === 0 ? 0.016 : Math.min((now - last) / 1000, 0.1);
      last = now;

      const target = ptrSeen ? 1 : 0;
      ptrStrength += (target - ptrStrength) * Math.min(1, dt * 6);

      scrollPhase = window.scrollY / Math.max(window.innerHeight, 1);

      const fadeT = Math.min(1, Math.max(0, (scrollPhase - fadeStart) / (fadeEnd - fadeStart)));
      const fadeEased = fadeT * fadeT * (3 - 2 * fadeT);
      const fadeTarget = 1 - (1 - fadedOpacity) * fadeEased;

      if (introStart === 0) introStart = now;
      const intro = Math.min(1, (now - introStart) / 900);

      fade += (fadeTarget * intro - fade) * Math.min(1, dt * 8);
      canvas.style.opacity = fade.toFixed(3);

      draw(now / 1000);
    }

    function start() {
      if (running || reduceMotion.matches) return;
      running = true;
      last = 0;
      raf = requestAnimationFrame(loop);
    }

    function stop() {
      running = false;
      cancelAnimationFrame(raf);
    }

    function renderStatic() {
      scrollPhase = 0;
      draw(0);
    }

    function applyFadeStatic() {
      const sp = window.scrollY / Math.max(window.innerHeight, 1);
      const t = Math.min(1, Math.max(0, (sp - fadeStart) / (fadeEnd - fadeStart)));
      const eased = t * t * (3 - 2 * t);
      fade = 1 - (1 - fadedOpacity) * eased;
      canvas.style.opacity = fade.toFixed(3);
    }

    let lastW = 0;
    let lastH = 0;
    let resizeTimer = 0;

    function onResize() {
      const w = window.innerWidth;
      const h = window.innerHeight;
      if (w === lastW && Math.abs(h - lastH) < 120) return;
      lastW = w;
      lastH = h;

      clearTimeout(resizeTimer);
      resizeTimer = window.setTimeout(() => {
        resize();
        if (reduceMotion.matches) renderStatic();
      }, 150);
    }

    function onVisibility() {
      if (document.hidden) {
        stop();
        return;
      }
      if (window.innerWidth !== lastW || window.innerHeight !== lastH) {
        lastW = window.innerWidth;
        lastH = window.innerHeight;
        resize();
      }
      if (reduceMotion.matches) renderStatic();
      else start();
    }

    function onMotionChange() {
      if (reduceMotion.matches) {
        stop();
        renderStatic();
      } else {
        start();
      }
    }

    function onPointerMove(event: PointerEvent) {
      if (event.pointerType === 'touch') return;
      ptrTargetX = event.clientX;
      ptrTargetY = event.clientY;
      if (!ptrSeen) {
        const dx = ptrTargetX - cssW / 2;
        const dy = ptrTargetY - cssH / 2;
        ptrU = dx * cos1 + dy * sin1;
        ptrV = -dx * sin1 + dy * cos1;
      }
      ptrSeen = true;
    }

    function onPointerLeave() {
      ptrSeen = false;
    }

    function onPointerDown(event: PointerEvent) {
      const dx = event.clientX - cssW / 2;
      const dy = event.clientY - cssH / 2;
      ripples.push({
        u: dx * cos1 + dy * sin1,
        v: -dx * sin1 + dy * cos1,
        born: performance.now() / 1000,
      });
      if (ripples.length > 3) ripples.shift();
    }

    lastW = window.innerWidth;
    lastH = window.innerHeight;
    resize();

    if (reduceMotion.matches) renderStatic();
    else start();

    window.addEventListener('resize', onResize, { passive: true });
    window.addEventListener('scroll', applyFadeStatic, { passive: true });
    applyFadeStatic();
    document.addEventListener('visibilitychange', onVisibility);
    reduceMotion.addEventListener('change', onMotionChange);

    if (!reduceMotion.matches) {
      window.addEventListener('pointermove', onPointerMove, { passive: true });
      window.addEventListener('pointerdown', onPointerDown, { passive: true });
      document.addEventListener('pointerleave', onPointerLeave);
      window.addEventListener('blur', onPointerLeave);
    }

    return () => {
      stop();
      clearTimeout(resizeTimer);
      window.removeEventListener('resize', onResize);
      window.removeEventListener('scroll', applyFadeStatic);
      document.removeEventListener('visibilitychange', onVisibility);
      reduceMotion.removeEventListener('change', onMotionChange);
      window.removeEventListener('pointermove', onPointerMove);
      window.removeEventListener('pointerdown', onPointerDown);
      document.removeEventListener('pointerleave', onPointerLeave);
      window.removeEventListener('blur', onPointerLeave);
    };
  });
</script>

<canvas bind:this={canvas} class="fixed inset-0 -z-10 block w-full h-full pointer-events-none opacity-0" aria-hidden="true"></canvas>
