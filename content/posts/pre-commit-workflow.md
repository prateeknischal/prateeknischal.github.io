---
title: "Test before you Go and Commit"
date: 2020-11-01T01:04:22+05:30
description: "Improve your go workflow using pre-commit githooks"
---


There is a certain confidence boost when you see that message "All tests pass",
unless you are like me and get lazy, skimping on writing good tests for your
logic. There are a lot of tools integrated with github, gitlab and probably
other git interfaces that run a set of pre-defined tasks before potentially
allowing you to raise a PR or mark it safe for merge. For the uninitiated, it's
know as [CI or continuous
integration](https://en.wikipedia.org/wiki/Continuous_integration) tools. For
example,
* [Azure pipelines](https://github.com/marketplace/azure-pipelines)
* [Github actions](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions)
* [CicleCI](https://github.com/circleci)
* [TravisCI](https://travis-ci.org/)

What if you could run those tests, lint checks and other stuff before you push
your commit and realize it's going to fail in your CI pipeline and make you look
like a n00b. Wouldn't that be great !

Unfortunately, there aren't a lot of tools that do the same for plain old git, I
mean locally in your development environment. There is one for nodejs, called
the [pre-commit](https://www.npmjs.com/package/pre-commit), but let's ignore
that since we are gophers here.

There is a project
[pre-commit/pre-commit](https://github.com/pre-commit/pre-commit) that can be
used to manage a wide variety of pre-commit stuff and in a pretty way. It looks
really good ! The only gripe that I have with it (highly personal) is it needs a
runtime. I am not saying that runtimes are bad, they just feel a bit bloated and
then you run into their dependencies and versions and give up.  The most painful
transition for me has been `python2.x` to `python3.x`. So, I try to avoid python
until `python3` is an adopted standard and the only version present in all of my
machines. If that's not a concern for you, then feel free to use the pre-commit
project. It will probably suit your needs much better.

If you share similar thoughts as me or don't have anything better to do than
reading this, let's see how these things are implemented.

### Hooks all over the place
I can feel the incoming disappointment. Git, the vanilla thing has something
called [githooks](https://git-scm.com/docs/githooks) at different stages that
you can use to hook in and run your stuff.

Let's try something !
```bash
$ git clone https://github.com/cloudmarker/cloudmarker.git
$ exa --tree cloudmarker/.git/hooks
.git/hooks
├── applypatch-msg.sample
├── commit-msg.sample
├── fsmonitor-watchman.sample
├── post-update.sample
├── pre-applypatch.sample
├── pre-commit.sample
├── pre-push.sample
├── pre-rebase.sample
├── pre-receive.sample
├── prepare-commit-msg.sample
└── update.sample
```

Notice the
[`pre-commit.sample`](https://github.com/git/git/blob/master/templates/hooks--pre-commit.sample)
file. It has a sample defintion that could be run before the commit, as the name
suggests. What if we create a file `pre-commit` with the scripts that we need to
run, tests, lints, checks, print xkcd, whatever be it.

That script would get called before you try and commit and the commit would fail
if the script exited with a non-zero error code, thus preventing you from your
impulsive force pushes.

### *make* it more manageable

This is a file present in the `.git` folder that is not under version control, so
how do you manage it better. If the commands or workflow changes in between, you
would have to keep updating the script manually which is not what we want.

One way to do it is, softlinks. Create a softlink to your version controlled
script file and make the pre-commit hook to point your script. A sample project
structure that you make want to follow.

```bash
$ exa --tree --level 1 .
.
├── go.mod
├── go.sum
├── main.go
├── main_test.go
├── Makefile
└── .pre-commit
```

The project contains a version controlled file called `.pre-commit` (a dotfile
to keep the folder view clean) which will contain your commands that you may
need to run.

The first time setup can be automated using your favourite tools. Let's try to
do something using [`gnu make`](https://www.gnu.org/software/make/manual/make.html)
(which is **not** my favourite).

Contents of the `.pre-commit`.
```sh
#/bin/bash
./.git/hooks/pre-commit.sample
make pre-commit
```

The make target.
```make
pre-commit:
    @go test ./...
    @go fmt ./...
    @goimports -w .
    @golint ./...

    @go vet ./...
    @gocyclo -over 10 .

deps:
    @echo "Installing tools: goimports, golint, gocyclo"
    @go get -u golang.org/x/lint/golint
    @go get -u github.com/fzipp/gocyclo
    @go get golang.org/x/tools/cmd/goimports

    @echo "Setting up pre-commit hook"
    @ln -snf ../../.pre-commit .git/hooks/pre-commit
    @chmod +x .pre-commit
```
Notice the `deps` target that downloads all the tools that I am using in my
pre-commit flow and then sets a softlink to the git hooks pointing to the
relative path of the `.pre-commit` file.

This may seem a bit annoying (to have an indirection from the pre-commit file to
the makefile) but it gives you a minimal pre-commit file and all of dependencies
and complexities are captured and controlled from a single place which is your
build mechanism, `make` in this case,

### Using this workflow
To setup this workflow, you could just do
```bash
$ make deps
```
And you are all set with the dependencies for your project and the git hook as
well. The next time you try a commit, it will run the whole `pre-commit` target
and break your commit if have messed something up automatically.

I like this because it's minimal (has it's own downsides) and does not require
installing any extra tools or libraries other than what you anyways need (make
is usually present on most unix systems). If your project is large and this is
not enough, I would recommend going with a more verbose and configurable tool,
but for small project, this should be familiar enough. I have seen people
customize their makefiles to extremes, so maybe that's all they need.

---
Discussion thread:
[here](https://www.reddit.com/r/golang/comments/jlqht8/a_minimal_precommit_go_workflow/)

