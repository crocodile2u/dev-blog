# Introduction to Gitlab CI for PHP developers

As a developer, you've probably at least heard something about [CI - Continuous integration](https://en.wikipedia.org/wiki/Continuous_integration). And if you haven't - you better fix it ASAP, because that's something awesome to have on your skill list and can get extremely helpful in your everyday work. This post will focus on CI for PHP devs, and specifically, on CI implementation from [Gitlab](https://docs.gitlab.com/ee/ci/README.html). I will suppose you know the basics of [Git](https://git-scm.com/), [PHP](https://php.net/), [PHPUnit](https://phpunit.de/), [Docker](https://www.docker.com/) and unix shell. Intended audience - intermediate PHP devs.

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

# What's happening here?
#
# 1. We get names and statuses of files that differ in current branch from their state in origin/master.
#    These come in form <status> <path> (multiline)
# 2. The output from git diff is filtered by unix grep utility, we only need files with names ending in .php
# 3. One more filter: filter *out* (grep -v) all lines starting with R or D.
#    D means "deleted", R means "renamed"
# 4. The filtered status-name list is passed on to awk command, which is instructed to take only the 2nd part
#    of every line, thus just the filename

git diff --name-status origin/master | grep '\.php$' | grep -v "^[RD]" | awk '{ print $2 }'
```

These scripts can easily be tested in your local environment ( at least if you have a Linux machine, that is ;-) ).

Now, as we have our first check, we'll finally create our `.gitlab-ci.yml`. This is where your pipeline is declared using [YAML notation](https://yaml.org/):

```yml
# we're using this beautiful tool for our pipeline: https://github.com/jakzal/phpqa
image: jakzal/phpqa:alpine

# For this sample pipeline, we'll only have 1 stage, in real-world you would like to also add at least "deploy"
stages:
  - QA

linter:
  stage: QA
  # this is the main part: what is actually executed
  script:
    - sh ci/get-changed-php-files.sh | xargs sh ci/linter.sh
```

The first line is `image: jakzal/phpqa:alpine` and it's telling Gitlab that we want to run our pipeline using a PHP-QA utility by [jakzal](https://github.com/jakzal). It is a docker image containing PHP and a huge variety of QA-tools. We declare one stage - QA, and this stage by now has just a single job named `linter`. Every job can have it's own docker image, but we don't need that for the purpose of this tutorial. Our project reaches [Step 2](https://gitlab.com/crocodile2u/ci-showcase/tree/step-2). Once I had pushed these changes, I immediately went to the [project's CI/CD page](https://gitlab.com/crocodile2u/ci-showcase/pipelines). Aaaand.... the pipeline was already running! I clicked on the `linter` job and saw the following happy green output:

```
Running with gitlab-runner 11.9.0-rc2 (227934c0)
  on docker-auto-scale ed2dce3a
Using Docker executor with image jakzal/phpqa:alpine ...
Pulling docker image jakzal/phpqa:alpine ...
Using docker image sha256:12bab06185e59387a4bf9f6054e0de9e0d5394ef6400718332c272be8956218f for jakzal/phpqa:alpine ...
Running on runner-ed2dce3a-project-11318734-concurrent-0 via runner-ed2dce3a-srm-1552606379-07370f92...
Initialized empty Git repository in /builds/crocodile2u/ci-showcase/.git/
Fetching changes...
Created fresh repository.
From https://gitlab.com/crocodile2u/ci-showcase
 * [new branch]      master     -> origin/master
 * [new branch]      step-1     -> origin/step-1
 * [new branch]      step-2     -> origin/step-2
Checking out 1651a4e3 as step-2...

Skipping Git submodules setup
$ sh ci/get-changed-php-files.sh | xargs sh ci/linter.sh
Job succeeded
```

It means that our pipeline was successfully created and run!

### PHP Code Sniffer.

[PHP Code Sniffer](https://github.com/squizlabs/PHP_CodeSniffer) is a tool for keeping app of your PHP files in one uniform code style. It has a hell of customizations and settings, but here we will only perform simple check for compatibilty with [PSR-2](https://www.php-fig.org/psr/psr-2/) standard. A good practice is to create a configuration XML file in your project. I will put it in the root folder. Code sniffer can use a few file names, of which I prefer `phpcs.xml`:

```xml
<?xml version="1.0"?>
<ruleset name="Custom Standard" namespace="MyProject\CS\Standard">

	<!-- do not use <exclude-pattern type="relative"> - works unexpectedly when PHPCS invoked for a single PHP file -->
	<exclude-pattern>/resources/</exclude-pattern>
	<exclude-pattern>/database/</exclude-pattern>
	<exclude-pattern>/routes/</exclude-pattern>
	<exclude-pattern>/storage/</exclude-pattern>
	<exclude-pattern>/config/</exclude-pattern>

    <!-- the heart of our PHPCS setup - apply PSR-2 standard -->
	<rule ref="PSR2"/>

</ruleset>
```

I also will append another section to `.gitlab-ci.yml`:

```yml
code-style:
  stage: QA
  script:
    # Variable $files will contain the list of PHP files that have changes
    - files=`sh ci/get-changed-php-files.sh`
    # If this list is not empty, we execute the phpcs command on all of them
    - if [ ! -z "$files" ]; then echo $files | xargs phpcs; fi```

Again, we check only those PHP files that differ from master branch, and pass their names to `phpcs` utility. That's it, [Step 3](https://gitlab.com/crocodile2u/ci-showcase/tree/step-3) is finished! If you go to see the pipeline now, you will notice that `linter` and `code-style` jobs run in parallel.

# Adding PHPUnit

Unit and integration tests are essential for a successful and maintaiable modern software project. In PHP world, [PHPUnit](https://phpunit.de/) is de facto standard for these purposes. The PHPQA docker image already has PHPUnit, but that's not enough. Our project is based on [Laravel](https://laravel.com/), which means it depends on a bunch of third-party libraries, Laravel itself being one of them. Those are installed into `vendor` folder with [composer](https://getcomposer.org/). You might have noticed that our `.gitignore` file has `vendor` folder as one of it entries, which means that it is not managed by the Version Control System. Some prefer their dependencies to be part of their Git repository, I prefer to have only the `composer.json` declarations in Git. Makes the repo much much smaller than the other way round, also makes it easy to avoid bloating your production builds with liraries only needed for development.
