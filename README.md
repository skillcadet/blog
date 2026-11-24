# Skillcadet blog

The source for [blog.skillcadet.com](https://blog.skillcadet.com), published with GitHub Pages and Jekyll.

## Preview locally with Docker

You only need Docker Desktop (with Docker Compose). No local Ruby, Jekyll, or Homebrew installation is required.

From the repository directory:

```sh
docker compose up
```

The first run downloads a Ruby image and installs the site's Ruby gems; it may take several minutes. Subsequent runs reuse the `bundle-cache` Docker volume.

Open [http://localhost:4000](http://localhost:4000) to see the site. Changes to Markdown and layouts should trigger a rebuild; refresh your browser if needed. Press **Ctrl+C** to stop, or run `docker compose down` in another terminal.

The local preview uses Jekyll 4 and the site's configured plugins. GitHub Pages may use different versions, so the local preview is not an exact reproduction of its build environment. The `jekyll-remote-theme` plugin downloads the Minima theme during the build, so an internet connection is needed.

To preview a pull request branch:

```sh
git fetch origin
git switch editorial/site-introduction
docker compose up
```

Before switching branches, commit or stash any uncommitted changes you want to preserve.

If you see a dependency error, copy the full Docker output into the issue or PR; don't install Ruby on your Mac to fix it.
