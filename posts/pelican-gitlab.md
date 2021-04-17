Title: Host a Pelican static website on GitLab Pages
Slug: pelican-gitlab
Date: 2021-04-14 22:05
Modified: 2021-04-14 22:05
Authors: Mark Mulligan
Category: Pelican
Tags: cicd, gitlab, pelican
Summary: Overview of my GitLab configuration for publishing static websites directly from GitLab.


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
