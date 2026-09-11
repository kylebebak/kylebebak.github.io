---
layout: post
comments: true
title: "Implementing Tags and Categories with Jekyll and GitHub Pages"
categories: code jekyll
tags: [github, github-pages, jekyll, git, markdown]
---

## Blogging Like a Hacker

Using a CMS is overkill if you're not managing __lots__ of content, which is why I decided to use Jekyll to build my blog. Jekyll has been around for a few years now. It runs as a process that watches your source files (e.g. __SASS__ and __Markdown__) for changes and builds them into corresponding __HTML__ and __CSS__ site files. These can be uploaded to and served by any server, such as the ones you get for free with [GitHub Pages](https://pages.github.com/), and rendered by any browser.

Jekyll is lightweight but very flexible and powerful. It integrates a nice template language called [Liquid](https://shopify.github.io/liquid/). Jekyll exposes all kinds of site variables and objects to Liquid and allows you to reference them wherever you want, which makes it easy to write composable components and generally reuse code.

For example, any HTML snippet in your `_includes` directory can be pulled into your content with Liquid's {% raw %}`{% include %}`{% endraw %} tag, and the content of the snippet itself can be set dynamically on "instantiation". Here's a snippet I use all over the site for inline icons. The icon, the text associated with it and the CSS classes applied to the text are parameters.

{% raw %}
~~~html
<span class="icon">{% include {{ include.icon }} %}</span><span class="icon-text {{ include.text-classes }}"> {{ include.text }}</span>
~~~
{% endraw %}

All pages can declare which __layout__ they "inherit" from, which makes it easy to standardize the look of your site and separate content from presentation.

Finally, Jekyll allows you to use or write your own [plugins](http://jekyllrb.com/docs/plugins/) to  do all kinds of things, like automatically generate content for your site whenever it's rebuilt, create custom Liquid tags and filters, etc.


## Classifying Content

I wanted to use Jekyll plugins for building __category__ and __tag__ classification systems for my posts, but for obvious reasons, GitHub Pages doesn't let you run any old 
code on their servers. Here's the [list of Ruby gems](https://pages.github.com/versions/) that they have installed, some of which are Jekyll plugins. Categories and tags are conspicuously absent.

Categories organize content into a tree. Their relationship with posts is __one-to-many__. The relationship with tags to posts is __many-to-many__, and the classification they provide is less rigid, connecting posts that would otherwise be siloed into different categories.

Given that I was going to have to build the categories and tags pages myself, and I'm not too good with Ruby, I decided to do it [with Python](https://github.com/kylebebak/kylebebak.github.io/tree/master/_build). I wrote one program for categories and another for tags. Both import methods from a `build.py` module that parses all of the posts and has methods for returning a dictionary of `{ category : [posts] }` or `{ tag : [posts] }`.

I use a `pre-commit` hook to ensure that these pages get built and included in every commit, which means they're always up to date when changes are pushed to the site.

~~~sh
#!/bin/bash -e

python _build/categories.py --posts_dir="_posts" --categories_file="_includes/categories.md"
git add _includes/categories.md

rm tags/*
python _build/tags.py --posts_dir="_posts" --tags_dir="tags" --tags_file="_includes/tags.html"
git add -u :/
git add _includes/tags.html tags/*
~~~

The `-e` flag in the shebang line ensures that the hook, and therefore the commit, fail if any of the commands in the hook exit with a non-zero exit status. The individual tag pages (not the main tags index) are built using a [system devised by Stephan Groß](http://www.minddust.com/post/tags-and-categories-on-github-pages/).

---

If you're thinking of blogging, or you already do, and you want more control and an excuse to improve your programming chops, go with a static site generator. You can edit your content in your text editor, all of it goes in version control, and your pages will load really really fast.
