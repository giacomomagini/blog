---
title: "Multiple Sentry instances on a single web page"
date: 2020-12-05T15:00:07Z
draft: false
tags: [sentry, mfe, frontend, spa]
---
## Introduction

As we all know Front-End got more complex and complex with more responsibility moving towards it.
As the Front-End grew, also the engineering world around it evolved coming up with new tools/frameworks and architectural patterns to keep these projects maintainable by larger engineering teams.

One of the architectural pattern I know very well it's Micro Front-End (MFE).
This pattern breaks apart a front-end application into smaller applications developed and shipped independently by different teams.
Every smaller application it's a Micro Front-End.

Although the applications are independent they all run in the same webpage into a main application working as a container that orchestrates these MFEs.

## One (big) downside

One of the hardest part of MFEs it's handling global instances.
It's not good practice to use global instances and you probably won't go that way designing your application, but what about third-party libraries?
`Sentry` was one of them.

This means you needed to have a single Sentry project for multiple MFEs.
This is bad cause it violates the principle of isolation and independence MFE aims for and could cause issues if Sentry's versions in the MFEs are incompatible enough.

## The dream comes true

I've been periodically looking online and on Sentry's docs how to have a local instance of Sentry's client but unfortunately, I've found nothing.... since three days ago! :tada:

With `@sentry/browser` version `5.28.0` it's now possible to create a local instance of sentry and use it whenever you want through imports.
Here a really simple example:

```typescript
import { BrowserClient, Hub } from "@sentry/browser";

// These variables are replaced by webpack at build time
declare const RELEASE: string;
declare const ENV: string;
declare const SENTRY_DSN: string;

const client = new BrowserClient({
  dsn: `${SENTRY_DSN}`,
  environment: `${ENV}`,
  release: `${RELEASE}`,
});

const hub = new Hub(client);

export const sentry = hub;
```

## Official resources

Of course, you can do more than what I did in my script, like customise the scope with tags.
Here the official documentation for [using a client directly](https://docs.sentry.io/platforms/javascript/troubleshooting/#using-a-client-directly).
