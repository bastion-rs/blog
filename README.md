# Bastion-rs Blog

This is the code that powers [the bastion-rs blog]. It is running
on [GitHub pages].

[the bastion-rs blog]: https://bastion-rs/blog
[GitHub pages]: https://pages.github.com/

This project has been forked from [the blog boilerplate](https://github.com/rust-community/rust-lang-blog-boilerplate), which has been forked from the awesome [github repo](https://github.com/rust-lang/blog.rust-lang.org) of the [RustLang blog](https://blog.rust-lang.org/)

## Contributing

Feedback, issues, and pull requests are welcome and appreciated if you find any typo or want to add contents.

## Running locally

Running locally is just a matter of installing jekyll and serving the site:

```
bundle install

# GitHub OAuth token avoids 403 rate limiting errors while generated crate data
export GITHUB_OAUTH_TOKEN=<YOUR_GITHUB_TOKEN>

bundle exec jekyll serve
```

The site should be running on [localhost:4000](http://localhost:4000)

## License

The Rust Programming Language Blog is primarily distributed under the terms of
CC-BY 4.0. 
So is this project.

See LICENSE for details.

## Code of conduct

Any project I create and I take part of respects the [Rust Code of Conduct](CODE_OF_CONDUCT.md).

If there is any issue send me a message and I will make sure the code is respected.

Happy blogging and happy coding ! :)