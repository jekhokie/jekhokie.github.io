---
layout: post
title:  "PlantUML Using vim"
date:   2020-07-24 23:02:00 -0400
categories: linux vim plantuml
logo: plantuml.jpg
---

[PlantUML](https://plantuml.com/) is a great modeling language/utility for software diagrams (and other types). Often, it's useful to have
a pattern of developing using `vim` and having the UML update automatically. This post details a quick way to offer this pattern.

### OS

This tutorial is written from the perspective of a Mac OS - while it may work on other operating systems, the commands would need to be
adapted to the specific OS expectations.

### System Dependencies

There are a couple of system dependencies that are required to get going:

```bash
$ brew install graphviz
$ brew install java
```

### Pathogen Dependencies

Next, we'll install the Pathogen (vim) dependencies. Again, this assumes you're using Pathogen as the package manager for vim - if not,
ensure you use whatever mechanism you typically use for package/feature management for vim:

```bash
$ cd ~/.vim/bundle/
$ git clone https://github.com/aklt/plantuml-syntax
$ git clone https://github.com/tyru/open-browser.vim.git
$ git clone https://github.com/weirongxu/plantuml-previewer.vim.git
```

### Using the PlantUML Renderer

You're now set up to render/auto-update your PlantUML diagrams generated using vim. Open a new file in vim (e.g. `example.puml`) and
add the following contents:

```bash
@startuml
Alice -> Bob: test
@enduml
```

Save the file, and then render it in a browser window using the ex command in vim: `:PlantumlOpen`. Your browser (default) should open and after
a few seconds, the above PlantUML should render in an image. Update your vim file and save the contents, you should see the browser image auto-update.
You're now set up to do some nice PlantUML creation with auto-updating images for inspection!

### Credit

The above tutorial was pieced together with some information from the following sites/resources, among others that were likely missed in this list:

* [PlantUML Syntax](https://github.com/aklt/plantuml-syntax)
* [Open Browser](https://github.com/tyru/open-browser.vim)
* [PlantUML Previewer](https://github.com/weirongxu/plantuml-previewer.vim)
