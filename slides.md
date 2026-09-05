---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
hideInToc: true
title: Container Training
author: Mischa Taylor
info: |
  ## Slidev Starter Template
  Presentation slides for developers.
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# open graph
# seoMeta:
#  ogImage: https://cover.sli.dev
themeConfig:
  paginationX: r
  paginationY: t
  paginationPagesDisabled: [1]
---

# Container Training

##### Mischa Taylor | 📧 <taylor@linux.com>

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Press Space for next page <carbon:arrow-right />
</div>

<div class="abs-br m-6 text-xl">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="slidev-icon-btn">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
hideInToc: true
routeAlias: toc
---

# Table of Contents

<Toc columns="2"/>

---
layout: section
---

# Apple Containerization on macOS

<br>
<br>
<Link to="toc" title="Table of Contents"/>

---
hideInToc: true
---

# container

`container` is Apple's command-line tool for running Linux containers on macOS.

You need a mac with Apple silicon and macOS 26 (Tahoe) or later to run `container`.

`container` does not come with macOS, you must download the latest signed
installer from https://github.com/apple/container/releases

---
hideInToc: true
---

# Check requirements

Check to make sure that your machine is an Apple Silicon mac.

```bash
# Architecture should be arm64
% uname -m
arm64
```

And make sure you are running at least macOS 26 (Tahoe).

```bash
# Should be at least ProductVersion 26
% sw_vers
ProductName:		macOS
ProductVersion:		26.6.2
BuildVersion:		25G83
```

---

# Installing container with gh

If you have the GitHUB cli installed:

```bash
mkdir -p /tmp/apple-container
cd /tmp/apple-container

gh release download \
  --repo apple/container \
  --pattern '*installer-signed.pkg'

sudo installer \
  -pkg container-*-installer-signed.pkg \
  -target /

container system start
```

---

# Installing container with curl

If you do not have the GitHUB cli installed, you can also download the
installer entirely with `curl`, using the GitHub API.

```bash
mkdir -p /tmp/apple-container
cd /tmp/apple-container

url=$(
  curl -s https://api.github.com/repos/apple/container/releases/latest |
  jq -r '.assets[]
         | select(.name | endswith("installer-signed.pkg"))
         | .browser_download_url'
)

curl -L -O "$url"

sudo installer \
  -pkg container-*-installer-signed.pkg \
  -target /
```

---

# Uninstalling container

To uninstall the Apple Container CLI tool, use the uninstall script
located in `/usr/local/bin`.

```bash
# Keep User Data
/usr/local/bin/uninstall-container.sh -k

# Delete all user data
/usr/local/bin/uninstall-container.sh -d
```