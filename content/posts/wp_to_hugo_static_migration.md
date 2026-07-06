---
title: "Retiring WordPress: a byte-identical migration to Hugo and GitHub Pages"
date: 2026-07-06
description: "How we moved a WordPress site off AWS onto Hugo and GitHub Pages without visitors noticing — capture, verify byte-for-byte, cut over DNS, and stop paying for an ALB."
author: "Matt Popa"
tags:
  - WordPress
  - Hugo
  - GitHub Pages
  - AWS
  - Static Sites
---
![wordpress hello](/images/wp_hello.webp)

A while back we wrote about [building a WordPress-as-a-service platform on AWS](/posts/aws_hosting_wp/) —
ALB, EC2, ACM, Route 53, the works. It worked great. It also cost real money every month to serve a
portfolio site that changes a few times a year, and it kept a PHP stack, a database, and a pile of
plugins patched and reachable from the internet for no good reason.

So we retired it. This is the story of moving [stefaniapana.design](https://stefaniapana.design/) from
that setup to a static Hugo site on GitHub Pages, with one hard requirement: **visitors must not be able
to tell**. Same design, same URLs, same lazy loading, same everything. Here's how that went, including
the gotchas that bit us on the way.

---

### Why bother, and why Hugo

The monthly bill was the trigger: a load balancer, an instance, and storage all billing around the
clock to serve mostly-static HTML. The hidden costs were worse: WordPress core updates, plugin CVEs,
PHP versions, database backups — a maintenance treadmill for a site with no dynamic content worth
the name.

We already host this blog with Hugo on GitHub Pages, so the target was obvious. GitHub gives you one
*user* site per account plus **unlimited project sites**, and each project site can carry its own custom
domain with a free auto-renewing Let's Encrypt cert. Hosting cost: essentially zero — the only thing
left on the bill is the DNS hosted zone.

---

### The trick: don't rebuild the theme, capture it

The classic migration advice is to export your content and rebuild the design with a Hugo theme. For a
designer's portfolio, that's a trap — you'll spend weeks chasing pixel differences and still lose.

We went the other way: treat the **rendered HTML as the source of truth**. WordPress (Kadence theme +
Gutenberg blocks, no page builder soup, thankfully) already produces the final markup. So:

1. Mirror the rendered site: `wget --mirror --page-requisites https://example.com/`
2. Store each URL's full HTML as a Hugo content file, one per page.
3. Render it through a single passthrough layout.

The entire "theme" is one file, `layouts/all.html`:

```
{{- $base := site.BaseURL -}}
{{- .RawContent
      | replaceRE `https://stefaniapana\.design/` $base
      | safeHTML -}}
```

That `replaceRE` is the load-bearing part: content is stored with production-absolute URLs, and the
layout rewrites the domain to whatever `baseURL` you're building with. A localhost build is fully
self-contained for testing; a production build reproduces the original markup **byte-for-byte**.

Tell Hugo to get out of the way in `config.yml` — no taxonomies, no generated sitemap (we keep the
WordPress ones for URL parity), no generator meta:

```
disableKinds: [taxonomy, term, rss, sitemap, robotsTXT, section]
disableHugoGeneratorInject: true
security:
  allowContent: [text/html]
```

Assets (theme CSS/JS, fonts, uploads) go into `static/` unchanged. WordPress serves them with
`?ver=1.5.1` cache-busting suffixes — keep those in the HTML, store the files without the query string,
and every URL keeps working.

For the media originals we didn't scrape — we pulled `wp-content/uploads` straight off the EC2 box via
SSM port forwarding (no SSH keys needed, and the instance role needs zero extra permissions):

```
aws ssm start-session --target i-xxxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["8909"],"localPortNumber":["8909"]}'
```

with a throwaway `python3 -m http.server 8909 --bind 127.0.0.1` on the instance serving a tarball.

---

### Trust nothing, diff everything

The nice thing about the capture approach is that "did we break anything?" stops being a matter of
opinion. Two verification loops, both cheap:

**Byte diff.** Build with the production `baseURL` and `cmp` every generated page against the wget
mirror. We iterated until all 41 URLs — pages, posts, category and tag archives, the author page, the
404 — came back identical. The only intentional diff on the whole site: one swapped `<script>` tag for
the contact form (below).

**Pixel diff.** Byte-identical HTML can still render differently if an asset 404s, so we screenshotted
every page with headless Chrome against both the local build and production, and compared the PNGs.
Fun fact: most pages produced *literally byte-equal screenshots*. The few that didn't turned out to be
the live site disagreeing with itself between two consecutive screenshots — parallax and image decode
timing — which you can prove by diffing production against production.

One catch worth stealing: **crawl coverage isn't sitemap coverage**. Our projects listing page was in
the WordPress sitemap but not linked from the nav (it's behind a dropdown), so the mirror missed it.
Cross-check every sitemap URL against your build before you call it done.

---

### The two dynamic bits

**Forms.** The Kadence form block posts to `admin-ajax.php`, which no longer exists. We wrote a drop-in
replacement for its frontend script — same validation logic, same spinner, same success and error
messages pulled from the WordPress database — that posts JSON to [FormSubmit](https://formsubmit.co/)
instead. No account needed, one activation click per (email, domain) pair, and the endpoint address is
the site's public contact email, so nothing new is exposed. Visitors see the exact same behavior.

**Search.** The header search box does `GET /?s=term`, and GitHub Pages ignores query strings. For a
16-page portfolio we left the markup identical and accepted that search quietly lands on the homepage.
Client-side search (Fuse.js) is the upgrade path if it ever matters.

---

### Gotchas that actually hurt

**Your uploads folder is not just images.** Plugins cache things in `wp-content/uploads`, and some of
those cached JSON payloads contain **API keys**. GitHub's secret scanning flagged one within minutes of
our first push. Screen everything non-media out of the tarball *before* committing, because scrubbing
git history afterwards is a much worse afternoon — and remember the old commits stay fetchable on
GitHub's servers until support runs a gc.

**GitHub commits as you, with your real email.** On branch-based Pages deployments, changing the custom
domain makes GitHub write `Create CNAME` / `Delete CNAME` commits to your publishing branch — authored
with your account's primary email address. Enable *"Keep my email addresses private"* in GitHub settings
before you start, not after you notice.

**Certificates need DNS first, and sometimes a nudge.** GitHub can only request the Let's Encrypt cert
after your apex A records point at Pages. Check `dig CAA` on your domain first (a leftover CAA record
from ACM days can silently block issuance — ours was clean), and if provisioning doesn't start, remove
and re-add the custom domain via the API. Ours went from nothing to issued in seconds after that nudge.

---

### The cutover

With everything verified, the actual switch was one Route 53 change batch: apex `A` records to GitHub's
four IPs (`185.199.108-111.153`), matching `AAAA` records, and `www` CNAME'd to `mattpopa.github.io`.
With a 60-second TTL the flip propagated almost immediately, we watched the cert issue, ticked
*Enforce HTTPS*, and re-ran the byte diff against the live domain. Identical. WordPress kept running
untouched for a few days as a rollback plan nobody needed, and then the EC2s and the ALB went to the
big cloud in the sky.

---

### Wrapping up

Static hosting for practically nothing, no more PHP to patch, no database to back up, and a git repo
where the whole site — content, history, and deploys — lives next to a README that tells you how to
add a post. The
capture-and-verify approach turned what's usually a redesign project into a weekend migration with a
provable result: not one visitor noticed, which was the whole point.
