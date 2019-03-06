## PSN documentation contributions

The doc is served on <a href="https://sictiam.github.io/psn-doc/" target="_blank">https://sictiam.github.io/psn-doc/</a>

### Installation

The documentation website is processed by Jekyll and served thanks to Github Pages. Please follow the installation instructions [here](https://help.github.com/articles/using-jekyll-with-pages/).

When installing gems, you might prefer to install them locally thanks to:
```
bundle install --path vendor/bundle
```

### Launch Jekyll

To launch Jekyll and process assets:
```
bundle exec jekyll serve
```

### Using Docker

As an alternative to installing Jekyll locally, you can also use Docker and Compose. A `docker-compose.yml` is provided so all you have to do is:
```
docker-compose up
```
and Jekyll will run the site and watch for file changes to automatically rebuild it.

### Staying up-to-date

Whether you use Ruby Gems or Docker, it's important to stay up-to-date with GitHub's versions of Jekyll and its plugins, so make sure to regularly run:
```
bundle update github-pages
```
or, if you opted for Docker:
```
docker-compose pull
```

### Specifics

##### Processing markdown

It seems including markdown files from index.html with `{% include my-file.md %}` just does a raw paste without processing the markdown. That's why we use (see in index.html) this syntax:
```
{% capture inc0 %}{% include 0-welcome.md %}{% endcapture %}{{ inc0 | markdownify }}
```

See more about this issue [here](https://github.com/jekyll/jekyll/issues/1303).

##### Provide DOM class and id

Kramdown is used so that we can add class and id on created DOM elements. For example to generate a `<p class="focus">paragraph</p>`:
```
paragraph
{: .focus}
```

More info [here](http://kramdown.gettalong.org/quickref.html#block-attributes).

### Credits

The styling and behavior of the documentation is deeply inspired by [meteor docs](http://docs.meteor.com/). The CSS relies on [normalize](https://necolas.github.io/normalize.css/) and [typeplate](http://typeplate.com).

### TODO

- document refresh tokens (creation and revocation)

### Documentation conventions

Shorts lists items do not end with punctuation marks:
- first item
- second
- and third

Longer ones (those that contains several sentences) do:
- sentence. Another one;
- sentence;
- sentence. Another one.

### Discussion

For the moment the documentation relies on the following choices, that needs to be discussed:

- no description of subscription callback for the moment, since it remains to be implemented
- replaced Kernel by Ozwillo when we're talking about APIs
