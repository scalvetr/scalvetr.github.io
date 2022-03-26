# scalvetr Blog

## Run locally

### Prerequisites

* Ruby
* Ruby Gems
* C++

See: https://jekyllrb.com/docs/installation/


### Install deps
Then install the following gems:

```shell
gem install jekyll -v 3.9.0
gem install github-pages -v 223
gem uninstall rouge -v 3.28.0

# with ruby 3.0
gem install webrick
```

### Build

```shell
jekyll build
```

### Run

```shell
jekyll serve --drafts
```