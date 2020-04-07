---
title: "Redirector Library"
date: 2020-04-06T20:29:07+02:00
draft: false
tags: ["redteam", "infra"]
---

tl;dr : https://github.com/upils/redirect-lib

## We need redirectors

I have started a project about `redirectors`. `redirectors` as in `command and control redirectors`.

My goal is to document and automate the deployment of various methods to redirect incoming traffic from a client (a compromized host) to another server (a C&C such as Cobalt Strike or Covenant). **The main purpose of this kind of tool is to hide the main server address and thus gain flexibility**.

If the payload is detected and the domain/IP address of the redirector if blocked, being able to setup another "entrypoint" is a key element in a redteam infrastructure.

## "L'argent c'est le nerf de la guerre"

Automating deployment/infrastructure tasks has multiple advantages like : limit OPSEC fails, increase reactivity and reduce costs. We are able to deploy and adapt faster, thus cheaper. So, **another goal of this project is to find cheapest solution** while ensuring the same "service level".

That is why I have, for now, excluded using [AWS spot instances](https://aws.amazon.com/ec2/spot/). Using these instances could further reduce the cost of some methods (basically every method needing a full cloud instance) but there is no warranty one instance will keep running full time during an engagement. Furthermore, as this solution are based on a "pool" of resources, I would need to manage on the fly DNS entry or add other components (AWS ELB ?) to the cloud stack to keep the redirection running.

## A word on categories

I have sorted methods in two main categories : `smart` and `dumb` methods.

- `smart` ones have features enabling the operator (you) to customize the redirection behavior. You may change redirection rules based on User Agent, source IP, URI requested, result of a fingerprinting script, etc.
- `dumb` well, not `smart`. so the method only redirect, that's it.

Then, in the `smart` category, I have mainly researched 3 kind of methods:

- web proxies (apache, nginx, etc.)
- CDN (cloudflare, Azure CDN, etc.)
- cloud functions

I would like to find completely different method to add more variety.

The project if available on [github](https://github.com/upils/redirect-lib).

## Under the hood

On every method researched, redirection is operated using either Layer 4 (TCP/IP) or Layer 7 (Application -> HTTP).

Methods such as `socat`, `ssh` or `nginx` (in "stream" mode) use Layer 4 (TCP/IP) redirection. As such, they are able to redirect HTTPS traffic without any specific configuration. The drawback is : as no decryption is done, no smart/custom filtering can be done. Every request submitted to the redirector will go straight to the C2. The only exception is the use of `ssl_preread` on the nginx method to read the SNI and know the domain requested.

Methods such as `cloudflare-wokers` or `traefik` use Layer 7 redirection. They need to decrypt the HTTPS traffic and are able to apply filtering.
