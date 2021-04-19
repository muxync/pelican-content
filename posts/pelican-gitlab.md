Title: Host a Pelican static website on GitLab Pages
Slug: pelican-gitlab
Date: 2021-04-14 22:05
Modified: 2021-04-14 22:05
Authors: Mark Mulligan
Category: Pelican
Tags: cicd, gitlab, pelican
Summary: Overview of my git configuration for publishing static websites directly from GitLab.


# Intro
For my [static site](pelican-create.html) I have been using my [Self-Managed GitLab instance](gitlab-docker-compose.html) as a staging environment before pushing my code to the [public gitlab.com endpoint](blog.muxync.net).  Part of my static site relies on git submodules for the `content`, `plugins`, and `themes` directories.  The `content` submodule contains my original content (posts, images, etc) but the `plugins` and `themes` directories are mirrors of upstream `pelican` projects that I created under my GitLab user with the [import repo by URL](https://docs.gitlab.com/ee/user/project/import/repo_by_url.html) functionality.

# Upstream
After initially adding a submodule (e.g. `themes`) to my [pelican-blog repo](https://gitlab.com/muxync/pelican-blog) with:

    :::shell
    git submodule add ../pelican-themes.git themes
    git submodule sync --recursive
    git submodule update --init --recursive

I added a new remote named `upstream` to make it easy to periodically sync upstream changes into my site.  The commands I ran to do that are:

    :::shell
    # From the pelican-blog/themes directory
    git remote -v
    git remote add upstream https://github.com/getpelican/pelican-themes.git
    git fetch --all --prune
    git checkout upstream/master -b upstream
    git branch -v
    git push origin --set-upstream upstream
    git remote set-url origin git@gitlab.lan:muxync/pelican-themes.git
    git remote -v
    git remote set-url --push upstream git@gitlab.lan:muxync/pelican-themes.git
    git remote -v
    git pull
    git push
    git checkout master

and here is the output of those commands:

    :::shell
    ~/Workspace/pelican-blog/themes (master 65dd78d)$ git remote -v
    origin	git@gitlab.lan:muxync/pelican-themes.git (fetch)
    origin	git@gitlab.lan:muxync/pelican-themes.git (push)
    ~/Workspace/pelican-blog/themes (master 65dd78d)$ git remote add upstream https://github.com/getpelican/pelican-themes.git
    ~/Workspace/pelican-blog/themes (master 65dd78d)$ git fetch --all --prune
    Fetching origin
    Fetching upstream
    From https://github.com/getpelican/pelican-themes
     * [new branch]      master     -> upstream/master
     * [new branch]      previews   -> upstream/previews
    ~/Workspace/pelican-blog/themes (master 65dd78d)$ git checkout upstream/master -b upstream
    Branch 'upstream' set up to track remote branch 'master' from 'upstream'.
    Switched to a new branch 'upstream'
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git branch -v
       master   65dd78d Merge pull request #705 from aleylara/master
    *  upstream 65dd78d Merge pull request #705 from aleylara/master
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git push origin --set-upstream upstream
    Total 0 (delta 0), reused 0 (delta 0)
     * [new branch]      upstream -> upstream
    Branch 'upstream' set up to track remote branch 'upstream' from 'origin'.
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git remote set-url origin git@gitlab.lan:muxync/pelican-themes.git
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git remote -v
    origin	git@gitlab.lan:muxync/pelican-themes.git (fetch)
    origin	git@gitlab.lan:muxync/pelican-themes.git (push)
    upstream	https://github.com/getpelican/pelican-themes.git (fetch)
    upstream	https://github.com/getpelican/pelican-themes.git (push)
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git remote set-url --push upstream git@gitlab.lan:muxync/pelican-themes.git
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git remote -v
    origin	git@gitlab.lan:muxync/pelican-themes.git (fetch)
    origin	git@gitlab.lan:muxync/pelican-themes.git (push)
    upstream	https://github.com/getpelican/pelican-themes.git (fetch)
    upstream	git@gitlab.lan:muxync/pelican-themes.git (push)
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git pull
    Already up to date.
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git push
    Everything up-to-date
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$ git checkout master
    Switched to branch 'master'
    Your branch is up to date with 'origin/master'.
    ~/Workspace/pelican-blog/themes (upstream 65dd78d)$

Now when I want to test upstream code I can do the following to test locally:

    :::shell
    # From the themes directory
    git checkout upstream
    git pull

    # From the pelican-blog directory
    make publish
    make serve

If that looks good I can test on my "staging" LAN GitLab instance with:

    :::shell
    # From the themes directory
    git checkout master
    git rebase upstream
    git push

    # From the pelican-blog directory
    git submodule update --remote --merge
    git add themes
    git commit -m "synced upstream changes to the themems submodule"
    git push

Then I can create a new remote/branch named `downstream` with:

    :::shell
    # From the themes directory
    git remote add downstream git@gitlab.com:muxync/pelican-themes.git
    git checkout -b downstream

    # From the pelican-blog directory
    git remote add downstream git@gitlab.com:muxync/pelican-blog.git
    git checkout -b downstream

That I can push to my "production" public GitLab instance with:

    :::shell
    # From the themes directory
    git checkout downstream
    git rebase master
    git push --set-upstream downstream downstream

    # From the pelican-blog directory
    git checkout downstream
    git rebase master
    git push --set-upstream downstream downstream

# GitLab CI
My `content` repo doesn't have an `upstream` remote/branch since it's all my original content, however it still has a `downstream` remote/branch that I use as described above.  The other major difference is I have automated the process of updating the `content` submodule in my `pelican-blog` repo with the following [pelican-content `.gitlab-ci.yml`](https://gitlab.com/muxync/pelican-content/-/blob/master/.gitlab-ci.yml):

    :::yaml
    downstream:
      trigger:
        project: muxync/pelican-blog
        strategy: depend
      rules:
        - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

So when I push to the `CI_DEFAULT_BRANCH` (in my case `master`) branch of the `pelican-content` repo it will automatically "trigger" a pipeline in the `pelican-blog` repo which will build and deploy my static site.  For reference, you can view the `content` stage of the following [pelican-blog `.gitlab-ci.yml`](https://gitlab.com/muxync/pelican-blog/-/blob/master/.gitlab-ci.yml):

    :::yaml
    image: python:3-alpine

    stages:
      - content
      - test
      - deploy

    variables:
      GIT_SUBMODULE_STRATEGY: recursive
      #GIT_USER_EMAIL: <set it project variables>
      #GIT_USER_NAME: <set it project variables>
      #SSH_PRIVATE_KEY: <set it project variables>

    content:
      stage: content
      cache: {}
      before_script:
        - apk update && apk add git openssh-client
        - git config --global user.email "${GIT_USER_EMAIL}"
        - git config --global user.name "${GIT_USER_NAME}"
        - eval $(ssh-agent -s)
        - echo "${SSH_PRIVATE_KEY}" | tr -d '\r' | ssh-add -
        - mkdir -p ~/.ssh
        - chmod 700 ~/.ssh
        - ssh-keyscan ${CI_SERVER_HOST} >> ~/.ssh/known_hosts
        - chmod 644 ~/.ssh/known_hosts
      script:
        - git checkout -b auto-content-${CI_PIPELINE_IID}
        - GIT_SUBMOD_SHA_BEFORE=$(git submodule status | grep content | awk '{print $1}')
        - git submodule update --remote --merge
        - git add content
        - GIT_SUBMOD_SHA_AFTER=$(git submodule status | grep content | awk '{print $1}')
        - git commit -m "automated content submodule update from ${GIT_SUBMOD_SHA_BEFORE} to ${GIT_SUBMOD_SHA_AFTER}"
        - git remote set-url origin git@${CI_SERVER_HOST}:${CI_PROJECT_PATH}.git
        - git push --set-upstream origin auto-content-${CI_PIPELINE_IID} -o merge_request.create -o merge_request.target=${CI_DEFAULT_BRANCH} -o merge_request.merge_when_pipeline_succeeds -o merge_request.remove_source_branch -o merge_request.title="Automated content submodule update ${CI_PIPELINE_IID}"
      rules:
        - if: '$CI_PIPELINE_SOURCE == "pipeline"'

    .build:
      cache: {}
      script:
        - apk update && apk add make
        - pip install -r requirements.txt
        - sed -i "s#^SITEURL.*#SITEURL = '${CI_PAGES_URL}'#g" pelicanconf.py publishconf.py
        - sed -i "s#^SEARCH_URL.*#SEARCH_URL = '/${CI_PROJECT_NAME}/search.html'#g" pelicanconf.py publishconf.py
        - DEBUG=1 make publish
        - cp content/extra/* public/

    test:
      extends: .build
      stage: test
      rules:
        - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

    pages:
      extends: .build
      stage: deploy
      artifacts:
        paths:
          - public
      rules:
        - if: '$CI_PIPELINE_SOURCE == "pipeline"'
          when: never
        - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

In addition to the `content` stage I use a `.build` template which gets used in the `extends` section of the `test` and `deploy` stages.  Those correspond to the `test` and `pages` jobs respectively which are essentially the same except:

1. `test` jobs only run on on merge request events (i.e. when a MR gets created or updated).
2. `pages` jobs never run on `pipeline` (i.e. what is confusingly named `trigger` in the `pelican-content` repo's `.gitlab-ci.yml`, [see the documentation](https://docs.gitlab.com/ee/ci/yaml/#common-if-clauses-for-rules)).
3. `pages` jobs only run on the `CI_DEFAULT_BRANCH` (in my case `master`) branch.
4. `pages` jobs will upload `artifacts` that get deployed to the GitLab pages endpoint (the job **must** be named `pages` for it to be deployed to GitLab Pages).

I could have just forked the [example Pelican site for GitLab Pages](https://gitlab.com/pages/pelican) but I wanted to gain a better understanding of how pelican worked with GitLab and I'm quite pleased with how it came out.  Learning the hard way has gotten me an easy to maintain with a fair amount of automation in a short amount of time.  Now all I have to do is create a Markdown text file and run a few git commands to update my website.
