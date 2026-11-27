![nscrub_logo]
# nScrub
This repository contains open source components and extensions for nScrub available at http://www.ntop.org/products/ddos-mitigation/nscrub/

Here you can find:
* SDK for developing Plugins
* Documentation

Enjoy!

[nscrub_logo]: https://www.ntop.org/images/2016/04/nscrub-logo.webp

### Documentation

If you want to learn more about nScrub please visit the [User's Guide](https://www.ntop.org/guides/nScrub/).


### Building the User's Guide

The User's Guide is a [VitePress](https://vitepress.dev) project in `doc/` (Markdown pages, same
URLs as the old Sphinx guide: `/guides/nscrub/<page>.html`). Requires Node 18+ and `make`.

```
cd doc
make dist      # build the static site into doc/dist (runs npm install if needed)
make serve     # build, then serve it at http://localhost:4173/guides/nscrub/   (PORT=8080 make serve)
```

For live reload while editing a page use `npm run dev` (http://localhost:5173/guides/nscrub/).

### Details
For more information about nScrub, please visit [http://ntop.org](https://www.ntop.org/producs/ddos-mitigation/nscrub/).
