---
title: Go App Engine SDK 1.5.5 released
date: 2011-10-11
by:
- Andrew Gerrand
tags:
- appengine
- gofix
- release
summary: Go App Engine SDK 1.5.5 includes Go release.r60.2.
---


Today we released version 1.5.5 the Go App Engine SDK.
You can download it from the [App Engine downloads page](http://code.google.com/appengine/downloads.html).

This release includes changes and improvements to the App Engine APIs and
brings the supporting Go tool chain to [release.r60.2](https://golang.org/doc/devel/release.html#r60)
(the current stable release).
Also included in this release are the [godoc](https://golang.org/cmd/godoc/),
[gofmt](https://golang.org/cmd/gofmt/),
and [gofix](https://golang.org/cmd/gofix/) tools from the Go tool chain.
They can be found in the root directory of the SDK.

Some changes made in this release are backwards-incompatible,
so we have incremented the SDK `api_version` to 3.
Existing apps will require code changes when migrating to `api_version` 3.

The gofix tool that ships with the SDK has been customized with App Engine-specific modules.
It can be used to automatically update Go apps to work with the latest appengine
packages and the updated Go standard library.
To update your apps, run:

	/path/to/sdk/gofix /path/to/your/app

The SDK now includes the appengine package source code,
so you can use the local godoc to read App Engine API documentation:

	/path/to/sdk/godoc appengine/datastore Get

**Important note:** We have deprecated `api_version` 2.
Go apps that use `api_version` 2 will stop working after 16 December 2011.
Please update your apps to use `api_version` 3 before then.

See the [release notes](http://code.google.com/p/googleappengine/wiki/SdkForGoReleaseNotes)
for a full list of changes.
Please direct any questions about the new SDK to the [Go App Engine discussion group](http://groups.google.com/group/google-appengine-go).
