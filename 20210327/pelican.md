Title: Hosting a static website on GitLab Pages with Pelican
Date: 2021-03-27 10:20
Modified: 2021-03-28 19:30
Category: Pelican
Tags: cicd, gitlab, pelican
Slug: pelican-setup
Authors: Mark Mulligan
Summary: Walk through Pelican installation and configuration for publishing static websites directly from a GitLab repository.

### <span style="color:red">**DISCLAIMER:**</span> This document shows how easy it is to get started with Pelican but is not intended to replace [the official Pelican documentation](https://docs.getpelican.com/en/stable/index.html)

#### **TL;DR**

1. Install system dependencies, e.g. for Debian based systems:

        :::shell
        sudo apt install git python3-pip

2. Clone my repository/submodules and install pelican dependencies:

        :::shell
        git clone --recurse-submodules git@gitlab.com:muxync/pelican-blog.git
        cd pelican-blog
        sudo pip3 install -r requirements.txt

3. Generate and serve static site

        :::shell
        make publish
        make serve
        # View at http://127.0.0.1:8000

4. Replace *my code* with your own:

        :::shell
        ${EDITOR} pelicanconf.py
        # Make changes to AUTHOR/SITENAME/etc and save
        # Repeat step 3

5. Replace *my content* with your own:

        :::shell
        # Remove the existing content submodule
        git submodule deinit content
        git rm content
        rm -rf .git/modules/content

        # Add your new content submodule
        git submodule add ../pelican-content.git content
        git submodule sync --recursive
        git submodule update --init --recursive
        # Repeat step 3

6. Update *your content* within *your code* repo:

        :::shell
        git submodule update --remote --merge

# Intro

For my first post I'll show how you can setup and host your own static website built with [Pelican](https://getpelican.com) and hosted by [GitLab](https://about.gitlab.com).  If you want to you can learn more [about me](pages/about.html) or you can clone this repo and try it out yourself.

I have decided to license [my code](https://gitlab.com/muxync/pelican-blog) under the [MIT License](https://en.wikipedia.org/wiki/MIT_License) which will match the [pelican bootstrap3 theme](https://github.com/getpelican/pelican-themes/blob/master/pelican-bootstrap3) I'm using but [my content](https://gitlab.com/muxync/pelican-content) I will license under the [CC0 1.0 Universal (CC0 1.0) Public Domain Dedication](https://creativecommons.org/publicdomain/zero/1.0) so you can do with it what you please, no attribution required.

This repo uses the following repos as git submodules:

  - [pelican-themes](https://gitlab.com/muxync/pelican-themes)
  - [pelican-plugins](https://gitlab.com/muxync/pelican-plugins)
  - [pelican-content](https://gitlab.com/muxync/pelican-content)

Please see the respective licenses for `pelican-themes`/`pelican-plugins` as those will vary depending on what you use.

# Install

Start by installing system dependencies, e.g. for Debian based systems:

    :::shell
    sudo apt install git python3-pip

Then use git to recursively clone my repo:

    :::shell
    git clone --recurse-submodules git@gitlab.com:muxync/pelican-blog.git

Go to the `pelican-blog` directory:

    :::shell
    cd pelican-blog

View the Python dependencies

    :::shell
    cat requirements.txt

You should see something like:

    :::shell
    beautifulsoup4
    markdown
    pelican
    typogrify

Then install them:

    :::shell
    sudo pip3 install -r requirements.txt

# Usage

Generate the HTML for my static site:

    :::shell
    make publish  # or 'pelican' if you prefer

Serve the static site:

    :::shell
    make serve  # or 'pelican --listen' if you prefer

Open a web browser to view the static site at [http://127.0.0.1:8000](http://127.0.0.1:8000)

# Modifications

## Code

You can use my pelican config as a starting point you and replace *my code* with your own:

    :::shell
    ${EDITOR} pelicanconf.py

Make changes to `AUTHOR`/`SITENAME`/etc and save.  You should check out [pelicanthemes.com](http://pelicanthemes.com) for a `THEME` other than `pelican-bootstrap3` if you want (there are several in the `themes` directory which itself points to the [pelican-themes repo](https://github.com/getpelican/pelican-themes) as does the [pelican-plugins repo](https://github.com/getpelican/pelican-plugins)).  Otherwise you can see some [Bootstrap themes](https://bootswatch.com) and use it with `BOOTSTRAP_THEME` (I went with "simplex").  Sadly not all Bootstrap themes are available by default but you can add more to the `themes/pelican-bootstrap3/static/css/` directory or you can see what themes are included with: 

    :::shell
    ls themes/pelican-bootstrap3/static/css/bootstrap.*

Additionally you can set `PYGMENTS_STYLE` for the syntax highlighting style.  A [demo site](https://pygments.org/demo) is available for your reference (I went with "monokai" which happens to match my [Vim text editor](https://www.vim.org) theme).

I'm not an artist so I found some Creative Commons Zero 1.0 Public Domain Licensed art to use for `BANNER`/`SITELOGO`/`FAVICON`.  I got [my banner art](https://openclipart.org/detail/202226/banner-5) and [site logo](https://openclipart.org/detail/27563/little-penguin) at [openclipart.org](https://openclipart.org/share) and [my favicon](https://freefavicon.com/freefavicons/animal/iconinfo/little-penguin-152-27563.html) at [freefavicon.com](https://freefavicon.com/about) (Free Favicon itself sources the majority of its images from Open Clipart).

## Content

You can use my blog as a starting point you can replace *my content* with your own.  Remove the existing `content` git submodule:

    :::shell
    # Remove the existing content submodule
    git submodule deinit content
    git rm content
    rm -rf .git/modules/content

Replace it with your own `content` git submodule (assuming you have a similarly named repo available for your user at `../pelican-content.git`):

    :::shell
    # Add your new content submodule
    git submodule add ../pelican-content.git content
    git submodule sync --recursive
    git submodule update --init --recursive

Then you can update *your content* (`pelican-content`) within *your code* (`pelican-blog`) repo the next time you push a change to your `pelican-content` repo:

    :::shell
    git submodule update --remote --merge

If that seems overly complex you could instead replace the `content` git submodule with a directory containing your content.  I went with this approach because it keeps my code and content separate so I can license them differently.  It works well for me and the GitLab CI makes it painless to manage (I'll cover that another time).


```
mkdir mirrors
cd mirrors/
git clone https://github.com/gfidente/pelican-svbhack
rm -rf pelican-svbhack/
git clone git@gitlab.lan:mirrors/pelican-svbhack.git
cd pelican-svbhack/
git remote -v
git remote add upstream https://github.com/gfidente/pelican-svbhack.git
git fetch --all --prune
git checkout upstream/master -b upstream
git push origin --set-upstream upstream
git remote -v
git branch -va
git remote set-url --push git@gitlab.lan:mirrors/pelican-svbhack.git
git remote set-url --push upstream git@gitlab.lan:mirrors/pelican-svbhack.git
git remote -v
git pull
git push
..
cd pelican-svbhack/
git remote -v
git remote set-url master git@gitlab.lan:d/pelican-theme-svbhack.git
git remote set-url origin git@gitlab.lan:d/pelican-theme-svbhack.git
git remote -v
git pull
```
