<script lang="ts">
  import { glyphs, type GlyphName } from './glyphs';

  interface Node {
    id: string;
    label: string;
    caption: string;
    icon: GlyphName;
    x: number;
    y: number;
  }

  const nodes: Node[] = [
    { id: 'betrieb', label: 'Fachbetrieb', caption: 'Erfassung im Tagesgeschäft', icon: 'shop', x: 90, y: 90 },
    { id: 'kasse', label: 'Kassensystem', caption: 'Bestand und Verkauf', icon: 'pos', x: 90, y: 290 },
    { id: 'schnittstelle', label: 'Schnittstelle', caption: 'Verschlüsselte Übergabe', icon: 'interface', x: 320, y: 190 },
    { id: 'plattform', label: 'Plattform', caption: 'Zentrale Datenhaltung', icon: 'platform', x: 550, y: 190 },
    { id: 'automatisierung', label: 'Automatisierung', caption: 'Wiederkehrende Prozesse', icon: 'automation', x: 790, y: 90 },
    { id: 'auswertung', label: 'Auswertung', caption: 'Kennzahlen im Zugriff', icon: 'chart', x: 790, y: 290 },
    { id: 'abrechnung', label: 'Abrechnung', caption: 'Übergabe an Kostenträger', icon: 'invoice', x: 1020, y: 90 },
  ];

  const connections = [
    { from: 'betrieb', to: 'schnittstelle' },
    { from: 'kasse', to: 'schnittstelle' },
    { from: 'schnittstelle', to: 'plattform' },
    { from: 'plattform', to: 'automatisierung' },
    { from: 'plattform', to: 'auswertung' },
    { from: 'automatisierung', to: 'abrechnung' },
  ];

  const NODE_W = 152;
  const NODE_H = 62;
  const CORNER = 12;
  const PAD = 24;

  const byId = new Map(nodes.map((n) => [n.id, n]));

  const maxX = Math.max(...nodes.map((n) => n.x)) + NODE_W / 2 + PAD;
  const maxY = Math.max(...nodes.map((n) => n.y)) + NODE_H / 2 + PAD;
  const minX = Math.min(...nodes.map((n) => n.x)) - NODE_W / 2 - PAD;
  const minY = Math.min(...nodes.map((n) => n.y)) - NODE_H / 2 - PAD;
  const viewBox = { minX, minY, w: maxX - minX, h: maxY - minY };

  function tracePath(a: Node, b: Node): string {
    const x1 = a.x + NODE_W / 2;
    const y1 = a.y;
    const x2 = b.x - NODE_W / 2;
    const y2 = b.y;

    if (Math.abs(y1 - y2) < 1) return `M ${x1} ${y1} L ${x2} ${y2}`;

    const mx = x1 + (x2 - x1) / 2;
    const dir = y2 > y1 ? 1 : -1;
    const r = Math.min(CORNER, Math.abs(y2 - y1) / 2, Math.abs(x2 - x1) / 2);

    return [
      `M ${x1} ${y1}`,
      `L ${mx - r} ${y1}`,
      `Q ${mx} ${y1} ${mx} ${y1 + r * dir}`,
      `L ${mx} ${y2 - r * dir}`,
      `Q ${mx} ${y2} ${mx + r} ${y2}`,
      `L ${x2} ${y2}`,
    ].join(' ');
  }

  const traces = connections.map((c) => {
    const a = byId.get(c.from)!;
    const b = byId.get(c.to)!;
    return { ...c, d: tracePath(a, b), key: `${c.from}-${c.to}` };
  });

  let activeNode = $state<string | null>(null);

  function isLit(t: { from: string; to: string }): boolean {
    return activeNode !== null && (t.from === activeNode || t.to === activeNode);
  }
</script>

<div class="text-brand-300">
  <svg
    class="hidden min-[56.25rem]:block w-full h-auto"
    viewBox={`${viewBox.minX} ${viewBox.minY} ${viewBox.w} ${viewBox.h}`}
    aria-hidden="true"
    preserveAspectRatio="xMidYMid meet"
  >
    <defs>
      <pattern id="pb-grid" width="22" height="22" patternUnits="userSpaceOnUse">
        <circle cx="1" cy="1" r="1" fill="currentColor" opacity="0.08" />
      </pattern>
    </defs>

    <rect
      x={viewBox.minX}
      y={viewBox.minY}
      width={viewBox.w}
      height={viewBox.h}
      fill="url(#pb-grid)"
      class="text-brand-300"
    />

    <g fill="none" stroke-linecap="round">
      {#each traces as t (t.key)}
        <path
          d={t.d}
          stroke-width="1.25"
          class="transition-[stroke] duration-260 {isLit(t)
            ? 'stroke-brand-300/50'
            : activeNode !== null
              ? 'stroke-brand-300/10'
              : 'stroke-brand-300/22'}"
        />
        <path
          d={t.d}
          pathLength="100"
          stroke-width="2"
          class="stroke-brand-300 [stroke-dasharray:14_100] animate-[pulse-run_3.4s_linear_infinite] drop-shadow-[0_0_3px_rgb(170_207_223/0.5)] transition-opacity duration-260 {isLit(
            t,
          )
            ? 'opacity-100'
            : activeNode !== null
              ? 'opacity-30'
              : 'opacity-85'}"
        />
      {/each}
    </g>

    <g>
      {#each nodes as n (n.id)}
        <g
          class="cursor-default"
          transform={`translate(${n.x - NODE_W / 2} ${n.y - NODE_H / 2})`}
          onmouseenter={() => (activeNode = n.id)}
          onmouseleave={() => (activeNode = null)}
          role="presentation"
        >
          <rect
            class="transition-[fill] duration-260 {activeNode === n.id
              ? 'fill-navy-500'
              : 'fill-navy-600'}"
            width={NODE_W}
            height={NODE_H}
            rx="10"
            ry="10"
          />
          <g
            class="fill-brand-300 {activeNode === n.id ? 'opacity-100' : 'opacity-85'}"
            transform="translate(16 19) scale(0.09375)"
          >
            {@html glyphs[n.icon]}
          </g>
          <text
            class="fill-white font-display font-bold text-[13px] tracking-[-0.01em]"
            x="16"
            y="52">{n.label}</text
          >
        </g>
      {/each}
    </g>
  </svg>

  <dl class="flex flex-col m-0 min-[56.25rem]:hidden">
    {#each nodes as n (n.id)}
      <div
        class="flex flex-col gap-0.5 py-3.5 border-t border-white/8 relative pl-[1.625rem] before:content-[''] before:absolute before:left-0 before:top-[1.4rem] before:w-3.5 before:h-[2px] before:rounded-full before:bg-brand-300 not-last:after:content-[''] not-last:after:absolute not-last:after:left-1.5 not-last:after:top-[1.8rem] not-last:after:-bottom-2 not-last:after:w-px not-last:after:bg-brand-300/22"
      >
        <dt class="font-display font-bold text-base text-white">{n.label}</dt>
        <dd class="m-0 text-small text-grau-300">{n.caption}</dd>
      </div>
    {/each}
  </dl>
</div>
