---
title: "UI Components"
published: true
visible: true
taxonomy:
  audience: [developer]
  difficulty: [intermediate]
  topic: [developer, ui, components]
metadata:
  description: "Preact components for common public-site patterns — search, auth, forms, payments"
---

# UI Components (`@dune/core/ui`)

`@dune/core/ui` exports six Preact components covering the most common public-site patterns. All components emit semantic HTML with `dune-*` BEM class names so you can style them with your own CSS. No external style dependencies are included.

```ts
import {
  SearchBar,
  LoginForm,
  ProfileCard,
  CommentSection,
  SubscriptionForm,
  FormRenderer,
} from "@dune/core/ui";
```

Components that make API calls (SearchBar, CommentSection, SubscriptionForm, FormRenderer) are client-side islands — they require JavaScript and use `useState`/`useEffect` from Preact. LoginForm and ProfileCard are server-renderable with no client-side dependencies.

---

## SearchBar

An accessible combobox that calls `/api/search` as the user types. Keyboard-navigable (↑↓ to move through results, Enter to navigate, Esc to close). Debounces 200ms.

```tsx
import { SearchBar } from "@dune/core/ui";

<SearchBar placeholder="Search the site…" limit={8} />
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `placeholder` | `string` | `"Search..."` | Input placeholder text |
| `limit` | `number` | `5` | Max results to fetch |
| `className` | `string` | — | Extra class on the container |

Renders a `<div class="dune-searchbar">` with an `<input>`, a `<ul class="dune-searchbar__results">`, and individual `<li class="dune-searchbar__result">` items.

---

## LoginForm

Server-renderable form for public site login. Renders OAuth provider buttons and/or a magic-link email input depending on the `providers` prop.

```tsx
import { LoginForm } from "@dune/core/ui";

<LoginForm
  providers={["github", "google", "magic"]}
  redirectTo="/dashboard"
/>
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `providers` | `string[]` | `[]` | `"github"`, `"google"`, `"discord"`, `"magic"` |
| `redirectTo` | `string` | — | Redirect target after login (passed as `?next=`) |
| `className` | `string` | — | Extra class on the form |

OAuth buttons link to `/auth/{provider}?next={redirectTo}`. The magic-link input POSTs to `POST /auth/magic-link/send`.

---

## ProfileCard

Displays an authenticated `SiteUser`'s info: avatar (if present), display name, email, roles, and a logout button.

```tsx
import { ProfileCard } from "@dune/core/ui";
import type { ProfileCardUser } from "@dune/core/ui";

const user: ProfileCardUser = {
  name: "Alice",
  email: "alice@example.com",
  avatarUrl: "https://avatars.githubusercontent.com/u/1234",
  roles: ["member"],
};

<ProfileCard user={user} />
```

| Prop | Type | Description |
|------|------|-------------|
| `user` | `ProfileCardUser` | `{ name?, email, avatarUrl?, roles }` |
| `className` | `string` | Extra class on the card container |

The logout button POSTs to `/auth/logout`. In a TSX page handler, read the current user from `ctx.state.siteUser` (set by the auth middleware) to populate this component.

---

## CommentSection

Client-side island for threaded comments on a page. Reads comments from `GET /api/comments/{route}` and posts new ones to `POST /api/comments/{route}`. Requires the comments system to be enabled (see [Comments](../../12.comments)).

```tsx
import { CommentSection } from "@dune/core/ui";

<CommentSection
  pageRoute="/blog/my-post"
  currentUser={{ email: "alice@example.com", name: "Alice" }}
/>
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `pageRoute` | `string` | — | Page route whose comments to load. Required. |
| `currentUser` | `{ email, name? }` | — | Logged-in user. When absent, a name/email input is shown. |
| `className` | `string` | — | Extra class on the section container |

---

## SubscriptionForm

Client-side island that initiates a Stripe Checkout session. POSTs to `/payments/checkout/{productId}` and follows the server redirect to Stripe. Shows a loading state during the request. Requires the payments system to be configured (see [Payments](../../06.extending/05.payments)).

```tsx
import { SubscriptionForm } from "@dune/core/ui";

<SubscriptionForm productId="membership" label="Subscribe — $10/month" />
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `productId` | `string` | — | Must match a product `id` in `site.yaml`. Required. |
| `label` | `string` | `"Subscribe"` | Button label text |
| `className` | `string` | — | Extra class on the form |

---

## FormRenderer

Client-side island that dynamically fetches a form schema from `GET /api/forms/{formName}` and renders the fields. Submits via `POST /api/forms/{formName}`. Handles loading, field validation errors, and success states. Falls back gracefully to a "form not available" message on 404.

```tsx
import { FormRenderer } from "@dune/core/ui";

<FormRenderer
  formName="contact"
  successMessage="Thanks! We'll be in touch."
/>
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `formName` | `string` | — | Form name as defined in `schemas/`. Required. |
| `successMessage` | `string` | `"Submitted successfully."` | Message shown after successful submission |
| `className` | `string` | — | Extra class on the form container |

Supports all blueprint field types: `text`, `email`, `tel`, `textarea`, `number`, `select`, `checkbox`, `file`, `hidden`.

---

## Styling

All components use `dune-*` prefixed BEM class names. Target them in your theme CSS:

```css
.dune-searchbar { position: relative; }
.dune-searchbar__results { list-style: none; position: absolute; background: white; }
.dune-searchbar__result { padding: 0.5rem 1rem; cursor: pointer; }
.dune-searchbar__result:hover { background: #f0f0f0; }

.dune-profile-card { display: flex; align-items: center; gap: 1rem; }
.dune-profile-card__avatar { width: 40px; border-radius: 50%; }
```

No default styles are included — the components render unstyled HTML.
