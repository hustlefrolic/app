# Build Brief: hustlefrolic.com Rebuild (WordPress -> Laravel)

Paste this as the first message in Claude Code. Build **one phase at a time**. Finish a phase, show me how to test it, and wait for my go-ahead before starting the next.

---

## 0. Ground rules

1. **Content stays exactly as it is.** Every page and blog post keeps its current wording, headings, and meaning. Only the design and the plumbing change. Never rewrite, summarize, "improve", or regenerate copy. If text looks wrong, flag it to me instead of fixing it.
2. **URLs stay exactly as they are.** The current site uses flat WordPress permalinks (`/about/`, `/tdee-calculator/`, `/how-to-lose-belly-fat-naturally-no-crash-dieting/`). The new site must serve the same URLs with a trailing slash so Google rankings are preserved.
3. **Design is new and modern.** Clean, fast, mobile-first. Details in section 7.
4. **Scope is a lead-generation site.** Visitors get lightweight lead records tied to calculator use. **Out of scope:** payments, subscriptions, workout programs, client dashboards, coaching features.
5. **Hosting is Hostinger Business shared hosting.** PHP + MySQL, SSH, Composer, cron. There is no persistent Node process, no Redis, and no Docker. Design everything to run on that.
6. **The source of truth for content** is `WordPress_2026-09-28.xml` (WordPress export). I will place it in `/source/`. I will also provide the `wp-content/uploads` folder as `/source/uploads/` when we reach the media step.

---

## 1. Tech stack

- Laravel (latest stable) on PHP 8.2+
- Livewire (latest stable) for the calculator gate and interactive forms
- Tailwind CSS + Vite (build assets in CI or locally, not on the server)
- MySQL (Hostinger)
- Filament (latest stable) for the admin panel and blog CMS, on a separate `admin` auth guard
- Laravel Socialite (Google sign-in), passwordless magic-link login for leads
- Cloudflare Turnstile for spam protection
- Queue driver: `database`. Cache and session: `file` or `database`. Scheduler via a cron entry running `php artisan schedule:run` every minute.
- Mail: SMTP (Hostinger mailbox to start, keep it swappable to Brevo or Resend through `.env`)

---

## 2. Content inventory (from the export)

- **25 blog posts**, all published, Gutenberg block format. Categories: Fat Loss, Nutrition, Mindset, Recovery, Workouts, Featured, Uncategorized. No tags.
- **26 pages**, mostly built in Elementor.
- **157 media attachments**
- **SEO data** stored by All in One SEO (AIOSEO) in these post meta keys: `_aioseo_title`, `_aioseo_description`, `_aioseo_og_title`, `_aioseo_og_description`, `_aioseo_twitter_title`, `_aioseo_twitter_description`, `_aioseo_keywords`, `_aioseo_og_article_section`, `_aioseo_og_article_tags`
- **3 Contact Form 7 forms:** Contact form 1, Contact Us, Subscription
- **Menu:** Fitness Tools, Services (header). Footer links to be confirmed with me.

### Pages and routes (all served at `/{slug}/`)

| Slug | Type | Notes |
|---|---|---|
| `/` | Home | The page currently titled "Online Personal Training - Fitness Coach" is the front page |
| `/services/` | Static | |
| `/about/` | Static | Short page |
| `/free-consultation/` | Static + form | Lead form saved to the admin panel |
| `/contact-us/` | Static + form | |
| `/resources/` | Static | |
| `/success-stories/` | Static | **Currently empty. Ask me before building.** |
| `/blog/` | Blog index | Paginated, filter by category |
| `/fitness-calculators/` | Calculator hub | Very large page (about 140 KB of markup). Rebuild as a clean grid of the 12 tools. |
| `/tdee-calculator/` | Calculator | |
| `/bmr-calculator/` | Calculator | |
| `/bmi-calculator/` | Calculator | |
| `/calorie-calculator/` | Calculator | |
| `/protein-calculator/` | Calculator | |
| `/body-fat-calculator/` | Calculator | |
| `/macro-calculator/` | Calculator | |
| `/fat-intake-calculator/` | Calculator | |
| `/lean-body-mass-calculator/` | Calculator | |
| `/one-rep-max-calculator/` | Calculator | |
| `/target-heart-rate-calculator/` | Calculator | |
| `/daily-water-intake-calculator/` | Calculator | |
| `/2-minute-breathing-exercises-to-reset-your-nervous-system/` | Static + widget | Contains a breathing exercise widget |
| `/privacy-policy/`, `/terms-and-conditions/`, `/disclaimer/`, `/refund-policy/` | Legal | Import as-is into a clean prose layout |
| `/{post-slug}/` | Blog post | Same flat structure as today |
| `/category/{slug}/` | Category archive | Keep the WordPress path |

**Routing rule:** posts and pages share one flat namespace. Add a catch-all route `/{slug}/` that resolves a page first, then a post, else 404. Enforce **globally unique slugs across pages and posts** and reserve system paths (`admin`, `blog`, `category`, `login`, `auth`, `api`, `sitemap.xml`, `robots.txt`, etc.).

**Trailing slash policy:** canonical URLs end with `/`. Add middleware that 301-redirects the non-slash version to the slash version.

---

## 3. Database schema

**admins**: id, name, email (unique), password, timestamps. Used only by the `admin` guard. Enable 2FA on the admin panel.

**leads**
- id, name (nullable), email (nullable, unique), phone (nullable), google_id (nullable)
- email_verified_at (nullable)
- consent_at, consent_version (which consent text they accepted), marketing_opt_in (bool)
- source (`gate`, `contact`, `consultation`, `newsletter`, `google`), first_utm_source / medium / campaign / term / content, referrer, landing_page
- first_seen_at, last_seen_at, ip_hash, user_agent
- status (`new`, `contacted`, `client`, `lost`), admin_notes
- Rule: at least one of email or phone is required.

**calculator_runs**
- id, lead_id (FK), calculator_slug, inputs (JSON), results (JSON), session_id, ip_hash, page_url, created_at

**events** (funnel tracking)
- id, lead_id (nullable), session_id, type (`calculator_viewed`, `gate_shown`, `gate_submitted`, `gate_abandoned`, `run_completed`, `magic_link_sent`, `form_submitted`), calculator_slug (nullable), meta (JSON), created_at

**contact_messages**: id, lead_id (nullable), name, email, phone, subject, message, form (`contact`, `consultation`, `newsletter`), status (`new`, `read`, `replied`), created_at

**posts**
- id, wp_id, title, slug (unique), excerpt, content (longText, sanitized HTML), featured_image (nullable), category_id, status (`draft`, `scheduled`, `published`), published_at, reading_time (computed on save)
- seo_title, seo_description, seo_keywords, og_title, og_description, twitter_title, twitter_description
- created_at, updated_at, updated_at index for sitemap `lastmod`

**categories**: id, name, slug (unique), description. Seed: Fat Loss, Nutrition, Mindset, Recovery, Workouts, Featured, Uncategorized.

**pages**: id, wp_id, title, slug (unique), template (`home`, `default`, `legal`, `calculator`, `calculator-hub`, `blog`), content (longText, nullable), sections (JSON, nullable), seo_* (same as posts), status

**media**: keep it simple. Files live in `storage/app/public/uploads/` mirroring the WordPress `YYYY/MM/filename` structure so old image URLs can be mapped 1:1.

**redirects**: id, from_path (unique), to_path, status_code (default 301), hits. Managed from the admin panel.

---

## 4. WordPress import (artisan commands)

Write these as re-runnable, idempotent commands keyed on `wp_id`.

1. `php artisan hf:import-inspect`: parse the XML and print a table of everything found (pages, posts, categories, meta keys). No writes. Use this first to confirm the counts match section 2.
2. `php artisan hf:import-posts`
   - Read each `post` item with status `publish`.
   - Convert Gutenberg output to clean HTML: strip `<!-- wp:... -->` comments, keep the actual markup, headings, lists, tables, links, and images.
   - Map the category, slug, publish date, and the AIOSEO meta fields to the `seo_*` columns.
   - Rewrite image URLs from `https://hustlefrolic.com/wp-content/uploads/...` to the local storage path.
   - Set `featured_image` from `_thumbnail_id` if present.
   - Never modify wording. Report any post whose imported text length is under 1,000 characters.
3. `php artisan hf:import-pages`
   - Import legal pages (privacy, terms, disclaimer, refund) as HTML content, stripping Elementor and theme wrappers (breadcrumb SVGs, inline theme classes) but keeping every word.
   - For Elementor-built pages (home, services, about, free-consultation, resources, contact-us, breathing page), **do not copy the Elementor markup**. Instead run step 4.
4. `php artisan hf:extract-page-copy {slug}`: read `_elementor_data` (JSON) and the rendered content for the page, and output a plain Markdown file per page in `/storage/app/page-copy/{slug}.md` containing every heading, paragraph, list, button label and link, in reading order. **I will review these files before you build templates**, and the Blade templates must use exactly this text.
5. `php artisan hf:extract-calculators`: for each of the 12 calculator pages, extract the existing custom HTML/CSS/JS block into `/storage/app/calculators/{slug}/` (markup, script, and any explanatory text below the tool). These are already custom-built with the navy/orange design and are the source for the new calculator pages.
6. `php artisan hf:import-media`: copy `/source/uploads/` into `storage/app/public/uploads/`, keeping paths.

**Stub posts:** these 7 posts have only a few hundred characters in the export: Best Gym Workout Plan for Beginners, Push Pull Legs vs Bro Split, Home Workout Plan Without Equipment, How to Progressive Overload, Metabolism Explained, Morning vs Evening Cardio, and Why Your Chest Isn't Growing (this last one is longer, about 24K chars, so check it separately). Import the 6 short ones as **draft**, not published, and list them for me. I will decide what to do with them.

---

## 5. Calculator gate and lead capture

### User flow

1. Visitor opens a calculator page and fills in the form. A `calculator_viewed` event is logged.
2. They click **Calculate Now**.
3. **Already identified** (valid lead session): compute and show the result immediately.
4. **Not identified:** open a modal (Livewire) with:
   - Name
   - Email **or** phone number (at least one required; email is the primary field)
   - "Continue with Google" button
   - Consent checkbox (required, unchecked by default) linking to the Privacy Policy
   - Optional checkbox for fitness tips by email
   - Turnstile widget
5. On submit: create or find the lead, start a persistent session, run the calculation, show the result, save a `calculator_run`. The calculator inputs the user typed must be preserved and not reset.
6. Returning visitors: a "Already signed up? Get a sign-in link" option sends a magic link to their email (signed URL, 15 minutes, single use).

### Identification and security

- Identification cookie: `hf_lead` (signed, encrypted, HttpOnly, SameSite=Lax, 1 year) plus the Laravel session.
- **Calculations must run server-side.** Port each calculator's JavaScript formulas to PHP classes in `app/Calculators/` (one class per calculator, same interface, e.g. `calculate(array $inputs): array`). The result is only returned once a lead is identified. This stops people bypassing the gate by reading the JS.
- **Parity tests:** for each calculator, write PHPUnit tests using fixed inputs, with expected outputs taken from running the existing JS. Results must match to the same rounding.
- Server-side validation on all inputs, with sensible min/max ranges.
- Rate limiting: gate submission (e.g. 10/hour per IP), magic link requests (e.g. 5/hour per email and per IP), contact forms.
- Turnstile on the gate, contact, consultation, and newsletter forms. Also add a honeypot field.
- Store **hashed** IPs (SHA-256 + app-key salt), never raw IPs.
- Phone numbers are collected but **not verified** in this version. Store them normalized to E.164 (default country IN). Design the code so OTP verification can be added later behind an interface.

### Compliance (India DPDP Act)

- Consent checkbox is mandatory, never pre-ticked, and unbundled from marketing consent.
- Store `consent_at` and `consent_version`. Keep the consent text versioned in config.
- Admin can export a lead's data and delete a lead (deletes runs and events too).
- Every marketing email includes an unsubscribe link.

---

## 6. Admin panel (Filament, at `/admin`)

Separate `admin` guard. Only the seeded admin can log in. No public registration.

**Dashboard widgets:** new leads (today, 7 days, 30 days), leads over time (line chart), calculator runs per calculator (bar chart), top sources/UTM, conversion funnel (viewed -> gate shown -> submitted -> result), latest leads table, unread messages count.

**Resources:**
- **Leads:** searchable and filterable by date, source, status, calculator used, has-phone, has-email. Columns: name, email, phone, source, runs count, last seen, status. Edit status and notes. **Detail view shows a timeline of every calculator run with inputs and results.** Bulk CSV export. Delete/export personal data action.
- **Calculator runs:** full log with filters; export CSV.
- **Contact messages:** inbox with status; reply link.
- **Posts (blog CMS):** rich text editor (with image upload into `uploads/`), category, featured image, excerpt, slug (auto from title, editable, uniqueness validation across pages and posts), SEO fields (title, description, keywords, OG/Twitter), status with scheduled publishing (scheduler publishes at `published_at`), live reading-time, preview link for drafts.
- **Categories**
- **Pages:** edit text of the static and legal pages, plus SEO fields.
- **Redirects:** manage 301 redirects and see hit counts.
- **Settings:** site name, contact email, WhatsApp number, social links, consent text, notification email.

**Notifications:** email me on each new lead and each new contact/consultation message (queued).

---

## 7. Design system

Keep the existing brand palette and type from the calculator suite, applied across the whole site:

- Navy dark `#1f242e`, navy mid `#2c3340`, orange `#ff5e2e`, orange deep `#e04a1e`, orange pale `#ffe6dc`, plus an off-white background and a light navy tint.
- Font: **Inter Tight** (300-700). Load from Google Fonts with `font-display: swap`, or self-host.
- Define these as Tailwind theme tokens and CSS variables.

**Look and feel:** modern, confident, uncluttered.
- Sticky header with logo, main nav (Fitness Tools dropdown, Services, Blog, About, Contact) and a primary "Free Consultation" button. Mobile menu is a full-screen drawer.
- Hero sections with one clear headline and one primary CTA. Generous whitespace, 8px spacing scale.
- Rounded cards (12-16px), soft shadows, subtle hover states. Orange is used for CTAs and key accents only.
- Calculators: app-like form, large inputs, unit toggles, and a clear result card with the key number highlighted. The gate modal should feel light, not like a wall.
- Blog: clean index grid with category filter, reading-optimized article layout (about 70ch measure), table of contents for long posts, reading time, related posts, author box, and a CTA to a relevant calculator.
- Footer: nav, legal links, socials, newsletter form.
- Motion: subtle fade/slide on scroll only, respecting `prefers-reduced-motion`.
- Accessibility: WCAG AA contrast (check orange on navy and on white), visible focus states, semantic HTML, alt text.
- Performance: WebP images, `loading="lazy"`, responsive `srcset`, no unused JS. Target Lighthouse 90+ on mobile.

Do not add stock imagery or new copy. Where a section needs an image I don't have, use a neutral placeholder and list it for me.

---

## 8. SEO (must not regress)

- Per-page `<title>`, meta description, canonical (with trailing slash), Open Graph and Twitter tags from the imported AIOSEO values. Fall back to the page or post title if missing.
- `/sitemap.xml` (pages, posts, categories, with `lastmod`) and `/robots.txt` referencing it.
- JSON-LD: `Organization`, `WebSite`, `Article` on posts, `BreadcrumbList`, and `FAQPage` only where the content has an actual FAQ.
- Redirects to add at launch: `/?p={id}` -> new URL (use `wp_id`), `/wp-sitemap.xml` and `/sitemap_index.xml` -> `/sitemap.xml`, `/feed/` -> `/blog/`, `/author/*` -> `/about/`, `/wp-login.php` and `/wp-admin` -> `/`.
- Provide an RSS feed at `/feed.xml`.
- Generate a **pre-launch URL check**: an artisan command that reads every URL from the export and confirms it returns 200 (or the intended 301) on the new site.

---

## 9. Hostinger deployment notes

- PHP 8.2 or newer, with required extensions enabled in hPanel (`intl`, `mbstring`, `gd` or `imagick`, `zip`, `fileinfo`).
- The web root must serve Laravel's `public/` folder. Check whether hPanel lets me change the document root; if not, keep the app outside `public_html` and point `public_html` to `app/public` (symlink), and tell me the exact steps.
- Deploy through Git (hPanel Git or SSH `git pull`) or GitHub Actions over SSH. Build Vite assets in CI or locally and deploy the built `public/build`.
- `.env` lives only on the server. `APP_DEBUG=false`, HTTPS forced.
- Run `php artisan migrate --force`, `config:cache`, `route:cache`, `view:cache` on deploy. Run `php artisan storage:link` (check that symlinks work on the plan; otherwise write directly under `public/uploads`).
- Cron: `* * * * * php /path/to/artisan schedule:run` in hPanel. Queue worker: run `queue:work --stop-when-empty` from the scheduler every minute.
- Enable daily backups (hPanel) and a database dump job before each deploy.
- Build and test on a **staging subdomain** (e.g. `staging.hustlefrolic.com`) with `noindex`, and only switch the live domain after sign-off.

---

## 10. Build phases

**Phase 1: Foundation.** Laravel project, Tailwind theme tokens, base layout (header, footer, mobile menu), all migrations, Filament installed with the `admin` guard and seeded admin, staging deploy pipeline working on Hostinger.
*Done when:* staging shows the empty layout, `/admin` login works with 2FA, deployment steps documented.

**Phase 2: Import and blog.** Run `hf:import-inspect`, then import categories, posts, and media. Build blog index, category pages, single post template, and the Filament Posts/Categories resources. Routing catch-all and trailing-slash middleware.
*Done when:* all 25 posts (stubs as drafts) render at their original URLs with correct SEO tags and images, and I can create and schedule a post from the admin panel.

**Phase 3: Static pages.** Run `hf:extract-page-copy`, wait for my review, then build Home, Services, About, Resources, Free Consultation, Contact Us, the breathing exercise page, the legal pages, and (once I decide) Success Stories. Contact and consultation forms save to `contact_messages`, send email notifications, and are protected by Turnstile.
*Done when:* all page text matches the approved copy exactly and forms work end to end.

**Phase 4: Calculators and lead gate.** Extract calculators, build the hub and 12 calculator pages, PHP calculator classes with parity tests, the gate modal, Google sign-in, magic links, event logging, and run storage.
*Done when:* every calculator matches the old JS results, the gate works for first-time and returning visitors, and no result can be obtained without an identified lead.

**Phase 5: Admin analytics.** Dashboard widgets, Leads, Calculator runs, Messages, Redirects and Settings resources, CSV exports, and lead notifications.
*Done when:* I can see a complete journey for a test lead (views, gate, run, inputs and results) inside the admin panel.

**Phase 6: SEO, QA, launch.** Sitemap, robots, JSON-LD, redirects, the URL check command, Lighthouse and accessibility pass, cross-browser and mobile testing, performance tuning, and cutover plan (DNS is unchanged; switch the document root or deployment on the live domain, keep the WordPress files and a database backup for 30 days).
*Done when:* the URL check passes for 100% of legacy URLs and Search Console can accept the new sitemap.

---

## 11. Open questions (ask me before the relevant phase)

1. What should happen with the 6 stub posts (short content), and is "Why Your Chest Isn't Growing" complete?
2. Success Stories page: include it? Do I have testimonials to supply?
3. Header and footer link structure beyond Fitness Tools and Services.
4. Which email address receives lead and contact notifications, and which sender address should outgoing mail use?
5. Do I want the Google Analytics / Search Console / any Meta Pixel scripts carried over (they are not in the export)?
6. Logo and favicon files.

Start with **Phase 1** and tell me what you need from me.
