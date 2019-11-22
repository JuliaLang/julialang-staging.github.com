This is the GitHub repository for the Julia programming language project's main website, [julialang.org](http://julialang.org/). The repository for the source code of the language itself can be found at [github.com/JuliaLang/julia](https://github.com/JuliaLang/julia).

The Julia website is generated using GitHub pages and Jekyll, as [explained here](https://help.github.com/articles/using-jekyll-with-pages).

## Running using Docker

After installing [`docker`](http://docker.com/), run `make run` to build and run the website within the a container built from the [`jekyll/jekyll` image](https://hub.docker.com/r/jekyll/jekyll/) that contains all the necessary prerequisites.  Modifying files will cause the website to rebuild in real time.

## Installing locally

In short, be sure you have ruby installed, and then run these commands

    gem install bundler

to install bundler, run

    bundle install --path vendor --binstubs

from the root of this repository to fetch all dependencies and install them locally.
You only have to do this once.

Now you can run

    ./bin/jekyll serve --future

in the root of this repository to serve it, usually on [http://localhost:4000](http://localhost:4000).
Or with other options, such as internet-visible on port 4444 with all drafts:

    ./bin/jekyll serve --incremental -H 0.0.0.0 -P 4444 -l --future --unpublished --drafts

You can then edit the sources and Jekyll will rebuild changed files. To see
changes, reload the page you edited.
