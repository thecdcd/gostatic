# gostatic

Gostatic is a static site generator. What differs it from most of other tools is
that it's written in Go and tracks changes, which means it should work
reasonably [fast](#speed).

Features include:

 - No run-time dependencies, just a single binary - download it and run
 - Dependency tracking and re-rendering only changed pages
 - Markdown support
 - Flexible [filter system](#processors)
 - Simple [config syntax](#configuration)
 - HTTP server and watcher (instant rendering on changes)

Binary builds:

| Linux         | OS X          | Windows       |
|:--------------|:--------------|:--------------|
| [64 bit][l64] | [64 bit][x64] | [64 bit][w64] |
| [32 bit][l32] | [32 bit][x32] | [32 bit][w32] |

[l64]: http://solovyov.net/files/gostatic-64-linux
[l32]: http://solovyov.net/files/gostatic-32-linux
[x64]: http://solovyov.net/files/gostatic-64-osx
[x32]: http://solovyov.net/files/gostatic-32-osx
[w64]: http://solovyov.net/files/gostatic-64-win.exe
[w32]: http://solovyov.net/files/gostatic-32-win.exe

If you want to download specific version, url pattern is
`http://solovyov.net/files/gostatic-<version>-<32/64>-<win.exe/linux/osx>`.

## Quick Start

Run `gostatic -i my-site` to generate basic site in directory `name`. It will
have a basic `config` file, which you should edit to put relevant variables at
the top - it also contains description of how files in your `src` directory are
treated.

`src` directory obviously contains sources of your site (name of this directory
can be changed in `config`). You can follow general idea of this directory to
create new blog posts or new pages. All files, which are not mentioned in
`config`, are just copied over. Run `gostatic -fv config` to see how your `src`
is processed.

`site.html` is a file that defines templates your are able to use for your
pages. You can see those templates mentioned in `config`.

And, finally, there is a `Makefile`, just for convenience. Run `make` to build
your site once or `make w` to run watcher and server, to see your site changes
in real time.

Also, you could look at [my site](https://github.com/piranha/solovyov.net) for
an example of advanced usage.

Good luck! And remember, your contributions either to gostatic or to
documentation (even if it's just this `README.md`) are always very welcome!

## Documentation index:

- [Approach](#approach)
- [Speed](#speed)
- [External Resources](#external-resources)
- [Configuration](#configuration)
  - [Constants](#constants)
- [Page config](#page-config)
- [Processors](#processors)
- [Templating](#templating)
  - [Global Functions](#global-functions)
  - [Page interface](#page-interface)
  - [Page list interface](#page-list-interface)
  - [Site interface](#site-interface)


## Approach

Each given file is processed through a pipeline of filters, which modify the
state and then rendered on disk. Single input file corresponds to a single
output file, but filters can generate virtual input files.

Each file can have dependencies, and will be rendered in case it does not exist,
or its source is newer than output, or one of this is the case for one of its
dependencies.

All read pages are sorted by date. This date is taken in their
[config](#page-config) or, in case if config is absent or dates there are equal,
by file modification time.

## Speed

On late 2008 MacBook (C2D 2.4 GHz, 8 GB RAM, 5400 rpm HDD) it takes `0.3s` to
generate a site of 250 pages. It costs `0.05s` to check there are no
modifications and `0.1s` to re-render a single changed page (along with index
and tag pages, coming to 77 pages in total).

## External resources

 - Jack Pearkes wrote [Heroku buildpack][] for gostatic and an
   [article about it][].

[Heroku buildpack]: https://github.com/pearkes/heroku-buildpack-gostatic
[article about it]: http://pretengineer.com/post/gostatic-buildpack-for-heroku/

## Configuration

Config syntax is Makefile-inspired with some simplifications, look at the
example:

```Makefile
TEMPLATES = site.tmpl
SOURCE = src
OUTPUT = site

# this is a comment
*.md:
    config
    ext .html
    directorify
    tags tags/*.tag
    markdown
    template page # yeah, this is a comment as well

index.md: blog/*.md
    config
    ext .html
    inner-template
    markdown
    template page

*.tag: blog/*.md
    ext .html
    directorify
    template tag
    markdown
    template page
```

Here we have constants declaration (first three lines), a comment and then three
rules. One for any markdown file, one specifically for index.md and one for
generated tags.

Note: Specific rules override matching rules, but there is no very smart logic
in place and matches comparisons are not strictly defined, so if you have
several matches you could end up with any of them. Though there is order: exact
path match, exact name match, glob path match, glob name match. NOTE: this may
change in future.

Rules consist of path/match, list of dependencies (also paths and matches, the
ones listed after colon) and commands.

Each command consists of a name of processor and (possibly) some
arguments. Arguments are separated by spaces.

Note: if a file has no rules whatsoever, it will be copied to exactly same
location at destination as it was in source without being read into memory. So
heavy images etc shouldn't be a problem.

### Constants

There are three configuration constants. `SOURCE` and `OUTPUT` speak for
themselves, and `TEMPLATES` is a list of files which will be parsed as Go
templates. Each file can contain few templates.

You can also use arbitrary names for constants to
[access later](#site-interface) from templates.

## Page config

Page config is only processed if you specify `config` processor for a page. It's
format is `name: value`, for example:

```
title: This is a page
tags: test
date: 2013-01-05
```

Parsed properties:

- `title` - page title.
- `tags` - list of tags, separated by `,`.
- `date` - page date, could be used for blog. Accepts formats from bigger to
  smaller (from `"2006-01-02 15:04:05 -07"` to `"2006-01-02"`)

You can also define any other property you like, it's value will be treated as a
string and it's key is capitalized and put on the `.Other`
[page property](#page-interface).

## Processors

You can always check list of available processors with `gostatic --processors`.

- `config` - reads config from content. Config should be in format "name: value"
  and separated by four dashes on empty line (`----`) from content.

- `ignore` - ignore file.

- `rename <new-name>` - rename a file to `new-name`. New name can contain `*`,
  then it will be replaced with whatever `*` captured in path match. Right now
  rename touches **whole** path, so be careful (you may need to include whole
  path in rename pattern) - *this may change in future*.

- `ext <.ext>` - change file extension to a given one (which should be prefixed
  with a dot).

- `directorify` - rename a file from `whatever/name.html` to
  `whatever/name/index.html`.

- `markdown` - process content as Markdown.

- `inner-template` - process content as Go template.

- `template <name>` - pass page to a template named `<name>`.

- `tags <path-pattern>` - create (if not yet) virtual page for all tags of a
  current page. This tag page has path formed by replacing `*` in
  `<path-pattern>` with a tag name.

- `relativize` - change all urls archored at `/` to be relative (i.e. add
  appropriate amount of `../`) so that generated content can be deployed in a
  subfolder of a site.

- `external <command> <args...>` - call external command with content of a page
  as stdin and using stdout as a new content of a page. Has a shortcut:
  `:<command> <args...>` (`:` is replaced with `external `).

## Templating

Templating is provided using
[Go templates](http://golang.org/pkg/text/template/). See link for documentation
on syntax.

Each template is executed in context of a page. This means it has certain
properties and methods it can output or call to generate content, i.e. `{{
.Content }}` will output page content in place.

### Global functions

Go template system provides some convenient
[functions](http://golang.org/pkg/text/template/#hdr-Functions), and gostatic
expands on that a bit:

 - `changed <name> <value>` - checks if value has changed since previous call
   with the same name. Storage, used for checking, is global over whole run of
   gostatic, so choose unique names.

 - `cut <value> <begin> <end>` - cut partial content from `<value>`, delimited
   by regular expressions `<begin>` and `<end>`.

 - `hash <value>` - return 32-bit hash of a given value.

 - `version <path>` - return relative url to a page with resulting path `<path>`
   with `?v=<32-bit hash>` appended (used to override cache settings on static
   files).

### Page interface

- `.Site` - global [site object](#site-interface).
- `.Rule` - rule object, matched by page.
- `.Pattern` - pattern, which matched this page.
- `.Deps` - list of pages, which are dependencies for this page.
- `.Next` - next page in a list of all site pages (use specific PageSlice's
  `.Next` method if you need more precise matching).
- `.Prev` - previous page in a list of all site pages (use specific PageSlice's
  `.Prev` method if you need more precise matching).

----

- `.Source` - relative path to page source.
- `.FullSource` - full path to page source.
- `.Path` - relative path to page destination.
- `.FullPath` - full path to page destination.
- `.ModTime` - page last modification time.

----

- `.Title` - page title.
- `.Tags` - list of page tags.
- `.Date` - page date, as defined in [page config](#page-config).
- `.Other` - map of all other properties (capitalized) from
  [page config](#page-config).

----

- `.Content` - page content.
- `.Url` - page url (i.e. `.Path`, but with `index.html` stripped from the end).
- `.UrlTo <other-page>` - relative url from current to some other page.
- `.Rel <url>` - relative url to given absolute (anchored at `/`) url.
- `.Is <url>` - checks if page is at passed url (or path) - use it for marking
  active elements in menu, for example.
- `.UrlMatches <pattern>` - checks if page url matches regular expression
  `<pattern>`.

### Page list interface

- `.Get <n>` - [page](#page-interface) number `<n>`.
- `.First` - first page.
- `.Last` - last page.
- `.Len` - length of page list.
- `.Prev <page>` - return page with earlier date than given. Returns nil if no
  earlier pages exist or page is not in page list.
- `.Next <page>` - return page with later date than given. Returns nil if no
  later pages exist or page is not in page list.

----

- `.Children <prefix>` - list of pages, nested under `<prefix>`.
- `.WithTag <tag-name>` - list of pages, tagged with `<tag-name>`.

----

- `.BySource <path>` - finds a page with source path `<path>`.
- `.ByPath <path>` - finds a page with resulting path `<path>`.

### Site interface

- `.Pages` - [list of all pages](#page-list-interface).
- `.Source` - path to site source.
- `.Output` - path to site destination.
- `.Templates` - list of template files used for the site.
- `.Other` - any other properties (capitalized) defined in site config.
