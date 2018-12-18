---
layout: post
title: "writing a Pocket -> Kindle tool in Go"
description:
date: 2018-04-29
tags:
comments: false
---

I love to read, and I love physical books, but I also 1) travel a lot and 2) don't own (or like to own) a ton of stuff. And so I own a Kindle.

I also use [Pocket](https://getpocket.com/a/queue/list/) to save articles, short stories, poems, etc. to read later.

When I first got my Kindle, I looked into existing tools that forward Pocket articles to your kindle. These tools exist ([this one](https://p2k.co/), or [this one](https://www.crofflr.com/#/home)), but I don't love the way these services combine articles into one Kindle document. Once you get a lot of these Pocket deliveries to your Kindle, it makes it difficult to find a specific article. Plus, most of these tools aren't free.

So to take the problem into my own somewhat capable hands, I decided to create a small tool called "pocketsender" that can forward all your unread Pocket articles to your Kindle, as separate documents.

## how?

First, I knew that I'd have to use the [Pocket API](https://getpocket.com/developer/) to get my articles, then use Amazon's ["email to kindle"](https://www.amazon.com/gp/sendtokindle/email) feature that allows you to attach PDFs. Amazon's backend for this will auto-convert PDFs if the email subject line is "convert."

Quickly, I realized there would be a problem: when you call the Pocket API to get your articles, it doesn't give you back the article text (assumedly for copyright reasons). You only get an excerpt of the article, and the URL.

And so the workflow would have to go like this:

1. 🔑 Authenticate with the Pocket API
2. Get all my unread articles. For each article...
3. Get the URL, navigate to it, and grab the HTML contents
4. 📄 Convert the HTML to PDF
5. Save that PDF file to a local dir
6. 💌 Dynamically construct an email, uploading that PDF as an attachment
7. Send that email to my Kindle's email address. Then the Amazon backend converts the PDF attachment to Kindle format, and sends it to my Kindle device.

Not the prettiest, but it will get the job done! Let's implement it.

## go scaffolding

I knew ahead of time that I'd want to have a [Docker](https://www.docker.com/) image that can execute a Go binary, with some input config. So I implemented the pocketsender as a Go CLI that could run as a standalone executable or in Docker.

I used [urfave/cli](https://github.com/urfave/cli), and wrote a `main.go` that runs my [`pocketsender.Check()`](https://github.com/m-okeefe/pocketsender/blob/master/main.go#L51) function. This `Check` function will hold the whole 7-step workflow described above.

I also knew that I'd have to vendor some dependencies. And since I'm a [dep convert](http://m-okeefe.com/dep), I ran `dep.init` to initialize my `vendor/` folder.  

## 🔑 pocket API

After some poking around, I found that someone [already wrote](https://github.com/motemen/go-pocket) a Pocket API client in Go -- score! So I vendored that and used its [`client.Retrieve()`](https://github.com/motemen/go-pocket/blob/master/api/retrieve.go#L153) function to get [unread](https://github.com/motemen/go-pocket/blob/master/api/retrieve.go#L27) Pocket articles.

I also used the same client to [archive](https://github.com/motemen/go-pocket/blob/master/api/modify.go#L10) each Pocket article after emailing it to my Kindle, so that it would no longer show up in my Pocket List (or be emailed again on a subsequent run).

## 📄 pdf generation

Now that I can get the URLs of all my unread articles, I need a way to pull down PDFs of each. I didn't really want to write my own tool that does this, so I used an existing Go [wrapper](https://github.com/SebastiaanKlippert/go-wkhtmltopdf) on [wkhtmltopdf](https://wkhtmltopdf.org/index.html), which is open source.

For this step, I instantiate a PDF converter, pass `wkhtmltopdf` the URL of an article, and save the resulting file in a temporary `/pdf` directory.

Unfortunately as I tested my Go binary on my Mac, I encountered some "SSL Handshake Error" errors on a few of the article conversions, possibly [this bug](https://github.com/wkhtmltopdf/wkhtmltopdf/issues/2912). But I temporarily backburnered this problem because I knew I'd be containerizing it, and thus running in Linux, not MacOS.

## 💌 email to Kindle  

For composing and sending emails to my Kindle email address, I used the [gomail](https://github.com/go-gomail/gomail) package, which worked great. I simply compose an email where the `to` and `from` addresses are based on the user's config. Then I [attach](https://github.com/m-okeefe/pocketsender/blob/master/pkg/pocketsender/pocketsender.go#L139) the pdf I just generated. Then send it off to Amazon's servers to do the `pdf` -> `azw` conversion and the actual sending to device. :)

## ✨ make the magic happen

After finishing my Go code, I wrote a simple [`Makefile`](https://github.com/m-okeefe/pocketsender/blob/master/Makefile) and [`Dockerfile`](https://github.com/m-okeefe/pocketsender/blob/master/Dockerfile) to containerize it. The base image I used is Ubuntu [with wkhtmltopdf pre-installed](https://hub.docker.com/r/openlabs/docker-wkhtmltopdf/).   

```
$ docker run -v /Users/meokeefe/Documents/go/src/github.com/m-okeefe/pocketsender/config.json:/pocketsender/config.json  meganokeefe/pocketsender:0.0.1 --config /pocketsender/config.json

****************************************************************************************
P O C K E T S E N D E R             v0.0.1
****************************************************************************************
2018/04/29 13:21:03 Instantiating Pocket API client...
2018/04/29 13:21:03 Retrieving unread Pocket articles for account...
2018/04/29 13:21:04 Got 100 unread pocket articles, emailing...
2018/04/29 13:21:04

 ---> Article 1/43:
2018/04/29 13:21:04 Converting to pdf:  http://inference-review.com/article/time-travelers
2018/04/29 13:21:13 Emailing PDF to kindle...
2018/04/29 13:21:20 Sent!
```

## what's next

Now that I have a working tool, there are lots of things I want to do to improve it! Including:

- Prevent users from having to pass in their plaintext gmail password (maybe have pocketsender use [Vault](https://www.vaultproject.io/), and force users to pass in secrets in a secure way)
- Instead of running as a "one and done," have pocketsender watch the user's pocket queue for new articles. This would let me (for instance) deploy pocketsender in [Kubernetes](https://kubernetes.io/), Cloud, etc. and forget about it : )
- Integrate more tightly with the Pocket API- for example, support autogenerating Access Keys. [go-pocket](https://github.com/motemen/go-pocket/blob/master/auth/auth.go) does this well.
- Reduce the Docker image size. Right now [it's a whopping](https://hub.docker.com/r/meganokeefe/pocketsender/tags/) 301MB, given that the [base image](https://hub.docker.com/r/openlabs/docker-wkhtmltopdf/tags/) I'm using is 200+ MB.   
- Fix the OpenSSL bugs so that pocketsender can also run from source on Mac, in addition to Linux. :)

The initial source code is [here](https://github.com/m-okeefe/pocketsender). Thanks for reading!
