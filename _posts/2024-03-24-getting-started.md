---
title: Where Should I Start?
author: uzrg
date: 2024-03-24 20:55:00 +0800
categories: [Blogging, Tutorial]
tags: [getting started, PowerShell, Git, GitHub, Jekyll, Markdown]
pin: true
img_path: "/posts/20240324"
---

Welcome. This site hosts tutorials and reference material for IT systems
administration work, built with Jekyll and hosted on GitHub Pages. This
first post walks through how the site itself was built, so it can serve
as a reference the next time I stand up a similar site — and hopefully
save you some time if you're doing the same.

## What is Jekyll?

<a href="https://jekyllrb.com/" target="_blank">Jekyll</a> is a static site generator written in <a href="https://www.ruby-lang.org/en/" target="_blank">Ruby</a> that turns <a href="https://www.markdownguide.org/getting-started/" target="_blank">Markdown</a> into a static site you can host on <a href="https://pages.github.com/" target="_blank">GitHub Pages</a> for free. Its biggest advantage is the theme ecosystem: someone else has already solved layout, navigation, and styling, so you're free to focus on content. This site runs on the <a href="https://chirpy.cotes.page/" target="_blank">Chirpy</a> theme. You don't need to know Ruby to use it, but it's worth learning <a href="https://www.markdownguide.org/basic-syntax/" target="_blank">Markdown syntax</a> — you'll be writing in it for every post.
One caveat: even though you won't be writing Ruby, you do need Ruby and Bundler installed locally to build and preview the site before pushing changes.

## Implementation Steps

1. Set up the local dev environment
   • Visual Studio Code
   • Git
   • Ruby
2. Clone the Chirpy theme into a local Git repository
3. Customize the site
   • Favicons
   • Avatar
   • URLs
   • Contact, authors, sharing options
   • Posts
4. Build and test locally
5. Deploy to GitHub Pages with an automated build/deploy workflow

Steps 1 and 2 are mechanical enough that I wrote a PowerShell script to
handle both at once — worth using if you'd rather not click through
installers by hand.

### Environment Setup Script

The script lives at <a href="https://github.com/uzrg/psprofile" target="_blank">github.com/uzrg/psprofile</a>. It checks for existing installs of Git, VS Code, and Ruby across the usual install paths before offering to set up what's missing, then clones the theme repo. I load it as a PowerShell profile (`CurrentUserAllHosts`, at `~\Documents\WindowsPowerShell\Profile.ps1`) so it's available every time I open a console — worth doing the same if you expect to reuse it.

Clone it with whichever method you're comfortable with:

**HTTPS:**

```bash
git clone https://github.com/uzrg/psprofile.git
```

**SSH** (requires a key already set up):

```bash
git clone git@github.com:uzrg/psprofile.git
```

**GitHub CLI:**

```bash
gh repo clone uzrg/psprofile
```

Or just open the repo page and use Code → Download ZIP.

Built for Windows; provided as-is.

### Customizing the Site Locally

If the PowerShell profile above is loaded, open the cloned repo in VS Code with:

```
code $devOpsPath\<repo directory>
```

Most of what you'll customize lives in `_config.yml`.

#### Favicon

Follow Chirpy's own <a href="https://chirpy.cotes.page/posts/customize-the-favicon/" target="_blank">favicon guide</a> to swap in your own icon set.

#### Avatar

Drop a 300×300 image (I named mine `avatar.jpg`) under `assets/img`, then point `_config.yml` at it:

```yaml
avatar: "/<your github username>/assets/img/avatar.jpg"
baseurl: "/<your github username>"
```

Why the username shows up in `baseurl` too — see the URL section below.

> My avatar and favicon come from a photo I took with my phone at sunrise, July 11, 2021. Still my favorite sunrise.
> {: .prompt-tip }

#### URL

```yaml
url: "https://<your github username>.github.io"
```

> **Note:** GitHub Pages serves your site at `{{ site.url }}/{{ site.baseurl }}`. With username `uzrg`, that's [https://uzrg.github.io/uzrg](https://uzrg.github.io/uzrg/) — and it's the same path Jekyll uses to resolve everything under `assets/img`, so getting `baseurl` right matters beyond just the homepage.
> {: .prompt-tip }
>
> `_config.yml` also holds your social links — X, GitHub, LinkedIn, and the rest.

#### Authors, Contact, and Share Settings

> These live under `_data/origin`:
>
> - `authors.yml`
> - `contact.yml`
> - `share.yml`
>   {: .prompt-tip }
>
> They're self-explanatory once you open them — adjust to taste.

#### Writing Posts

Chirpy's <a href="https://chirpy.cotes.page/posts/write-a-new-post/" target="_blank">Writing a New Post</a> guide covers the front-matter conventions and file naming you'll need.

### Building and Testing Locally

Once you've customized the config and written a post or two, preview it locally before pushing. If the setup script ran cleanly earlier, Ruby, Bundler, and their paths are already in place.

Open a terminal — VS Code's integrated one works fine — in `$devOpsPath\<your github username>`, then run:

```bash
bundle install
```

followed by:

```bash
bundle exec jekyll serve
```

That builds the site and serves it locally on port 4000. Open `http://127.0.0.1:4000/<baseurl>/` in your browser — for me, with `baseurl: "/uzrg"`, that's `http://127.0.0.1:4000/uzrg/`. If your `baseurl` is empty, drop it from the URL entirely.

One easy mistake: `baseurl` must start with a leading slash (`/`), or the site will build but every internal link will break.

### Deploying to GitHub Pages

Once the local preview looks right, push it:

1. **Point the repo at GitHub:**

   ```bash
   git remote add origin https://github.com/<your github username>/<your github project>.git
   ```

2. **Check what's changed:**

   ```bash
   git status
   ```

3. **Stage it:**

   ```bash
   git add *
   ```

4. **Commit:**

   ```bash
   git commit -m "Initial site setup and customization"
   ```

5. **Add Linux as a supported platform** — GitHub Actions runners are Linux, and your `Gemfile.lock` (if committed) needs to know that:

   ```bash
   bundle lock --add-platform x86_64-linux
   ```

6. **Commit that too:**

   ```bash
   git commit -m "Add Linux platform support for GitHub Actions"
   ```

7. **Push:**
   ```bash
   git push -u origin main
   ```

If any of this git workflow is unfamiliar, the <a href="https://git-scm.com/book/en/v2" target="_blank">Pro Git Book</a> is the best free reference I know of.

#### Enabling Pages

Last step: in the repo's Settings → Pages, point GitHub at your deployment branch and build workflow.

## Post-Launch Housekeeping

Getting the site live is only the first milestone. A handful of things
are easy to miss on initial setup and quietly cost you traffic,
clarity, or wasted CI minutes down the line. Worth running through
these once the site is up.

### Run one Pages deployment workflow, not two

If you clicked through GitHub's own quick-start prompt for enabling
Pages at any point, it likely added a generic
`Deploy Jekyll with GitHub Pages dependencies preinstalled` workflow to
`.github/workflows/`. That workflow builds through the `github-pages`
gem, which only supports a fixed allowlist of themes — Chirpy, and any
theme installed from a git source rather than that allowlist, isn't on
it. The workflow fails on every push, every time.

If you're also running a custom workflow that does
`bundle install && bundle exec jekyll build` directly — the approach
used above — you likely have both triggering on every push to your
default branch, racing each other under the same Pages deployment
lock. One always fails, one always succeeds, and the failing one is
pure noise. Keep the custom `bundle exec jekyll build` workflow and
delete the generic one.

### Keep `categories` to two; put everything else in `tags`

Chirpy's documentation is explicit: `categories` is designed to hold
**up to two** elements — a parent and a child — which drive the
theme's nested category-browsing page. `tags` can hold as many as you
like. It's an easy rule to lose track of once you're copy-pasting
front matter from post to post; a post with five or six categories
still builds fine, but only the first two participate in that nested
hierarchy, and the theme quietly generates a separate, thin archive
page for every category past that. Worth a periodic skim of your
posts' front matter — the site builds cleanly either way, so nothing
will flag it for you.

### Replace the placeholder site description

`_config.yml`'s `description:` field feeds both the page's meta
description and the `og:description` used in social link previews —
Slack, X, LinkedIn unfurls, all of it. The Chirpy starter ships this
with generic text describing the theme, not your site, and it's easy
to leave as-is since nothing breaks if you do. Write one or two
sentences that actually describe your site — keep it under roughly 155
characters so search engines don't truncate it.

### Verify with Google Search Console and Bing Webmaster Tools

`_config.yml` includes a `webmaster_verifications` block with empty
fields for Google, Bing, and a few others. Most people never fill
these in, which means the site has never actually been verified with
either search engine. It's a five-minute task, and it unlocks sitemap
submission, indexing status, and search query data:

```yaml
webmaster_verifications:
  google: your-google-verification-code
  bing: your-bing-verification-code
```

1. In [Google Search Console](https://search.google.com/search-console),
   add a property using **URL prefix**. For a GitHub Pages project
   site, that must be the full path including your repo name — e.g.
   `https://<username>.github.io/<repo>/` — not the bare
   `<username>.github.io` domain. The bare domain 404s for a project
   site, and verification fails with "Could not find your site."
2. Choose the **HTML tag** method, not "HTML file" — the file method
   expects something uploaded outside your normal build, which doesn't
   fit a workflow where everything ships through Jekyll. Copy just the
   `content="..."` value into the config above.
3. Once the site rebuilds with that value live, go back to Search
   Console and verify. `jekyll-seo-tag` renders the meta tag for you —
   nothing else to configure.
4. Repeat for [Bing Webmaster Tools](https://www.bing.com/webmasters),
   or skip the manual step: once Google is verified, use Bing's
   **Import from Google Search Console** option instead.
5. In both tools, submit `sitemap.xml` under their respective Sitemaps
   sections — already generated automatically by the `jekyll-sitemap`
   plugin, nothing to build. Bing doesn't inherit the Google submission
   even through the import, so it needs submitting on its own.
   Processing can take anywhere from minutes to a day, so a "couldn't
   fetch" or "pending" status right after submitting isn't cause for
   alarm — check back before assuming something's broken.

### Decide on comments deliberately

The `comments` block in `_config.yml` supports Disqus, Utterances, or
Giscus, but ships with `provider:` empty — meaning comments stay off
by default, even though individual posts default to `comments: true`.
If you want them on, pick a provider and fill in its section. Giscus,
backed by GitHub Discussions on your own repo, is a solid default for
a technical audience: no third-party account beyond GitHub, no ads, no
tracking.

One thing worth knowing before you pick: **giscus has no pre-publish
moderation.** It's a thin UI over GitHub Discussions — a commenter
authenticates with their own GitHub account and posts a real
Discussion reply directly, live and public the moment they submit it.
There's no "hold for review" step anywhere in that path, because
GitHub Discussions itself doesn't have one; giscus can't add a queue
on top of infrastructure it doesn't control. The same is true of
Utterances, which works the same way against GitHub Issues instead.

What both give you is moderation *after* the fact: delete any comment
as a repo maintainer, lock a thread, block a user, or use GitHub's own
abuse reporting. Requiring a real GitHub sign-in does cut down on
anonymous drive-by trolling, but it doesn't stop a bad comment from
being visible to every reader until you personally catch and remove
it.

Disqus is the one of the three with a genuine pre-approval queue — it
can require every comment to be moderator-approved before it appears
at all (Disqus dashboard → Settings → General). That capability is
also its trade-off: third-party ads and tracking by default, and an
external account for commenters instead of the GitHub identity most of
this site's audience already has. If keeping abusive or derogatory
comments from ever going live matters more than the ad-free,
GitHub-native experience, that trade-off points at Disqus, not Giscus,
despite Giscus otherwise being the better fit for a technical
audience.

If Giscus is still the right call for you, enabling it is a short
checklist:

1. **Enable Discussions** on your repo — Settings → Features → check
   "Discussions." It's off by default.
2. **Create a Discussion category** for comments — Discussions tab →
   new category, type "Announcement" so only maintainers can start
   top-level discussions in it. Giscus creates one per post
   automatically; you don't want random top-level posts landing there.
3. **Install the [giscus GitHub App](https://github.com/apps/giscus)**
   on the repo — this is what lets giscus read and write discussion
   threads on your behalf.
4. **Go to [giscus.app](https://giscus.app)**, enter your repo, and
   select that category. It generates the exact `repo`, `repo_id`,
   `category`, and `category_id` values for the next step.
5. **Fill in `_config.yml`'s `giscus:` block with those values, and
   set `comments.provider: giscus`.**
6. **Verify:** once the site rebuilds, any post — comments default to
   `true` — should show the giscus widget at the bottom, backed by
   that Discussion category.

### Leave `social.email` blank unless you want it public

The sidebar's contact icon only shows up if `social.email` is set, and
when it is, the theme applies light obfuscation — splitting the
address across two JavaScript strings joined only when clicked, rather
than a plain `mailto:` link sitting in the page source. That's enough
to dodge the crudest scrapers; it is not real privacy. Treat it as
"slightly harder to scrape," not "hidden." If you'd rather not expose
any address, leave the field blank — the theme skips the icon
entirely rather than rendering something broken or fake-looking.
