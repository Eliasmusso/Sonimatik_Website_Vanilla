<script lang="ts">
  // TODO: Point BACKEND_URL to the actual PHP endpoint once the backend is set up.
  const BACKEND_URL = '';

  const contactSubjects = [
    { value: 'mcemm', label: 'Anfrage McEmm' },
    { value: 'zentralfakturierung', label: 'Zentralfakturierung' },
    { value: 'soniprotect', label: 'SoniProtect' },
    { value: 'schnittstellen', label: 'Schnittstellen & Integration' },
    { value: 'sicherheit', label: 'Informationssicherheit & Datenschutz' },
    { value: 'allgemein', label: 'Allgemeine Anfrage' },
    { value: 'karriere', label: 'Karriere' },
  ] as const;

  const fields = [
    { name: 'name', label: 'Name', type: 'text', required: true, autocomplete: 'name' },
    { name: 'company', label: 'Firma', type: 'text', required: false, autocomplete: 'organization' },
    { name: 'email', label: 'E Mail', type: 'email', required: true, autocomplete: 'email' },
    { name: 'phone', label: 'Telefon', type: 'tel', required: false, autocomplete: 'tel' },
  ] as const;

  type FieldKey = 'subject' | 'name' | 'company' | 'email' | 'phone' | 'message' | 'consent';
  type FieldErrors = Partial<Record<FieldKey, string>>;

  let status = $state<'idle' | 'sending' | 'sent' | 'error'>('idle');
  let formMessage = $state('');
  let errors = $state<FieldErrors>({});
  let formEl = $state<HTMLFormElement | null>(null);

  const renderedAt = Date.now();

  const fromUrl =
    typeof location !== 'undefined' ? new URLSearchParams(location.search).get('betreff') : null;
  const initialSubject = contactSubjects.some((s) => s.value === fromUrl)
    ? (fromUrl as string)
    : 'allgemein';

  function collect() {
    const fd = formEl ? new FormData(formEl) : new FormData();
    return {
      name: String(fd.get('name') ?? '').trim(),
      company: String(fd.get('company') ?? '').trim(),
      email: String(fd.get('email') ?? '').trim(),
      phone: String(fd.get('phone') ?? '').trim(),
      subject: String(fd.get('subject') ?? '').trim(),
      message: String(fd.get('message') ?? '').trim(),
      consent: fd.get('consent') === 'on',
      website: String(fd.get('website') ?? ''),
      renderedAt,
    };
  }

  function validateAll(data = collect()): FieldErrors {
    const out: FieldErrors = {};
    if (!contactSubjects.some((s) => s.value === data.subject)) {
      out.subject = 'Bitte wählen Sie einen Betreff.';
    }
    if (data.name.length < 2) {
      out.name = 'Bitte geben Sie Ihren Namen an.';
    } else if (data.name.length > 120) {
      out.name = 'Der Name ist zu lang.';
    }
    if (data.company.length > 160) {
      out.company = 'Der Firmenname ist zu lang.';
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email) || data.email.length > 200) {
      out.email = 'Bitte geben Sie eine gültige E Mail Adresse an.';
    }
    if (data.phone.length > 60) {
      out.phone = 'Die Telefonnummer ist zu lang.';
    }
    if (data.message.length < 20) {
      out.message = 'Bitte beschreiben Sie Ihr Anliegen in mindestens 20 Zeichen.';
    } else if (data.message.length > 5000) {
      out.message = 'Die Nachricht ist zu lang (maximal 5000 Zeichen).';
    }
    if (!data.consent) {
      out.consent = 'Bitte bestätigen Sie die Verarbeitung Ihrer Angaben.';
    }
    return out;
  }

  function validateField(name: FieldKey) {
    const all = validateAll();
    errors = { ...errors, [name]: all[name] };
  }

  async function onSubmit(event: SubmitEvent) {
    event.preventDefault();
    if (status === 'sending') return;

    const payload = collect();
    const nextErrors = validateAll(payload);

    if (Object.keys(nextErrors).length > 0) {
      errors = nextErrors;
      status = 'error';
      formMessage = 'Bitte prüfen Sie die markierten Felder.';
      return;
    }

    status = 'sending';
    formMessage = '';
    errors = {};

    // TODO: Connect contact form submission to PHP backend once BACKEND_URL is configured.
    if (!BACKEND_URL) {
      status = 'sent';
      formMessage = 'Vielen Dank. Ihre Anfrage ist eingegangen, wir melden uns zurück.';
      formEl?.reset();
      return;
    }

    try {
      const response = await fetch(`${BACKEND_URL}/kontakt.php`, {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify(payload),
      });

      const body = (await response.json().catch(() => ({}))) as {
        ok?: boolean;
        message?: string;
        errors?: FieldErrors;
      };

      if (response.ok && body.ok) {
        status = 'sent';
        formMessage = 'Vielen Dank. Ihre Anfrage ist eingegangen, wir melden uns zurück.';
        formEl?.reset();
        return;
      }

      status = 'error';
      errors = body.errors ?? {};
      formMessage =
        body.message ?? 'Die Nachricht konnte nicht versendet werden. Bitte versuchen Sie es erneut.';
    } catch {
      status = 'error';
      formMessage =
        'Verbindung fehlgeschlagen. Bitte prüfen Sie Ihre Internetverbindung und versuchen Sie es erneut.';
    }
  }
</script>

{#if status === 'sent'}
  <div class="flex flex-col items-start gap-3 py-4" role="status">
    <span class="w-9 h-[3px] rounded-full bg-success-300 mb-2" aria-hidden="true"></span>
    <h3 class="m-0 font-display font-bold text-h3 tracking-[-0.012em] text-white">
      Anfrage eingegangen
    </h3>
    <p class="m-0 text-grau-300 max-w-[46ch]">{formMessage}</p>
    <button
      class="mt-2 p-0 bg-transparent border-0 border-b border-brand-300/40 text-brand-300 font-display font-medium text-[0.9375rem] cursor-pointer hover:border-brand-300"
      type="button"
      onclick={() => {
        status = 'idle';
        formMessage = '';
      }}
    >
      Weitere Anfrage senden
    </button>
  </div>
{:else}
  <form bind:this={formEl} class="flex flex-col gap-6" novalidate onsubmit={onSubmit}>
    <div class="grid grid-cols-1 sm:grid-cols-2 gap-5">
      {#each fields as field (field.name)}
        <div class="flex flex-col gap-2">
          <label
            class="flex items-baseline gap-2 font-display font-medium text-[0.8125rem] tracking-[0.06em] uppercase text-grau-300"
            for={`cf-${field.name}`}
          >
            {field.label}
            {#if !field.required}
              <span class="font-sans font-normal text-xs tracking-normal normal-case text-grau-300"
                >optional</span
              >
            {/if}
          </label>
          <input
            class="w-full py-[0.8125rem] px-[0.9375rem] rounded-control border bg-navy-900 text-white font-sans text-base leading-[1.5] transition-colors duration-200 hover:border-white/20 focus:outline-none focus:border-brand-300 {errors[
              field.name
            ]
              ? 'border-error-300'
              : 'border-white/12'}"
            id={`cf-${field.name}`}
            name={field.name}
            type={field.type}
            autocomplete={field.autocomplete}
            required={field.required}
            onblur={() => validateField(field.name)}
          />
          {#if errors[field.name]}
            <p class="m-0 flex items-center gap-1.5 text-[0.8125rem] text-error-300">
              {errors[field.name]}
            </p>
          {/if}
        </div>
      {/each}

      <div class="flex flex-col gap-2 sm:col-span-2">
        <label
          class="flex items-baseline gap-2 font-display font-medium text-[0.8125rem] tracking-[0.06em] uppercase text-grau-300"
          for="cf-subject"
        >
          Betreff
        </label>
        <div class="relative flex">
          <select
            class="w-full py-[0.8125rem] pl-[0.9375rem] pr-10 rounded-control border bg-navy-900 text-white font-sans text-base leading-[1.5] appearance-none cursor-pointer transition-colors duration-200 hover:border-white/20 focus:outline-none focus:border-brand-300 {errors.subject
              ? 'border-error-300'
              : 'border-white/12'}"
            id="cf-subject"
            name="subject"
            required
            onblur={() => validateField('subject')}
          >
            {#each contactSubjects as s (s.value)}
              <option class="bg-navy-800 text-white" value={s.value} selected={s.value === initialSubject}>
                {s.label}
              </option>
            {/each}
          </select>
          <svg
            class="absolute right-[0.9375rem] top-1/2 -translate-y-1/2 pointer-events-none text-brand-300"
            viewBox="0 0 16 16"
            fill="none"
            aria-hidden="true"
            width="14"
            height="14"
          >
            <path
              d="M4 6.5 8 10.5 12 6.5"
              stroke="currentColor"
              stroke-width="1.5"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        </div>
        {#if errors.subject}
          <p class="m-0 flex items-center gap-1.5 text-[0.8125rem] text-error-300">
            {errors.subject}
          </p>
        {/if}
      </div>

      <div class="flex flex-col gap-2 sm:col-span-2">
        <label
          class="flex items-baseline gap-2 font-display font-medium text-[0.8125rem] tracking-[0.06em] uppercase text-grau-300"
          for="cf-message"
        >
          Ihr Anliegen
        </label>
        <textarea
          class="w-full py-[0.8125rem] px-[0.9375rem] rounded-control border bg-navy-900 text-white font-sans text-base leading-[1.5] resize-y min-h-32 transition-colors duration-200 hover:border-white/20 focus:outline-none focus:border-brand-300 {errors.message
            ? 'border-error-300'
            : 'border-white/12'}"
          id="cf-message"
          name="message"
          rows="6"
          required
          onblur={() => validateField('message')}
        ></textarea>
        {#if errors.message}
          <p class="m-0 flex items-center gap-1.5 text-[0.8125rem] text-error-300">
            {errors.message}
          </p>
        {/if}
      </div>
    </div>

    <div class="absolute -left-[9999px] w-px h-px overflow-hidden" aria-hidden="true">
      <label for="cf-website">Website</label>
      <input id="cf-website" name="website" type="text" tabindex="-1" autocomplete="off" />
    </div>

    <input type="hidden" name="renderedAt" value={renderedAt} />

    <div class="flex items-start gap-3">
      <input
        class="shrink-0 w-[1.125rem] h-[1.125rem] mt-1 accent-brand-300 cursor-pointer"
        id="cf-consent"
        name="consent"
        type="checkbox"
        required
      />
      <label class="text-sm leading-[1.6] text-grau-300 cursor-pointer" for="cf-consent">
        Ich bin damit einverstanden, dass meine Angaben zur Bearbeitung meiner Anfrage verarbeitet
        werden. Weitere Hinweise in der
        <a href="/datenschutz" class="text-brand-300 underline decoration-brand-300/40 underline-offset-4">
          Datenschutzerklärung
        </a>.
      </label>
    </div>
    {#if errors.consent}
      <p class="m-0 flex items-center gap-1.5 text-[0.8125rem] text-error-300">{errors.consent}</p>
    {/if}

    <div class="flex flex-wrap items-center gap-y-4 gap-x-6 pt-1">
      <button
        class="inline-flex items-center gap-2.5 py-[0.9375rem] px-6 border-0 rounded-control bg-brand-300 text-navy-900 font-display font-bold text-[0.9375rem] cursor-pointer transition-colors duration-200 hover:bg-brand-200 disabled:opacity-60 disabled:cursor-progress"
        type="submit"
        disabled={status === 'sending'}
      >
        {status === 'sending' ? 'Wird gesendet …' : 'Anfrage senden'}
        {#if status !== 'sending'}
          <svg viewBox="0 0 16 16" width="16" height="16" fill="none" aria-hidden="true">
            <path
              d="M2.5 8h10M9 4.5 12.5 8 9 11.5"
              stroke="currentColor"
              stroke-width="1.5"
              stroke-linecap="round"
              stroke-linejoin="round"
            />
          </svg>
        {/if}
      </button>
      <p class="flex-[1_1_16rem] m-0 text-[0.8125rem] leading-[1.6] text-grau-300 max-w-[44ch]">
        Ihre Angaben werden ausschließlich zur Bearbeitung Ihrer Anfrage verwendet und nicht an
        Dritte weitergegeben.
      </p>
    </div>

    <p
      class="m-0 text-sm min-h-[1.4em] {status === 'error' ? 'text-error-300' : 'text-grau-300'}"
      role="status"
      aria-live="polite"
    >
      {formMessage}
    </p>
  </form>
{/if}
