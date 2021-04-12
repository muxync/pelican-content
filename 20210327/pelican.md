Title: Hosting a static website on GitLab Pages with Pelican
Date: 2021-03-27 10:20
Modified: 2021-03-28 19:30
Category: Pelican
Tags: cicd, gitlab, pelican
Slug: pelican-setup
Authors: Mark Mulligan
Summary: Walk through Pelican installation and configuration for publishing static websites directly from a GitLab repository.

**TL;DR**

1. Install system dependencies, e.g. for Debian based systems:
```
sudo apt install git python3-pip
```
2. Clone the repository/submodules and install pelican dependencies:
```
git clone --recurse-submodules git@gitlab.com:muxync/pelican-blog.git
cd pelican-blog
sudo pip3 install -r requirements.txt
```
3. Generate and serve static site
```
make publish
make serve
# View at http://127.0.0.1:8000
```
4. Replace my *code* with your own:
```
${EDITOR} pelicanconf.py
# Make changes to AUTHOR/SITENAME/etc and save
# Repeat step 3
```
5. Replace my *content* with your own:
```
# Remove the existing content submodule
git submodule deinit content
git rm content
rm -rf .git/modules/content

# Add your new content submodule
git submodule add ../pelican-content.git content
git submodule sync --recursive
git submodule update --init --recursive

# Repeat step 3
```

# Intro
For my first post I'll explain how you can setup and host your own static website built with [Pelican](https://getpelican.com) and hosted by [GitLab](https://about.gitlab.com).  If you want to you can learn more [about me](pages/about.html) or you can [clone this repo](https://gitlab.com/muxync/pelican-blog.git) and try it out yourself.

I have decided to license the aforementioned code under the [MIT License](https://en.wikipedia.org/wiki/MIT_License) which will match the [pelican bootstrap3 theme](https://github.com/getpelican/pelican-themes/blob/master/pelican-bootstrap3) I'm using but [my content](https://gitlab.com/muxync/pelican-content) I will license under the [CC0 1.0 Universal (CC0 1.0) Public Domain Dedication](https://creativecommons.org/publicdomain/zero/1.0) so you can do with it what you please, no attribution required.


`cat requirements.txt`
```
markdown
pelican
typogrify
```

```
sudo pip3 install -r requirements.txt
```


```
make publish
pelican --listen
```



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


```
#git submodule add ../pelican-theme-svbhack.git theme
git submodule add ../pelican-themes.git themes
git submodule update --remote

```


```
git clone https://github.com/getpelican/pelican-themes.git

```
https://github.com/getpelican/pelican-themes/tree/master/pelican-bootstrap3


https://bootswatch.com
https://pygments.org/demo

I'm not an artist so I found some Creative Commons Zero 1.0 Public Domain Licensed art.  I found [my banner art](https://openclipart.org/detail/202226/banner-5) at [openclipart.org](https://openclipart.org/share) and [my favicon](https://freefavicon.com/freefavicons/animal/iconinfo/little-penguin-152-27563.html) at [freefavicon.com](https://freefavicon.com/about) (Free Favicon itself sources the majority of its images from Open Clipart).

