---
title: "Github Actions: a concrete example with Hugo and Latex"
date: 2019-11-05T10:26:59+01:00
draft: false
categories: ["Development"]
tags: ["cd", "github-actions", "github", "hugo"]
---

Months ago I decided to write my curriculum using Latex and to use Github to version it ([here](https://github.com/marcodenisi/cv) the repository). I said to myself that it would be beautiful to automatically create the latest cv version as a pdf file whenever I update the source files, maybe as a Github *release*.  

I had the same desire when I started thinking about this blog: wouldn't be amazing if at each new content, a new blog version was deployed automatically?

Well, that's the story of this journey: from all manual to automatic flow using Github Actions.

# Github Actions
Github Actions it's a new Github *feature* allowing users to create CI/CD workflows within the repository.

A **workflow** is defined in a *yaml* file and it's nothing but a list of **jobs**. Each job is then composed by **steps**. Lastly, each step is a set of **actions**. And it's the action the Github atomic unit: each of these performs an operation (e.g. it checkouts the repository, it creates a release, ...). 

In a single repository, many workflows can exist and they should be defined in the `.github/workflows` folder.

Each workflow is triggered by an event. For example, we could have a workflow triggered whenever a push is done in a branch:

```yaml
name: Deploy new version    # workflow name
on: 
  push:
    branches:
      - master
```
But what happens under the hood? Github has a pool of virtual machines (hosted on Azure). In each of these VMs are installed some special services called *runners*. When a workflow instance is created, jobs contained are executed by these runners.

[Here](https://help.github.com/en/github/automating-your-workflow-with-github-actions) the documentation.

# CV e Latex
How to compile a Latex document? [Here](https://github.com/marcodenisi/cv/blob/master/.github/workflows/main.yml) the file defining the workflow.

```yaml
name: Build CV
on:
  push:
    tags:
      - 'v*.*.*'
jobs:
  release_cv:
    runs-on: ubuntu-latest
    steps:
    ...
  
  push_to_blog:
    needs: release_cv
    runs-on: ubuntu-latest
    steps:
    ...
```
This is an excerpt showing the yaml structure. Here we can find the workflow name (Build CV) and the triggering event (a push of a tag). Two jobs are defined as well:

- `release_cv`: it compiles `.tex` files and it creates a new release on Github
- `push_to_blog`: pushes the release to the blog repository

Actions composing these jobs are pretty straight forward. The interesting part is about the interaction between jobs. I said before how jobs are executed by (probably different) runners in (probably different) virtual machines. So, potentially, they do not share any kind of data. In this context **artifacts** come into play. Artifacts are nothing but objects that are saved and retrieved at runtime by different jobs.

In fact, the last action of the first job is the artifact upload:
```
    - name: Upload to virtual env
      uses: actions/upload-artifact@v1
      with:
        name: cv
        path: cv_MarcoDenisi.pdf
```

and the first action of the second job is the download of the same artifact:
```
    - name: Download from virtual env
      uses: actions/download-artifact@v1
      with:
        name: cv
```

# Blog e Hugo

The blog compilation and upload is quite homemade, [here](https://github.com/marcodenisi/marcodenisi-dev/blob/master/.github/workflows/main.yml) the complete version:
```yaml
name: Deploy new version
on: 
  push:
    branches:
      - master
      
jobs:
  deploy_blog:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout master
      uses: actions/checkout@v1
    - name: Build blog and deploy
      run:  |
        ... bash script
``` 
As you can see, each action can also execute simple bash scripts (omitted for the sake of brevity). Not very elegant, but it works!

# Putting all together

I accomplished multiple goals:

- dismiss an old (and ugly) solution composed by Travis script and bash scripts
- learn a new tool
- keep the CV updated
- ... and so keep the blog updated as well

Currently Github Actions is free for all repositories, both public and private (the latter with some limitations). I think it's really worth start looking into it to have an integrated solution all in one place.