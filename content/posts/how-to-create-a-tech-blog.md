---
title: "How to create a tech blog"
date: 2021-09-12T21:18:57+04:00
lastmod: 2021-09-13T06:18:06+04:00
author: "Murr Kyuri"
slug: "how-to-create-a-tech-blog"
draft: false
categories:
- Blog management
tags:
- hugo
- blog
- wrangler
- cloudflare
- github
description: "Complete guide how to create static blog site using hugo, github, wrangler, and cloudflare workers"
---

This article will show step by step how to create a blog just like this one.

## Assumptions

* You can work with terminal (bash, zsh, fish, etc.).
* You can work with [git](https://git-scm.com/) and familiar with [trunk git flow](https://www.atlassian.com/continuous-delivery/continuous-integration/trunk-based-development).
* You are familiar with [cloudflare](https://www.cloudflare.com/), ideally worked with it earlier and your domain is parked on cloudflare.
* You are familiar with [hugo](https://gohugo.io/)

## Tech stack

I think markdown is the best format for writing any kind of documentation or blog,
because of its simplicity.

My blog does not contain anything except text, some style files, and images
so I decided to use something simple and statically-compiled.
I chose hugo framework, because it is super fast, easy to use, and configure.
Hugo is a cli tool that compiles `.md` files into static files that you can serve later using nginx or some other technology of your choice.
It is very easy to start working with hugo,
you can check its [getting started article](https://gohugo.io/getting-started/quick-start/).

I am opinionated that trunk flow is very good for such kind of development as blog,
so I am hosting my blog on github and using github actions to deploy it.
When it comes to serving, I think cloudflare workers are the best match for start.
You can use quota up to 100,000 requests per day, which is more than enough for beginning.

## Action

### Create site

First you need [git](https://github.com/git-guides/install-git) and [hugo](https://gohugo.io/getting-started/installing/) installed on your computer.

Our site's name will be blogg, just substitute it with whatever you're going to call your blog.

Open your terminal and run:

```bash
hugo new site blogg
git init
```
This will create a new hugo site and initialize git repository in it.

Then you have to choose a theme to use in your blog. There you can find [list of themes](https://themes.gohugo.io/).

Pick one, click on it and you'll see a button `Download`, click on it.
![Download hugo tania](img/hugo-tania-download.png)

Click on green `Code` button, choose `HTTPS` and click copy button
![Clone hugo tania](img/hugo-tania-clone.png)

Go to your project's folder
```bash
cd blogg
```

Add your theme as git submodule and to config.toml
```bash
git submodule add git@github.com:WingLim/hugo-tania.git themes/hugo-tania
```

Then add you `config.toml` file
```toml
baseURL = "https://blogg.domain/"
languageCode = "en-us"
title = "Blogg blog"
theme= "hugo-tania"
titleEmoji = "🐱"

[markup]
[markup.highlight]
  noClasses = false
  lineNos = true
```

Now you should be able to preview you site.
```bash
hugo serve
```
You will see output like
```
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
```
Open [the address](http://localhost:1313/) in your browser and you should see your landing web page.
![Empty hugo site](img/empty-hugo-site.png)

### Push to git

Add a file named `.gitignore` with following contents to your project folder
```md
# Created by https://www.toptal.com/developers/gitignore/api/hugo
# Edit at https://www.toptal.com/developers/gitignore?templates=hugo

### Hugo ###
# Generated files by hugo
/public/
/resources/_gen/
hugo_stats.json

# Executable may be added to repository
hugo.exe
hugo.darwin
hugo.linux

# End of https://www.toptal.com/developers/gitignore/api/hugo

dist/
```

Now you should [push your repository to github](https://docs.github.com/en/github/importing-your-projects-to-github/importing-source-code-to-github/adding-an-existing-project-to-github-using-the-command-line#adding-a-project-to-github-without-github-cli).
You should skip `git init -b main` command that is provided in the github's tutorial, because we already executed it.

At this stage you should have your hugo project in github.

### Deploying from local machine

You will need wrangler installed on your computer to make an initial deploy and check if everything works correctly.
Check this tutorial to [learn how to install wrangler](https://developers.cloudflare.com/workers/cli-wrangler/install-update).

Cloudflare worker requires three secrets
* CF_ACCOUNT_ID
* CF_ZONE_ID
* CF_API_TOKEN

You can get first two of them from `Overview` of your domain in cloudflare, scroll to the very bottom of the page.

![Copy cloudflare zone id and account id](img/cloudflare-account-and-zone-ids-copy.png)

For api token go to [api tokens page](https://dash.cloudflare.com/profile/api-tokens).
Press `Create Token` button, in a new page select `Edit Cloudflare Workers` as a template.
Copy your api token and save it in a **safe place**.

Adding these secrets to github is nessecary for pipeline, so we'll do it right away.
Go to your github repository page, navigate to `Settings` and find `Secrets` tab.
Add your secrets to repository by pressing `New repository secret`.

![Add secrets to github repository](img/github-cloudflare-secrets-new.png)

Return to your terminal. Now you need to authorize your wrangler cli.
Run
```bash
wrangler login
```
This will open a browser window, where you should confirm that you want to authorize the application.


Then go to the project folder and create a file named `wrangler.toml` with following contents.
```toml
type = 'webpack'

[site]
bucket = "./public"
entry-point = "workers-site"

[env.staging]
name = "blogg-domain-dev"
usage_model = ''
workers_dev = true

[env.production]
name = "blogg-domain"
workers_dev = false
route = 'blogg.domain/*'
```

Substitute `name` keys in `env.staging` and `env.production`; also substitute `route` to your site's one.

Run
```bash
hugo --minify --gc
```

This will create a static build of the site.

Now you need to export cloudflare variables.

Run
```bash
export CF_ACCOUNT_ID=value you got from cloudflare
export CF_ZONE_ID=value you got from cloudflare
```

Then you need to run
```bash
wrangler build # <-- build worker
wrangler publish -c wrangler.toml --env production # <-- publish your site to worker
```

At this stage you should be able to view the site on your domain.

### Configure pipeline

As I mentioned earlier we will use github actions for CI/CD.

Basically you've already done most of the work, now you just need to add 2 files to your repository.


Deploy to staging when pull request is created against master branch.

`.github/workflows/deploy-staging.yml`

```yaml
name: Build & Publish to staging

on:
  pull_request:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      with:
        submodules: true

    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2
      with:
        hugo-version: 'latest'
        extended: true

    - name: Build site
      run: hugo --minify --gc

    - name: Publish to Workers Sites
      uses: cloudflare/wrangler-action@1.3.0
      with:
        apiToken: ${{ secrets.CF_API_TOKEN }}
        environment: 'staging'
      env:
        CF_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
        CF_ZONE_ID: ${{ secrets.CF_ZONE_ID }}
```

Release to main site, when changes are merged to master.

`.github/workflows/deploy.yml`

```yaml
name: Build & Publish

on:
  push:
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
      with:
        submodules: true

    - name: Setup Hugo
      uses: peaceiris/actions-hugo@v2
      with:
        hugo-version: 'latest'
        extended: true

    - name: Build site
      run: hugo --minify --gc

    - name: Publish to Workers Sites
      uses: cloudflare/wrangler-action@1.3.0
      with:
        apiToken: ${{ secrets.CF_API_TOKEN }}
        environment: 'production'
      env:
        CF_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
        CF_ZONE_ID: ${{ secrets.CF_ZONE_ID }}
```

## Conclusion

This article is a complete guide on how to create a blog using hugo 
and cloudflare workers with github actions as CI/CD
