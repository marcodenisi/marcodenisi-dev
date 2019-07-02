# Personal Website

[Personal website](www.marcodenisi.dev) and blog written using [Hugo](https://gohugo.io/). 
The template used is the [Cocoa](https://themes.gohugo.io/cocoa/).

## Installation

To run locally, open the terminal, navigate to the root folder and type `hugo server`. 
You can now access the blog through the browser at `localhost:1313`.

## Add Content

### Add New Blog Post
Just run

`hugo new blog/your-new-post.md`

This will create a new post in the default language directory. To add the post to other languages, copy/paste the file and add the `.en` suffix before the file extension.

## Deploy

Once you're done with the changes or adding a new blog post, you can deploy to GitHub Pages the new site version. 

To do so, just open your terminal, navigate to the root folder and type `./deploy "Your comment"`. After a couple of minutes you'll see changes at https://marcodenisi.github.io.
