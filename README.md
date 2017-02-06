# Posts

<http://kylebebak.github.io/>

Code and rants about other stuff, like music and footy. The site itself is generated by __Jekyll__ and hosted by GitHub, using the wonderful and free __GitHub Pages__. The categories and tags pages are generated by Python scripts in [_build](_build).


## Installation
Via `bundle` and `pip`. From the root directory, run `bundle`, and `pip install -r requirements.txt`.


## Continuous Integration
Done via Git hooks. These live in [_hooks](_hooks), and are deployed to `.git/hooks`, like this:

~~~sh
cd .git/hooks && ln -s -f ../../_hooks/* ./
~~~

Before commits, `_includes/categories.md` and `_includes/tags.html` are regenerated and added to the commit.

To check links, I run [LinkChecker](https://github.com/wummel/linkchecker/) against the site, using:

~~~sh
# local
linkchecker --no-warnings http://127.0.0.1:4000/
# production
linkchecker --no-warnings http://kylebebak.github.io/
~~~

I don't use `LinkChecker` in Git hooks because it's sort of slow, and also the scripts in [_build](_build) are written for Python 3, while `LinkChecker` is written for Python 2. The relevant field in LinkChecker's output is `Parent URL`, which points to files containing broken links.


## Sublime Text Project Settings
~~~json
{
  "folders":
  [
    {
      "path": "{/PATH/TO/PROJECT}",
      "folder_exclude_patterns":
      [
        "_site"
      ],
      "file_exclude_patterns":
      [
        "tags/*"
      ]
    }
  ]
}
~~~


## SEO
Meta tags in posts generated by [jekyll-seo-tag](https://github.com/jekyll/jekyll-seo-tag), `sitemap.xml` generated by [jekyll-sitemap](https://github.com/jekyll/jekyll-sitemap). A line in `robots.txt` points to `sitemap.xml`.


## Click and ASCII
Read [this](http://click.pocoo.org/5/python3/#python-3-surrogate-handling) to see why `click` has problems with encoding in Python 3. One fix is to export the locale before invoking `click`, like this:

~~~sh
export LANG=en_US.utf-8
export LC_ALL=en_US.utf-8
~~~


## Custom CSS and JS
I use [AnchorJS](https://github.com/bryanbraun/anchorjs) for automatic deep linking. The minified JS file is sourced in `/_includes/head.html`, and the function is executed before the closing `</body>` tag in `/_layouts/default.html`.


## License
Everything in this repo is licensed under the [MIT License](https://opensource.org/licenses/MIT).
