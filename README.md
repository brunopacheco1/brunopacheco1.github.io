# bruno.pacheco.lu

Personal blog of Bruno Pacheco, built with [Hugo](https://gohugo.io) and the
[Congo](https://github.com/jpanther/congo) theme, in English and Portuguese, and deployed to GitHub
Pages from `master`.

## Writing and previewing
Hugo downloads the theme as a Hugo module, which needs Go:

```bash
brew install hugo go
hugo server -D        # http://localhost:1313/, drafts included; /pt/ for Portuguese
```

Posts are `content/posts/<slug>.en.md` and `<slug>.pt.md`; images go in `static/images/<slug>/`.

## Publishing
Open a pull request: the `Build and deploy the site` workflow builds the site as a check. Merging
to `master` builds and deploys it.

## Versions
- **Hugo:** 0.169.0. CI pins it as `HUGO_VERSION` in `.github/workflows/hugo.yml`; Congo 2.14 needs
  0.158 or later.
- **Congo:** 2.14.0 (`go.mod`). Update it with `hugo mod get github.com/jpanther/congo/v2@<version>`;
  Dependabot proposes theme and action updates weekly.
