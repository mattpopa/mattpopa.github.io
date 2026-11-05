---
title: "Rebuilding this site with Claude Code, still free on GitHub Pages"
date: 2026-11-05
description: "How a stale Hugo blog became a firm site in one evening with Claude Code: what the agent did, the pain points it found and fixed, the GitHub Pages deploy that refused to deploy, and what still needed a human."
author: "Matei Popa"
tags:
  - Claude Code
  - Hugo
  - GitHub Pages
  - Static Sites
  - AI
  - DevOps
---
![Rebuilding this site with Claude Code](/images/claude_code_site_redesign.webp)

This site has run on Hugo with the PaperMod theme for a couple of years, free on GitHub Pages. It
worked, and it looked like what it was: a personal blog with a logo in a circle and "All Things DevOps"
under it. We wanted it to read like a firm, keep the zero hosting bill, keep the contact form and keep
the blog. Instead of a weekend of CSS, we handed the repo and the live site to Claude Code and worked
through it in one evening.

This is what happened, including the parts where the agent was wrong, the parts where GitHub was
broken, and the parts where only a phone screenshot told the truth.

---

### Start with a review, not a rewrite

The first prompt was not "redesign my site". It was "is this okay, and what do you recommend?" with the
repo and the URL. The agent cloned the repo, read the Hugo config, the theme pin and the deploy
workflow, then opened the live site in the browser and took screenshots at desktop and phone width.

The diagnosis was more useful than a redesign would have been: the generator was not the dated part,
the way the template was used was. A profile-mode homepage that said nothing about what we sell, a
contact page with a reCAPTCHA checkbox from 2014 and a redirect to a third-party thank-you page, and a
cookie banner for an analytics script that loaded regardless of what you clicked. Its recommendation:
stay on Hugo, write a real homepage, fix the contact page, drop the analytics and the banner, bump the
theme and Hugo. We agreed, and asked for a branch we could look at locally before anything went live.

---

### What the agent built

In about an hour of wall-clock time, on a branch, with a local `hugo server` running and screenshots at
every step:

- A homepage template with a hero, a stack strip, a row of facts, six service cards, three engagement
  models, the latest three posts and a closing call to action. All copy lives in `data/home.yaml`, so
  changing a sentence never means touching a template.
- About and contact pages as proper templates, with the headshot and logo converted to webp. The brand
  colours came from the logo itself: with no image library installed, the agent converted the PNG to
  BMP with `sips` and counted pixels in a few lines of Python.
- A stylesheet layered on top of PaperMod's variables, so light and dark mode both work and the theme
  stays upgradeable.
- Google Analytics, the consent banner and their leftover files removed.
- PaperMod bumped from a May 2025 pin to upstream, Hugo from 0.148 to 0.169, and the workflow trimmed:
  the Dart Sass install step was there for a theme that does not use Sass.
- The build verified under the exact Hugo version the workflow pins, in the official container, because
  the local Hugo was a few versions behind.

What we did in that hour was decide what the words say. The agent drafted copy from our CV; we cut it
down, used a real name instead of a nickname, removed client names and the email address, said
"Vanta or Drata" instead of one product, and dropped a headcount flex that invited the wrong question.
That split held for the whole evening: the agent does the mechanical work and the verification, the
human decides what is public.

---

### Pain points it found, and a few it caused

**A theme bump is an API change.** The new PaperMod flags dark mode with a `data-theme` attribute instead
of a `.dark` class. Every dark-mode override the agent had just written was silently ignored, and the
stack pills rendered light-on-light. Nothing failed, no warning, the build was green. A screenshot caught
it. The same bump moved markdown spacing to a new `md-content` class, so the custom about page lost its
paragraph gaps until the templates carried that class too.

**Hugo changed template lookup.** `layouts/page/contact.html` had resolved for years. On 0.164 it quietly
stopped, and the contact page rendered as a plain post. Moving the templates to `layouts/_default/`
fixed it. Again: green build, wrong page.

**The clever contact form was broken.** The agent replaced the captcha widget with a JavaScript submit
and a honeypot, which looks modern and shows an inline thank-you. Formspree answered every one of those
requests with `400 Please complete the reCAPTCHA`, because the captcha was still enabled on their side,
and we wanted it enabled. The fix was less clever: put the widget back, render it explicitly so it
follows the theme toggle without a page refresh, keep the inline submit with the captcha token, and fall
back to a normal form post if anything is refused. We tested it with a real message and checked the
inbox. A green pipeline proves nothing about a form.

**Desktop previews lie about phones.** The contact page looked right in a 375px preview in Chrome and
wrong on an iPhone: the input fields were wider than the card. iOS Safari sizes a text field inside a
nested grid cell by the field's own default width, not by the cell. Plain block layout with an explicit
width fixed it. The phone screenshot was the test that mattered.

**Deprecation warnings are cheap to clear.** The run logs warned that four actions still targeted Node
20. The agent checked each action's `runs.using`, bumped to the Node 24 majors, and found that
`upload-pages-artifact` v4 still pinned a Node 20 `upload-artifact` underneath, so it went to v5. It
also found that `deploy-pages` caps its timeout at 10 minutes no matter what you set, which mattered
for the next part.

---

### The deploy that would not deploy

We merged the pull request at 20:25. The Hugo build was green in 10 seconds. The deploy job then sat
polling GitHub's Pages backend, which reported `updating_pages` for 10 minutes until the action gave up.
Re-running the failed job: same. Dispatching a fresh run: same.

The agent did what you would want a colleague to do. It confirmed the artifact was 2.86 MB and clean,
the same size as the last good deploy. It checked the Pages settings, the deployment records on the
`github-pages` environment, GitHub's status page, and the community threads, where other people were
reporting the identical hang during a run of Actions and Pages incidents those two days. Then it pushed
a one-line commit raising the deploy timeout.

That commit deployed in 38 seconds. The timeout had nothing to do with it; the action caps it at 10
minutes anyway. What changed was the commit. All three stuck attempts were deployments of the same
merge commit, and the first one, cancelled mid-flight, had left GitHub's side in a state the retries
could not get past. Every push since has deployed in under a minute.

Two lessons. When a Pages deploy hangs, push a fresh commit instead of re-running. And the reason this
was never an emergency: GitHub Pages only replaces the site when a deploy succeeds. The old site stayed
up for the whole hour.

---

### What stayed human

- Every decision about words: the positioning sentence, what is a client name and what is "an AI
  product", which certifications to list, whether "15+ years" or "since 2022" is the honest number.
- Taste. Light theme by default? We tried it and went back to following the visitor's system setting.
  The hero logo on phones? Gone, the header already carries the mark.
- Ticking the captcha and checking the inbox. An agent cannot and should not do the first part.
- The phone. The final test for every layout change was a screenshot from a real device, pasted into
  the chat.

---

### What it costs

Hosting: nothing. GitHub Pages serves the site with a free certificate on a custom domain, Formspree's
free tier handles the form, Hugo builds in 10 seconds on a GitHub-hosted runner. The only subscription
is the one that runs the agent. The whole thing was one evening, about fifteen commits, and a pull
request we could read.

---

### Wrapping up

The site is not a product and this was not a rewrite. It was an evening of a capable agent doing the
work we would have procrastinated on for months, with us deciding what the site says and what stays
private. The parts worth copying: give the agent the live site and the repo, not a description of them;
ask for a review before asking for changes; make it show you screenshots, in both themes, at phone width;
verify with the exact versions your CI uses; and test the one thing that talks to the outside world with
a real message.

And when GitHub Pages hangs: new commit, not re-run.
