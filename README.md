# tomwrw.com

The source for my personal website, [tomwrw.com](https://tomwrw.com) - notes on Linux, open
source and security.

Built with [Hugo](https://gohugo.io) and my own theme,
[austere](https://github.com/tomwrw/austere-theme-hugo), which is included here as a git
submodule. Published to GitHub Pages by
[a workflow](.github/workflows/build-deploy.yml) on every push to `main`.

## Local development

Requires the **extended** edition of Hugo, 0.146.0 or newer. CI builds with the version
pinned in `HUGO_VERSION` in the workflows.

```bash
git clone --recurse-submodules https://github.com/tomwrw/tomwrw-com.git
cd tomwrw-com
hugo server
```

If you have already cloned without the submodule:

```bash
git submodule update --init --recursive
```

A production build, matching what CI does:

```bash
hugo --gc --minify --panicOnWarning --printPathWarnings
```

## Writing

```bash
hugo new posts/my-post.md
```

Posts live in `content/posts/`, other pages anywhere in `content/`. Keep a post's images
beside it in a page bundle and the theme handles resizing and `srcset`:

```
content/posts/a-walk/
├── index.md
└── hill.jpg
```

## Licence

The site content is © tomwrw. The theme is MIT licensed - see the
[theme repository](https://github.com/tomwrw/austere-theme-hugo).
