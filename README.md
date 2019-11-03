<!-- PROJECT SHIELDS -->
[![Build Status](https://github.com/marcodenisi/marcodenisi-dev/workflows/Deploy%20new%20version/badge.svg)](https://github.com/marcodenisi/marcodenisi-dev/actions)

# Personal Website

[Personal website](www.marcodenisi.dev) and blog written using [Hugo](https://gohugo.io/). 
The template used is the [Cocoa](https://themes.gohugo.io/cocoa/).
Continuous delivery provided by [Github Actions](https://github.com/actions).

## Installation

To run locally, open the terminal, navigate to the root folder and type `hugo server`. 
You can now access the blog through the browser at `localhost:1313`.

## Add Content

When adding new content, please open a new feature branch, do not work on master branch.

### Add New Blog Post
Just run

`hugo new blog/your-new-post.md`

This will create a new post in the default language directory. To add the post to other languages, copy/paste the file and add the `.en` suffix before the file extension.

## Deploy

Once you're done with the changes or adding a new blog post, you can deploy to GitHub Pages the new site version. 

To do so, open a new PR and merge the previously created feature branch into master. This will fire a new Github Actions Workflow that will build the site and push it to the blog repository. See `.github/workflows/main.yml` for details.