# corporate-website

Corporate website source powered by Hugo and themed by min_night.

* https://gohugo.io/
* https://themes.gohugo.io/min_night/

## Install Hugo

See Install Hugo:

* https://gohugo.io/getting-started/installing/

Confirm that you can run `hugo` command and the version.

```bash
$ hugo version
Hugo Static Site Generator v0.63.0-DEV linux/amd64 BuildDate: unknown
```

## Checkout repository

Clone this repository.

```bash
$ git clone git@github.com:kazamori/corporate-website.git
$ cd corporate-website/
```

### Run only once after the repository was cloned

This repository include Hugo's theme as `submodule`.
So you have to initialize/update theme repository for the first time when you cloned this repository.

```bash
$ git submodule update --init
```

Confirm `min_night` theme files are cloned.

```bash
$ ls kazamori/themes/min_night
LICENSE		README.md	exampleSite	layouts		theme.toml
NEWS.md		archetypes	images		static
```

## Build static site

Build contents and run as server.
The `server` command is useful since it automatically rebuild when you modify a content.

```bash
$ cd kazamori
$ hugo server
                   | EN
+------------------+----+
  Pages            | 24
  Paginator pages  |  0
  Non-page files   |  0
  Static files     | 16
  Processed images |  0
  Aliases          |  7
  Sitemaps         |  1
  Cleaned          |  0

Built in 78 ms
Watching for changes in path/to/corporate-website/kazamori/{archetypes,content,data,layouts,static,themes}
Watching for config changes in path/to/corporate-website/kazamori/config.toml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

Open http://localhost:1313/ via web browser.

## Deploy

To deploy static site distribution into any directory is like this.

```bash
$ hugo --destination path/to/html/directory
```

## License

Copyright © Kazamori LLC for contents, however some source code is come from Hugo and min_night theme and comply with their licenses.
