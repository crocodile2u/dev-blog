# Introduction to Gitlab CI for PHP developers

As a developer, you've probably at least heard something about [CI - Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration). And if you haven't - you better fix it ASAP, because that's something awesome to have on your skill list and can get extremely helpful in your everyday work. This post will focus on CI for PHP devs, and specifically, on CI implementation from [Gitlab](https://docs.gitlab.com/ee/ci/README.html). I will suppose you know the basics of [Git](https://git-scm.com/), [PHP](https://php.net/), [PHPUnit](https://phpunit.de/) and unix shell. Intended audience - intermediate PHP devs.

Adding something to your workflow must serve a purpose. In this case the goal is to automate routine tasks and achieve better quality control. Even a basic PHP project IMO needs the following:
* [linter](https://en.wikipedia.org/wiki/Lint_(software)) checks (cannot merge changes that are invalid on the syntax level)
* Code style checks
* Unit and integration tests

All of those can be just run eventually, of course. But I prefer an automated CI approach even in my personal projects because it leads to a higher level of discipline, you simply can't avoid following a set of rules that you've developed. Also, it reduces a risk of releasing a bug or regression, thus improving quality.

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

Next, I went to gitlab website and created a new public project: https://gitlab.com/crocodile2u/ci-showcase. Cloned the repo and copied all files and folders from the newly created project - the the new git repo. In the root folder, I placed a `.gitignore` file:

```
.idea
vendor
.env
```
Then the `.env` file:

```
APP_ENV=development
```

Then I generated the application encryption key: `php artisan key:generate`, and then I wanted to verify that the primary setup works as expected: `./vendor/bin/phpunit`, which produced the output `OK (2 tests, 2 assertions)`. Nice, time to commit this: `git commit && git push`

[At this point](https://gitlab.com/crocodile2u/ci-showcase/tree/step-1), we don't yet have any CI, let's do something about it!

### Adding .gitlab-ci.yml

Everyone going to implement CI with Gitlab, is strongly encouraged to bookmark this page: https://docs.gitlab.com/ee/ci/README.html. I will simply provide a short introduction course here plus a bit of boilerplate code to get you started easier.

First QA check that we're going to add is PHP syntax check. PHP has a built-in linter, which you can invoke like this: `php -l my-file.php`. This is what we're going to use. Because the `php -l` command doesn't support multiple files as arguments, I've written a small wrapper shell script and saved it to `ci/linter.sh`:

```sh
#!/bin/sh

last_status=0
status=0

# Loop through arguments and run php -l on each
for f in $@ ; do
    php -l $f > /dev/null
    last_status="$?"
    if [ "$last_status" -ne "0" ]; then
        # Anything fails -> the whole thing fails
        status="$last_status"
    fi
done

if [ "$status" -ne "0" ]; then
    echo "PHP syntax validation failed!"
fi

exit $status
```

Most of the time, you don't actually want to check each and every PHP file that you have. Instead, it's better to check only those files that have been changed. The Gitlab pipeline runs on every push to the repository, and there is a way to know which PHP files have been changed. Here's a simple script, meet `ci/get-changed-php-files.sh`:

```sh
#!/bin/sh

git diff --name-status master | grep '\.php$' | grep -v "^[RD]" | awk '{ print $2 }'
```

These scripts can easily be tested in your local environment ( at least if you have a Linux machine, that is ;-) ).

Now, as we have our first check, we'll create the 
