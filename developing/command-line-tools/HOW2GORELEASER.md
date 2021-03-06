# [Tools and Practice](../README.md) / Building internal tools with GoReleaser

## Overview

This is a biased document for how to build and release tools written in go.

If someone else has already written a tool and open sourced it use it and contribute to it.

Feel free to update and make changes!

## Split a tool out from an existing project

Follow these steps if this is a tool you developed in another repository and you need to preserve git history.
You can skip this if you are creating a completely new tool.

In your terminal clone a copy of the original repo into a new folder:

```sh
git clone git@github.com:OWNER/REPONAME.git NEWREPO
```

In that new repo folder remove the origin:

```sh
git remote rm origin
```

Filter out commits that change the specified directory:

```sh
git filter-branch --prune-empty --subdirectory-filter DIRNAME master
```

Create a new repo either in the GitHub UI or directly in [Infra Management Repo](https://github.com/trussworks/legendary-waddle/tree/master/trussworks-prod/github-global).

Note: This repo should be public and properly licensed. You can use a Github license template for MIT or BSD-3.

If you are making a repo in the Trussworks GitHub org, please add the repo to our [Infra Management Repo](https://github.com/trussworks/legendary-waddle/tree/master/trussworks-prod/github-global).
If you need help please reach out to the #infrasec Slack channel.

Add the new repo as upstream:

```sh
git remote add origin https://github.com/OWNER/NEWREPO.git
```

Push history:

```sh
git push origin .
```

### References

* [GitHub's documentation](https://help.github.com/en/github/using-git/splitting-a-subfolder-out-into-a-new-repository)
* [Chris Gilmer’s full instructions](
https://github.com/chrisgilmerproj/silliness#how-to-break-out-projects)

## Get the project building binaries

This assumes you are creating a binary from a Go project.

### Generate a `go.mod` file

Your project may not have a `go.mod` file already, you can skip ahead if so.

Initializes your `go.mod` file with your current version of Go:

```sh
go mod init github.com/OWNER/NEWREPO
```

Fills out your `go.mod` and `go.sum` files and builds your binary for your current OS:

```sh
go build
```

You might run into some issues getting this to work outside of a project you may have split this code from. Spend some time getting it to build.

When you feel good about it, add `go.mod` and `go.sum` to your git index with a commit.

[Blog post I used to figure this out](https://medium.com/mindorks/create-projects-independent-of-gopath-using-go-modules-802260cdfb51)

## Add pre-commit hooks

Write a `.pre-commit-config.yaml` file and add it to git with a commit.
Then install the hooks and run the hooks against all files for the first time.

```sh
pre-commit install --install-hooks
pre-commit run --all-files
```

Fix any issues you find and commit your changes to your repo.

### Example .pre-commit-config.yaml and .markdownlintrc

The following example `.pre-commit-config.yaml` does some nice things for a basic go project.

```yml
repos:
  - repo: git://github.com/golangci/golangci-lint
    rev: v1.21.0
    hooks:
      - id: golangci-lint

  - repo: git://github.com/pre-commit/pre-commit-hooks
    rev: v2.4.0
    hooks:
      - id: check-json
      - id: check-merge-conflict
      - id: check-yaml
      - id: detect-private-key
      - id: pretty-format-json
        args:
          - --autofix
      - id: trailing-whitespace

  - repo: git://github.com/igorshubovych/markdownlint-cli
    rev: v0.21.0
    hooks:
      - id: markdownlint
```

This example `.markdownlintrc` will work with the above `.pre-commit-config.yaml` example.
This ignores some stylistic but annoying triggers.

```json
{
  "default": true,
  "first-header-h1": false,
  "first-line-h1": false,
  "line_length": false,
  "no-multiple-blanks": false,
  "fenced-code-language": false
}
```

## Create Docker configuration

Suggested but optional. This is just another distribution method.

Create a `Dockerfile` at the root of your repo and commit it to git.

### Example Dockerfile

This example is a very basic Dockerfile that works with goreleaser with a tool/binary name of `binaryname`.

Goreleaser will use the binary it determines is correct to copy into the container.

```docker
FROM alpine:3
COPY binaryname /bin/binaryname
ENTRYPOINT [ "binaryname" ]
```

## Create a Docker repo

Create a Docker repo. There's Docker Hub, ECR, and GitHub.
You'll need to be sure to enable automation to push into whatever you use.

### Create a Docker Hub repo

We don't use the trussworks Docker Hub org very much so you'll need to reach out to the #infrasec Slack channel to find someone to help you.

Create a new Docker Hub repo following the same naming convention of the repo like `trussworks/NEWREPO`.

Configure the bot user to read/write to that repo.

Use the bot user credentials (username and a deploy key) for configuring CircleCI.

## Create goreleaser configuration

Create a `.goreleaser.yml` file and commit it.

### Example .goreleaser.yml

This has some decent build defaults. This will get dependencies from go.mod then build for both OSX and Linux.
Goreleaser will also update the release in GitHub with the artifacts it builds.
It will also push to a brew tap. Build a Docker container but skip shipping to it.

Note: I had to add `GO111MODULE=on` and `CGO_ENABLED=0` lines so the linux binary would work in Docker.

If you don't need to build a Docker container, just remove the docker stanza.

```yml
env:
  - GO111MODULE=on
before:
  hooks:
    - go mod download
builds:
- env:
    - CGO_ENABLED=0
  goos:
    - darwin
    - linux
  goarch:
    - amd64
  main: main.go
brews:
  - description: "WRITE A DESCRIPTION"
    github:
      owner: trussworks
      name: homebrew-tap
    homepage: "HOMEPAGE URL GOES HERE"
    commit_author:
      name: trussworks-infra
      email: infra+github@truss.works
dockers:
  -
    binaries:
      - <BINARYNAME>
    image_templates:
      - "OWNER/NEWREPO:{{ .Tag }}"
    skip_push: true
archives:
- replacements:
    darwin: Darwin
    linux: Linux
    windows: Windows
    386: i386
    amd64: x86_64
checksum:
  name_template: 'checksums.txt'
snapshot:
  name_template: "{{ .Tag }}-next"
changelog:
  sort: asc
  filters:
    exclude:
    - '^docs:'
    - '^test:'
```

### Test goreleaser locally

In your terminal from the root of your repo, run goreleaser without releasing with:

```sh
goreleaser --snapshot --skip-publish --rm-dist
```

This will create all of your binaries in the `dist` folder in your repo.

## Hook up CI

Write a CircleCi config file and commit it to your repo.
You will need to do some manual configuration in CircleCi to get this working.

### CircleCi config.yml example

Add the contents of this code block to .circleci/config.yml in your repo after setting your repo up with CircleCI.

Also configure the `GITHUB_TOKEN`, `DOCKER_USER`, and `DOCKER_PASS` environment variables from the CircleCi UI.

`GITHUB_TOKEN` is used by goreleaser to update release notes and push binaries to the release on GitHub.

This configuration creates two CircleCI workflows `validate` and `release`.

The `validate` workflow is run on each commit to the repository or branch push.
This will run your pre-commit hooks as defined in `.pre-commit-config.yaml`.

The `release` workflow will only run on certain tag pushes to the repo.
This workflow will setup to build Docker containers then run goreleaser as defined in `.goreleaser.yml`.
After binaries and containers are built by `.goreleaser.yml` this validates the container works by running the container with its default entry point using the `--help` flag.
This should just be a sanity check that the container "does the right thing".

If you are not building a Docker container, you must remove these steps in the release job:

* `setup_remote_docker`
* `Login to Docker Hub`
* `Check Docker container`
* `Docker push`

```yml
version: 2.1

references:
  circleci-docker-primary: &circleci-docker-primary trussworks/circleci-docker-primary:99bee5627ff234eb0f31f5899628bff03df78b6d

jobs:
  validate:
    docker:
      - image: *circleci-docker-primary
    steps:
      - checkout
      - restore_cache:
          keys:
            - pre-commit-dot-cache-{{ checksum ".pre-commit-config.yaml" }}
      - run:
          name: Run pre-commit tests
          command: pre-commit run --all-files
      - save_cache:
          key: pre-commit-dot-cache-{{ checksum ".pre-commit-config.yaml" }}
          paths:
            - ~/.cache/pre-commit
  release:
    docker:
      - image: *circleci-docker-primary
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Run goreleaser
          command: goreleaser --debug
      - run:
          name: Login to Docker Hub
          command: docker login -u $DOCKER_USER -p $DOCKER_PASS
      - run:
          name: Check Docker container
          command: docker run -it OWNER/NEWREPO:<< pipeline.git.tag >> --help
      - run:
          name: Docker push
          command: docker push OWNER/NEWREPO:<< pipeline.git.tag >>
workflows:
  version: 2.1
  validate:
    jobs:
      - validate
  release:
    jobs:
      - release:
          filters:
            branches:
              ignore: /^.*/
            tags:
              only: /^v.*/
```

### Run a release from GitHub

Cut a release from `master` with a tag using semantic versioning in the style of `v0.0.0` using the GitHub UI.

This will create a tag and CircleCi will automatically run the `release` workflow.

## Verify you can install from your configured Homebrew Tap

We'll assume you're using the Trussworks Homebrew tap.

Install the Trussworks tap to homebrew and then install the tool you built.

```sh
brew tap trussworks/tap
brew install tool-name
```

Be sure you updated the `README.md` in the new tool repo to have installation instructions.

## Broadcast this to others

Be sure you let other folks know you broke this out for them to use and contribute to! Consider doing one of these to help get more eyes on your new tool repo:

* Add the people you worked with on this tool to your PRs on this repo.
* Add some documentation to your repo and send the repo to Trussels in Slack!
* Do a demo for your team or other teams or as an OTT.
