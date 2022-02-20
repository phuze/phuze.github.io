---
layout: post
title: "How to Use Unsupported Plugins with GitHub Pages"
tags: ["Jekyll", "Jekyll 4.2.1", "GitHub Pages"]
comments: true
toc: true
---

A benefit of Jekyll, is it's seamless support for automated deployment, for free, to GitHub Pages. However, the more time you invest in your Jekyll-powered website, you're more likely to encounter unsupported plugins:

```
github-pages 223 | Error:  Liquid syntax error (line 28): Unknown tag
```

The reason — GitHub Pages only supports a specific list of [dependencies and plugins](https://pages.github.com/versions/){:target="_blank"}.

The way to make use of unsupported plugins, is to build your website locally, instead of relying on GitHub's automated build and deploy process for Jekyll.

## Step 1: Update your Gemfile

Update your `Gemfile` and use the core `jekyll` gem, rather than the `github-pages` gem.

{% highlight ruby %}
source 'https://rubygems.org'

# Jekyll
gem 'jekyll'
{% endhighlight %}

## Step 2: Create an action workflow

In your project, create a new workflow file `.github/workflows/deploy.yml`. This workflow ends up creating a new `gh-pages` branch for deployment. Find and change this if you prefer a different branch name:

{% highlight yaml %}
name: build and deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v2
    - name: Build the site in the jekyll/builder container
      run: |
        docker run \
        -v {% raw %}${{ github.workspace }}{% endraw %}:/srv/jekyll \
        -v {% raw %}${{ github.workspace }}{% endraw %}/_site:/srv/jekyll/_site \
        jekyll/builder:latest /bin/bash -c "chmod 777 /srv/jekyll" \
        jekyll/builder:latest /bin/bash -c "chown jekyll:jekyll -R /srv/jekyll" \
        jekyll/builder:latest /bin/bash -c "chown jekyll:jekyll -R /usr/gem" \
        jekyll/builder:latest /bin/bash -c "jekyll build --future"
    - name: Push the site to the gh-pages branch
      if: {% raw %}${{ github.event_name == 'push' }}{% endraw %}
      run: |
        sudo chown $( whoami ):$( whoami ) ${{ github.workspace }}/_site
        cd ${{ github.workspace }}/_site
        git init -b gh-pages
        git config user.name {% raw %}${{ github.actor }}{% endraw %}
        git config user.email {% raw %}${{ github.actor }}{% endraw %}@users.noreply.github.com
        git remote add origin https://x-access-token:{% raw %}${{ github.token }}{% endraw %}@github.com/${{ github.repository }}.git
        git add .
        git commit -m "Deploy site built from commit {% raw %}${{ github.sha }}{% endraw %}"
        git push -f -u origin gh-pages
{% endhighlight %}

### Understanding the Workflow

Let's take a closer look at what's happening in this workflow:

- `name` — The name of our workflow.
- `on` — Defines when the workflow will be triggered. in this example, thats either on a push or pull-request to the `main` branch
- `jobs` — A set of steps to be executed when our workflow is triggered. You can learn more about jobs from the [GitHub Docs](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#jobs){:target="_blank"}.
- `steps` — These are the individual steps which make up our job. 

Our job begins with a reusable action, [actions/checkout@v2](https://github.com/marketplace/actions/checkout){:target="_blank"}, which clones our project to the workflow's virtual environment:

{% highlight yaml %}
- uses: actions/checkout@v2
{% endhighlight %}

Following that, we're going to use the [jekyll/builder](https://hub.docker.com/r/jekyll/builder){:target="_blank"} docker image to build our project files:

{% highlight yaml %}
- name: Build the site in the jekyll/builder container
    run: |
    docker run \
    -v {% raw %}${{ github.workspace }}{% endraw %}:/srv/jekyll \
    -v {% raw %}${{ github.workspace }}{% endraw %}/_site:/srv/jekyll/_site \
    jekyll/builder:latest /bin/bash -c "chmod 777 /srv/jekyll" \
    jekyll/builder:latest /bin/bash -c "chown jekyll:jekyll -R /srv/jekyll" \
    jekyll/builder:latest /bin/bash -c "chown jekyll:jekyll -R /usr/gem" \
    jekyll/builder:latest /bin/bash -c "jekyll build --future"
{% endhighlight %}

It is necessary to grant the user `jekyll` ownership and permissions to both the `/srv/jekyll` and `/usr/gem` directories, otherwise you'll encounter build errors. This is why you see the `chmod` and `chown` commands.

> There was an error while trying to write to `/srv/jekyll/Gemfile.lock`. It is
> likely that you need to grant write permissions for that path.
> Error: Process completed with exit code 23.

The last step of our job, is to deploy our Jekyll website to GitHub pages. We accomplish this by creating a dedicated branch containing everything you'd find within the build directory, `/_site`. We'll then update our repository settings to serve our Pages site from that branch.

{% highlight yaml %}
- name: Push the site to the gh-pages branch
    if: {% raw %}${{ github.event_name == 'push' }}{% endraw %}
    run: |
    sudo chown $( whoami ):$( whoami ) ${{ github.workspace }}/_site
    cd ${{ github.workspace }}/_site
    git init -b gh-pages
    git config user.name {% raw %}${{ github.actor }}{% endraw %}
    git config user.email {% raw %}${{ github.actor }}{% endraw %}@users.noreply.github.com
    git remote add origin https://x-access-token:{% raw %}${{ github.token }}{% endraw %}@github.com/${{ github.repository }}.git
    git add .
    git commit -m "Deploy site built from commit {% raw %}${{ github.sha }}{% endraw %}"
    git push -f -u origin gh-pages
{% endhighlight %}

First we make sure our workspace environment has ownership over our `/_site` directory, and enter it:

{% highlight yaml %}
sudo chown $( whoami ):$( whoami ) ${{ github.workspace }}/_site
cd ${{ github.workspace }}/_site
{% endhighlight %}

Next we initialize a new repository under the `/_site` directory, along with a new `gh-pages` branch we'll use as our deployment branch:

{% highlight yaml %}
git init -b gh-pages
git config user.name {% raw %}${{ github.actor }}{% endraw %}
git config user.email {% raw %}${{ github.actor }}{% endraw %}@users.noreply.github.com
git remote add origin https://x-access-token:{% raw %}${{ github.token }}{% endraw %}@github.com/${{ github.repository }}.git
{% endhighlight %}

Lastly, we commit our built site to our `gh-pages` branch:

{% highlight yaml %}
git add .
git commit -m "Deploy site built from commit {% raw %}${{ github.sha }}{% endraw %}"
git push -f -u origin gh-pages
{% endhighlight %}

## Step 3: Commit the workflow

Go ahead and commit your workflow file. This also serves to trigger our workflow action, necessary for Step 4.

## Step 4: Update repository settings

You should now have a built project and a new branch named `gh-pages`. Visit your repository on GitHub, choose `Settings > Pages`, then change your `Source` to your new deployment branch:

![Image with caption](assets/pages-deploy-branch.png)
_Found at https://github.com/{username}/{repository}/settings/pages_

After a short wait, your website should now be available again, along with any features driven by those unsupported plugins.