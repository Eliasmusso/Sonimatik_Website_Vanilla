<script lang="ts">
  interface Item {
    label: string;
    href: string;
  }

  interface Props {
    items: Item[];
    ctaLabel: string;
    ctaHref: string;
  }

  let { items, ctaLabel, ctaHref }: Props = $props();

  let open = $state(false);

  function close() {
    open = false;
  }

  function onKeydown(event: KeyboardEvent) {
    if (open && event.key === 'Escape') {
      close();
    }
  }

  $effect(() => {
    document.documentElement.style.overflow = open ? 'hidden' : '';
    return () => {
      document.documentElement.style.overflow = '';
    };
  });
</script>

<svelte:window onkeydown={onKeydown} />

<button
  class="inline-flex items-center justify-center w-11 h-11 -mr-2 bg-transparent border-0 cursor-pointer relative z-60"
  type="button"
  aria-label={open ? 'Navigation schließen' : 'Navigation öffnen'}
  onclick={() => (open = !open)}
>
  <span class="flex flex-col gap-[5px] w-[1.375rem]" aria-hidden="true">
    <span
      class="block h-[2px] rounded-full bg-white transition-all duration-260"
      class:w-full={true}
      class:translate-y-[7px]={open}
      class:rotate-45={open}
    ></span>
    <span
      class="block h-[2px] rounded-full bg-white transition-all duration-200"
      class:w-[70%]={!open}
      class:w-full={open}
      class:opacity-0={open}
    ></span>
    <span
      class="block h-[2px] rounded-full bg-white transition-all duration-260"
      class:w-[85%]={!open}
      class:w-full={open}
      class:-translate-y-[7px]={open}
      class:-rotate-45={open}
    ></span>
  </span>
</button>

{#if open}
  <button
    type="button"
    class="fixed inset-0 z-40 block w-full h-full p-0 border-0 cursor-pointer bg-navy-900/72 backdrop-blur-[3px] animate-[nav-fade_220ms_var(--ease-brand)_both]"
    aria-label="Navigation schließen"
    tabindex="-1"
    onclick={close}
  ></button>

  <div
    class="fixed inset-y-0 right-0 z-50 w-[min(22rem,88vw)] flex flex-col justify-center gap-10 pt-20 px-7 pb-10 bg-navy-800 animate-[nav-slide_300ms_var(--ease-brand)_both] overflow-y-auto"
  >
    <nav>
      <ul class="flex flex-col gap-1 list-none m-0 p-0">
        {#each items as item, i (item.href)}
          <li>
            <a
              href={item.href}
              onclick={close}
              class="group flex items-center gap-3.5 py-3 font-display font-bold text-[1.375rem] tracking-[-0.02em] text-white no-underline border-b border-white/8 animate-[nav-item_320ms_var(--ease-brand)_both]"
              style={`animation-delay:${120 + i * 45}ms`}
            >
              <span
                class="shrink-0 w-3.5 h-[2px] rounded-full bg-brand-300 origin-left transition-transform duration-220 group-hover:scale-x-170"
                aria-hidden="true"
              ></span>
              {item.label}
            </a>
          </li>
        {/each}
      </ul>
    </nav>

    <a
      class="inline-flex items-center justify-center py-[0.9375rem] px-5 rounded-control bg-brand-300 text-navy-900 font-display font-bold text-[0.9375rem] no-underline"
      href={ctaHref}
      onclick={close}
    >
      {ctaLabel}
    </a>
  </div>
{/if}
