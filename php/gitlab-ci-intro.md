# Introduction to Gitlab CI for PHP developers

As a developer, you've probably at least heard something about [CI - Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration). And if you haven't - you better fix it ASAP, because that's something awesome to have on your skill list and can get extremely helpful in your everyday work. This post will focus on CI for PHP devs, and specifically, on CI implementation from [Gitlab](https://docs.gitlab.com/ee/ci/README.html). I will suppose you know the basics of [Git](https://git-scm.com/), [PHP](https://php.net/) and [PHPUnit](https://phpunit.de/).

Adding something to your workflow must serve a purpose. In this case the purpose is automation of routine tasks and better quality control. Even a basic PHP project IMO needs the following:
* [linter](https://en.wikipedia.org/wiki/Lint_(software)) checks (cannot merge changes that are invalid on the syntax level)
* Code style checks
* Unit and integration tests

All of those can be run eventually and manually, of course. But I prefer an automated CI approach even in my personal projects because it:
* leads to a higher level of discipline, and you can't avoid following a set of rules that you've developed
* reduces a risk of releasing a bug or regression, thus improving quality

Gitlab is as generous as giving you their CI for free, even for your private repos. At this point it is starting to look as advertising, therefore a quick comparison table for [Gitlab](https://about.gitlab.com/pricing/), Github, [Bitbucket](https://bitbucket.org/product/pricing). AFAIK, Github does not have a built-in solution, instead it is easily integrated with third parties, of which [Travis CI](https://github.com/marketplace/travis-ci/plan/MDIyOk1hcmtldHBsYWNlTGlzdGluZ1BsYW43MA==#pricing-and-setup) seems to be the most popular - I will therefore mention Travis here.

### Public repositories (OSS projects). All 3 providers have a free offer for the open-source community!

| Provider | Limits |
|---|---|
| Gitlab | 2,000 CI pipeline minutes per group per month, shared runners |
| Travis | Apparently unlimited |
| Bitbucket| 50 min/month, max 5 users, File storage <= 1Gb/month |

### Private repositories

| Provider | Price | Limits |
|---|---|---|
| Gitlab | Free | 2,000 CI pipeline minutes per group per month, shared runners |
| Travis | $69/month | Unlimited builds, 1 job at a time |
| Bitbucket| Free | 50 min/month, max 5 users, File storage <= 1Gb/month |

## Getting started

I made a small project based on Laravel framework and called it "ci-showcase". I work in Linux environment, and the commands I use in the examples, are for linux shell. They should be pretty much the same on Mac and nearly the same on Windows though.

```sh
composer create-project laravel/laravel ci-showcase
```
