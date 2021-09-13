---
title: "How to create a tech blog"
date: 2021-09-12T21:18:57+04:00
lastmod: 2021-09-13T06:18:06+04:00
slug: "how-to-create-a-tech-blog"
draft: false
categories:
- Blog management
tags:
- hugo
- blog
- wrangler
- cloudflare
---

This article will show step by step how to create a blog just like this one.

## Assumptions

* You can work with terminal (bash, zsh, fish, etc.).
* You can work with [git](https://git-scm.com/) and familiar with [git flow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow).
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
you can check its getting started article [here](https://gohugo.io/getting-started/quick-start/).

I am opinionated that git flow is very good for such kind of development as blog,
so I am hosting my blog on github and using github actions to deploy it.
When it comes to serving, I think cloudflare workers are the best match for start.
You can use quota up to 100,000 requests per day, which is more than enough for beginning.

## Action

### Create site

First you need [git](https://github.com/git-guides/install-git) and [hugo](https://gohugo.io/getting-started/installing/) installed on your computer.

Our site's name will be blogg, just substitute it with whatever you're going to call your blog.

Open your terminal and run:

```bash
$ hugo new site blogg
$ git init
```
This will create a new hugo site and initialize git repository in it.

Then you have to choose a theme to use in your blog. You can find list of themes [here](https://themes.gohugo.io/).

Pick one, click on it and you'll see a button `Download`, click on it.
![Download hugo tania](img/hugo-tania-download.png)

Click on green `Code` button, choose `HTTPS` and click copy button
![Clone hugo tania](img/hugo-tania-clone.png)

Go to your project's folder
```bash
$ cd blogg
```

Add your theme as git submodule and to config.toml
```bash
$ git submodule add git@github.com:WingLim/hugo-tania.git themes/hugo-tania
$ echo theme = \"hugo-tania\" >> config.toml
```

Now you should be able to preview you site.
```bash
$ hugo serve
```
You will see output like
```
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
```
Open [the address](http://localhost:1313/) in your browser and you should see your landing web page.
![Empty hugo site](img/empty-hugo-site.png)

### Push to git

Add a file named `.gitignore` with following contents to your project folder
```gitignore
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

Now you should push your repository to github. 
If you are not sure how to do it, check the guide [here](https://docs.github.com/en/github/importing-your-projects-to-github/importing-source-code-to-github/adding-an-existing-project-to-github-using-the-command-line#adding-a-project-to-github-without-github-cli).
You should skip `git init -b main` command that is provided in the github's tutorial, because we already executed it.

At this stage you should have your hugo project in github.

### Deploying from local machine

You will need wrangler installed on your computer to make an initial deploy and check if everything works correctly.
Check tutorial [here](https://developers.cloudflare.com/workers/cli-wrangler/install-update).

Cloudflare worker requires three secrets
* CF_ACCOUNT_ID
* CF_ZONE_ID
* CF_API_TOKEN

You can get first two of them from `Overview` of your domain in cloudflare, scroll to the very bottom of the page.

![Copy cloudflare zone id and account id](img/cloudflare-account-and-zone-ids-copy.png)

For api token go to [this](https://dash.cloudflare.com/profile/api-tokens) page.
Press `Create Token` button, in a new page select `Edit Cloudflare Workers` as a template.
Copy your api token and save it in a **safe place**.

Adding these secrets to github is nessecary for pipeline, so we'll do it right away.
Go to your github repository page, navigate to `Settings` and find `Secrets` tab.
Add your secrets to repository by pressing `New repository secret`.

![Add secrets to github repository](img/github-cloudflare-secrets-new.png)

Return to your terminal. Now you need to authorize your wrangler cli.
Run
```bash
$ wrangler login
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
$ hugo --minify --gc
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
$ wrangler build # <-- build worker
$ wrangler publish -c wrangler.toml --env production # <-- publish your site to worker
```

At this stage you should be able to view the site on your domain.
