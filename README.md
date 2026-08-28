# Security Research Blog — Setup

Hugo + PaperMod static site for publishing CVE advisories, auto-deployed to GitHub Pages.

## One-time setup

### 1. Create the repo
- New public GitHub repo. For a root user site, name it `USERNAME.github.io`.
  Otherwise any name works (site served at `USERNAME.github.io/reponame`).

### 2. Local scaffold
```bash
# Install Hugo extended locally (needed for PaperMod)
#   macOS:   brew install hugo
#   Linux:   see github.com/gohugoio/hugo/releases (get the *extended* build)

git clone https://github.com/USERNAME/USERNAME.github.io.git
cd USERNAME.github.io
# copy the contents of this deliverable into the repo, then:

# add PaperMod theme as a submodule
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 3. Configure
Edit `hugo.toml`: replace every `USERNAME` and `YOUR NAME` placeholder, and set
the correct `baseURL`:
- Root user site  -> `https://USERNAME.github.io/`
- Project site    -> `https://USERNAME.github.io/reponame/`

### 4. Enable Pages
Repo **Settings -> Pages -> Build and deployment -> Source: GitHub Actions**.

### 5. Push
```bash
git add .
git commit -m "init security blog"
git push origin main
```
The `deploy.yml` workflow builds and publishes automatically on every push to `main`.

## Writing a new advisory

```bash
hugo new posts/cve-2026-12345-sqli-acme.md   # generates front matter from archetype
```
Or copy `content/posts/cve-XXXX-XXXXX-template.md`. Set `draft = false` when ready.

Preview locally before pushing:
```bash
hugo server -D    # http://localhost:1313, -D includes drafts
```

## Cost
Fully free: Hugo (OSS), GitHub Pages hosting (public repos), GitHub Actions
(free tier is ample for a static site). A custom domain (~10 EUR/yr) is optional.
