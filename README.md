# Go website

[![Go Reference](https://pkg.go.dev/badge/golang.org/x/website.svg)](https://pkg.go.dev/golang.org/x/website)

This repo holds content and serving programs for the golang.org and go.dev web sites.

Content is in _content/ (golang.org), go.dev/_content/ (go.dev), and tour/ (tour.golang.org).
Server code is in cmd/ and internal/.

To run the combined golang.org+go.dev server to preview local content changes, use:

	go run ./cmd/golangorg

The supporting programs cmd/admingolangorg and cmd/googlegolangorg
are the servers for admin.golang.org and google.golang.org.
(They do not use the _content/ directories.)

Each command directory has its own README.md explaining deployment.

## JS/CSS Formatting

This repository uses [prettier](https://prettier.io/) to format JS and CSS files.

The version of `prettier` used is 1.18.2.

It is encouraged that all JS and CSS code be run through this before submitting
a change. However, it is not a strict requirement enforced by CI.

## Report Issues / Send Patches

This repository uses Gerrit for code changes. To learn how to submit changes to
this repository, see https://golang.org/doc/contribute.html.

The main issue tracker for the website repository is located at
https://github.com/golang/go/issues. Prefix your issue with "x/website:" in the
subject line, so it is easy to find.

