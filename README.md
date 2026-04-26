# Svelte 5 + SvelteKit Best Practices

Project conventions and idiomatic patterns for this codebase. When in doubt, follow the patterns here over older Svelte 3/4 tutorials online — most of them are out of date for Svelte 5 (runes) and recent SvelteKit (`resolve()` / `asset()`).

---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

## 1. Always use runes (Svelte 5 reactivity)

This project forces runes mode in `svelte.config.js`. **Don't** use the legacy `let count = 0` + reactive `$:` syntax.

```svelte
<script lang="ts">
	// State
	let count = $state(0);

	// Derived (recomputes when deps change)
	let doubled = $derived(count * 2);

	// Side effects
	$effect(() => {
		console.log('count is now', count);
	});

	// Props
	let { title, children } = $props();
</script>

<button onclick={() => count++}>
	{title}: {count} (×2 = {doubled})
</button>
```

**Avoid:**
- `export let foo` → use `let { foo } = $props()`
- `$: doubled = count * 2` → use `$derived`
- `on:click={...}` → use `onclick={...}` (lowercase, no colon)
- Stores (`writable`/`readable`) for component-local state → use `$state`. Stores are still fine for cross-tree shared state.

---

## 2. Snippets, not slots

Slots are deprecated. Use snippets + `{@render ...}`.

```svelte
<!-- Card.svelte -->
<script lang="ts">
	let { header, children } = $props();
</script>

<article class="card">
	{#if header}
		<header>{@render header()}</header>
	{/if}
	<div class="body">{@render children()}</div>
</article>
```

```svelte
<!-- usage -->
<Card>
	{#snippet header()}
		<h2>Title</h2>
	{/snippet}

	<p>Body content goes into the default `children` snippet.</p>
</Card>
```

---

## 3. Internal links: always `resolve()`

The `svelte/no-navigation-without-resolve` lint rule flags raw string hrefs. Use `resolve()` from `$app/paths` so the base path is applied and the route is type-checked.

```svelte
<script lang="ts">
	import { resolve } from '$app/paths';
</script>

<a href={resolve('/')}>Home</a>
<a href={resolve('/apply')}>Apply</a>

<!-- External, mailto, tel: raw string is fine -->
<a href="https://example.com">External</a>
<a href="mailto:hi@example.com">Email</a>
```

If `resolve('/foo')` errors, the route doesn't exist yet — create `src/routes/foo/+page.svelte`.

---

## 4. Static assets: `asset()` or `import`

Two patterns, pick by location:

```svelte
<script lang="ts">
	// Preferred — Vite hashes the URL, long-term cacheable, base-path safe
	import logo from '$lib/assets/logo.svg';
</script>

<img src={logo} alt="Logo" width="32" height="32" decoding="async" />
```

For files that **must** keep a fixed URL (`robots.txt`, `sitemap.xml`, OG images referenced by external services), keep them in `/static` and use `asset()` (or the `assets` constant if your SvelteKit version is older):

```svelte
<script lang="ts">
	import { asset } from '$app/paths';
</script>

<link rel="manifest" href={asset('/manifest.webmanifest')} />
```

**Rule of thumb:** routes → `resolve()`, static files → `asset()`, anything imported into JS → just `import` it.

---

## 5. Images: prevent CLS + decode off-thread

Always set `width`, `height`, `decoding`, and `loading`:

```svelte
<!-- Above the fold -->
<img src={hero} alt="…" width="1200" height="600" decoding="async" fetchpriority="high" />

<!-- Below the fold -->
<img src={thumb} alt="…" width="400" height="300" decoding="async" loading="lazy" />
```

For raster images, prefer `@sveltejs/enhanced-img` once installed — it auto-generates AVIF/WebP, srcsets, and dimensions from the source file.

---

## 6. Styles: use the design system

All design tokens live in `src/lib/styles/_variables.scss` and are exposed both as SCSS variables and as CSS custom properties on `:root`. **Never hard-code** colors, spacings, font sizes, radii, or shadows.

```svelte
<style lang="scss">
	@use '$styles/variables' as *;
	@use '$styles/mixins' as *;

	.card {
		background: $color-white;
		color: $color-fg;
		padding: $space-5;
		border-radius: $radius-lg;
		box-shadow: $shadow-md;
		transition: transform $transition-base;

		@include breakpoint-up($bp-md) {
			padding: $space-6;
		}
	}

	.title {
		@include font-bold;
		font-size: $font-size-2xl;
		letter-spacing: $letter-spacing-tight;
	}

	.subtitle {
		@include font-regular-italic;
		color: $color-muted;
	}
</style>
```

In markup, the `font-{weight}[-italic]` utility classes are also available globally:

```svelte
<h1 class="font-black">Heading</h1>
<p class="font-light-italic">Subtitle</p>
```

When a value needs to be dynamic at runtime (e.g. theme switch), use the CSS variable (`var(--color-bg)`); when it's static at build time, prefer the SCSS variable (`$color-bg`).

---

## 7. Component-scoped CSS by default

Svelte scopes `<style>` to the component automatically. Don't reach for global styles unless you mean to:

```svelte
<style lang="scss">
	/* scoped — only this component */
	.btn { padding: $space-3 $space-5; }

	/* opt out per-selector when you must */
	:global(.markdown-body p) { margin-bottom: $space-4; }
</style>
```

Truly global styles belong in `src/lib/styles/app.scss`.

---

## 8. Data loading: `+page.ts` / `+page.server.ts`

Don't `fetch` inside `onMount` for page data. Use a `load` function — it runs on the server first, then hydrates, and the data is available on first paint.

```ts
// src/routes/posts/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ fetch, params }) => {
	const res = await fetch(`/api/posts`);
	return { posts: await res.json() };
};
```

```svelte
<!-- src/routes/posts/+page.svelte -->
<script lang="ts">
	let { data } = $props();
</script>

{#each data.posts as post (post.id)}
	<article>{post.title}</article>
{/each}
```

Use `+page.server.ts` (server-only) when the loader needs secrets, DB access, or private APIs. Use `+page.ts` (universal) when it can run anywhere.

---

## 9. Forms: use actions, not custom fetch

SvelteKit form actions give you progressive enhancement for free.

```svelte
<!-- src/routes/contact/+page.svelte -->
<script lang="ts">
	import { enhance } from '$app/forms';
	let { form } = $props();
</script>

<form method="POST" use:enhance>
	<input name="email" type="email" required />
	<button type="submit">Send</button>
	{#if form?.error}<p class="error">{form.error}</p>{/if}
</form>
```

```ts
// src/routes/contact/+page.server.ts
import type { Actions } from './$types';

export const actions: Actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const email = data.get('email');
		if (!email) return { error: 'Email is required' };
		// …send it…
		return { success: true };
	}
};
```

The form works without JS; `use:enhance` upgrades it to AJAX when JS is available.

---

## 10. Keyed `{#each}` blocks

Always provide a key when iterating over data that can change:

```svelte
<!-- Good: keyed by stable id -->
{#each users as user (user.id)}
	<UserRow {user} />
{/each}

<!-- Bad: unkeyed; Svelte will reuse DOM nodes incorrectly on reorder -->
{#each users as user}
	<UserRow {user} />
{/each}
```

---

## 11. Accessibility quick wins

- Every `<img>` needs `alt` (use `alt=""` for purely decorative).
- Interactive elements: `<button>` for actions, `<a href>` for navigation. Don't put `onclick` on a `<div>`.
- Respect `prefers-reduced-motion` (already wired up globally in `app.scss`).
- Focus styles: don't `outline: none` without providing an alternative — the `focus-ring` mixin gives you a consistent one.

```svelte
<button class="btn">
	Save
</button>

<style lang="scss">
	@use '$styles/mixins' as *;

	.btn:focus-visible {
		@include focus-ring;
	}
</style>
```

---

## 12. Performance defaults

- **Fonts** are self-hosted via `@fontsource-variable/montserrat` — don't add Google Fonts `<link>` tags.
- **Hover preloading** is on (`data-sveltekit-preload-data="hover"` on `<body>`); pages start loading on link hover.
- **Code splitting** is automatic per-route. Don't manually `import()` page components.
- For large client-only libraries (charts, editors, maps), defer the import:

```svelte
<script lang="ts">
	import { onMount } from 'svelte';
	let chart: any;

	onMount(async () => {
		const { default: Chart } = await import('chart.js/auto');
		chart = new Chart(/* … */);
	});
</script>
```

---

## 13. File layout

```
src/
├── app.html              ← shell; minimal, no per-page content
├── app.d.ts              ← ambient types (App.Locals, App.PageData, etc.)
├── lib/
│   ├── assets/           ← images/fonts that get hashed by Vite
│   ├── components/       ← reusable .svelte files
│   ├── server/           ← server-only modules (never bundled to client)
│   ├── styles/           ← global SCSS (variables, mixins, reset, app)
│   └── index.ts          ← public re-exports for $lib
└── routes/
    ├── +layout.svelte    ← root layout (nav, global styles import)
    ├── +page.svelte
    └── apply/
        └── +page.svelte
```

Anything in `src/lib/server/**` is **statically guaranteed** never to ship to the browser — put DB clients, secrets, and API tokens there.

---

## 14. TypeScript: lean on `./$types`

SvelteKit generates per-route types. Always import from `./$types` instead of redeclaring:

```ts
import type { PageLoad, PageServerLoad, Actions, RequestHandler } from './$types';
```

For component props, type them inline on `$props()`:

```svelte
<script lang="ts">
	type Props = {
		title: string;
		count?: number;
		children: import('svelte').Snippet;
	};

	let { title, count = 0, children }: Props = $props();
</script>
```

---

## 15. What not to do

- ❌ Don't write `.svelte` files with `<script>` lacking `lang="ts"` — this project is TypeScript.
- ❌ Don't add `index.html` files; SvelteKit owns the HTML shell via `app.html`.
- ❌ Don't put route logic in `+layout.svelte` — use `+layout.ts` / `+layout.server.ts`.
- ❌ Don't hard-code colors/spacings/font sizes — use tokens.
- ❌ Don't use `<a href="/foo">` for internal routes — use `resolve('/foo')`.
- ❌ Don't `fetch` in `onMount` for data the page needs on first paint — use `load`.
- ❌ Don't disable lint rules without explaining why in a comment.
