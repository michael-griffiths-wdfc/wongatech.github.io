# wongatech.github.io

## Posting content
The _posts folder is where blog posts live. These files are generally Markdown or HTML.
Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

At a minimum all posts should start with:

```
---
layout: post
title: Replace with the title of your post
---
```

For more information checkout the [Jekyll Documentation](http://jekyllrb.com/docs/posts/)

## Running the site locally
You can find instructions on how to run the repo locally ath the following link:
[Setting up your site locally](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)

The basic requirements:
1. Make sure you have Ruby 2.1.0 or higher installed
2. Install bundler `gem install bundler`
3. Run `bundle install` in the root of the repo

Once installed simply run
```
bundle exec jekyll serve
```
And browse to the provided URL