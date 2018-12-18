---
layout: post
title: "migrating from govendor to dep"
description:
date: 2018-04-10
tags:
comments: false
---


I love Go, but I can't say that Go package management is my favorite thing. Anytime I work in python (pip) or javascript (npm) or ruby (bundler), I breathe a little easier.

But in Go, there's just [a lot going on](https://github.com/blindpirate/report-of-build-tools-for-java-and-golang), frequent change (tools come and go), and nothing is officially supported yet in the toolchain. I have helped start and maintain maybe half a dozen Go repositories in the last year, and up until today, all of them used [govendor](https://github.com/kardianos/govendor).

But! There is hope with [dep](https://golang.github.io/dep/), an awesome (official!) experimental dependency manager. Once I heard that dep is on its way to be go official, I was excited to give it a try.

And so my first task was to try to [migrate](https://golang.github.io/dep/docs/migrating.html) a new API server I'm building from govendor to dep.

I started by removing the `/vendor` folder that was hidden away into the root of the repository, removing the (govendor-specific) `vendor.json` file, then running `dep init` from the root.

This command builds a vendor folder and a `Gopkg.toml` / `Gopkg.lock`, through magic (Inference + Solving).

I ran `dep init` for the first time and got the all clear!

From there, I could run `dep status` and see something like this:

```
➜  api-server git:(master) dep status
PROJECT                              CONSTRAINT     VERSION        REVISION  LATEST   PKGS USED
github.com/gogo/protobuf             v1.0.0         v1.0.0         1adfc12   v1.0.0   2
github.com/golang/glog               branch master  branch master  23def4e   23def4e  1
github.com/golang/protobuf           v1.0.0         v1.0.0         9255415   v1.0.0   5
github.com/google/gofuzz             branch master  branch master  24818f7   24818f7  1
github.com/googleapis/gnostic        v0.1.0         v0.1.0         ee43cbb   v0.1.0   3
github.com/howeyc/gopass             branch master  branch master  bf9dde6   bf9dde6  1
github.com/imdario/mergo             v0.3.4         v0.3.4         9d5f127   v0.3.4   1
github.com/json-iterator/go          1.1.3          1.1.3          ca39e5a   1.1.3    1
...
```

This CLI output is just prettified `Gopkg.lock`. I really like how you can see the revision SHA along with the tagged releases that dep has cloned.

As I added some code to the API server, along with a few import statements, all I had to do was run `dep ensure -add <package-name>` for each external package I needed.

When I tried to rebuild my API server binary and got a few hundred errors from the vendored packages-- unrecognized types, invalid parameters, the gamut. My intuition was that dep's Inference/Solving didn't account for certain version requirements between indirect dependencies.

The specific issue was between the vendored [Helm API](https://github.com/kubernetes/helm/blob/master/docs/developers.md) and its dependency, [grpc-go](https://github.com/grpc/grpc-go).

After some sleuthing, I learned that dep had pulled a totally different version of the Helm API than the one my team was using, with `govendor`. This was because my teammate did a local `govendor add +m` (or the like) to add local packages in his `GOPATH` to the `/vendor` folder, whereas dep does clean clones only, and puts them in a special spot. However, while my teammate had pulled a specific release of the Helm API, dep had pulled the master branch.

I got around this by creating a `constraint` in my `Gopkg.toml` --

```
[[constraint]]
  name = "k8s.io/helm"
  version = "v2.8.2"
```

Then re-running `dep ensure`. This downgraded the Helm API to the latest stable release. But! Then I tried to rebuild the api server binary and still got a ton of grpc errors. Hmm.

After some [more sleuthing](https://github.com/kubernetes/helm/blob/release-2.8/glide.yaml#L28) I learned that the latest release of Helm used a slightly older version of go-grpc. Aha!

And so after creating one last `constraint`, using `=` to fix the `go-grpc` version at exactly the one that helm requires...

```
[[constraint]]
  name = "google.golang.org/grpc"
  version = "=v1.7.2"
```

My `Makefile` finally completed, and our repo was back in business! This was a great first introduction to dep, and so far it's worked really well for our project- I can't wait to continue using it as the project develops.
