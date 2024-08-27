---
title: "Gitlab runner: Invalid US-ASCII character on line 51"
date: "2019-09-14"
tags:
  - jekyll
  - cdci
  - gitlab
  - runner
categories:
  - blog
  - gitlab
author: jtorrex
---

# Description:

Solving problems with charset on GitLab Runners.

# Building static websites on Gitlab

This blog/static page, it's generated using Jekyll <https://jekyllrb.com/> and a fantastic template provided by Kopplin <https://github.com/sergiokopplin/indigo>.

For simplify the post creations and changes on the website, I integrated the repository hosted on GitLab to build and deploy the static files of the website to the destination webserver serving the page.

This part it's acomplished building the web on a Gitlab runner when I commit and push changes to the repo, the runner contains the necessary Jekyll software and the Ruby Gems to build it, and finally, I push the static site pages to the final destination.

## Problems with chars

When I started, all the initial phases of the pipeline running in the Gitlab runner seems to work fine except the job when I build the statics, in this part, I found this unexpected error:

```
bundle exec jekyll build
Configuration file: /builds/torrex/t0rrex.net/_config.yml
            Source: /builds/torrex/t0rrex.net
       Destination: /builds/torrex/t0rrex.net/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
  Liquid Exception: Liquid error (line 27): Invalid US-ASCII character "\xE2" on line 51 in /_layouts/default.html
jekyll 3.7.4 | Error:  Liquid error (line 27): Invalid US-ASCII character "\xE2" on line 51
```

Digging in some websites/posts, someone suggest to apply this workarond, because the Gitlab runner doesn't had configured the correct locales to manage some UTF-8 characters.

## Workaround to enable UTF-8

The workaround that worked for me it's basically to configure the correct charset and locales before the task when the website is build. As an example I install and configure the required packages, and after, configure the runner environment:

```
build_site:
  stage: build
  only:
    - master
  script:
    - apt-get update >/dev/null
    - apt-get install -y locales rsync >/dev/null
    - echo "en_US UTF-8" > /etc/locale.gen
    - locale-gen en_US.UTF-8
    - export LANG=en_US.UTF-8
    - export LANGUAGE=en_US:en
    - export LC_ALL=en_US.UTF-8
```

After this section, we can add the rest of the script code to build the website with Jekyll, and, in the next time that the runner tries to build again the website, the error doesn't appear anymore, so we can consider that the problem was solved.

I hope that this workaround was useful for you in the case that you find similar errors working with Gitlab Runners, Jekyll pages, etc.

For any correction/suggerence, please write me directly to my public mail.

Regards!
