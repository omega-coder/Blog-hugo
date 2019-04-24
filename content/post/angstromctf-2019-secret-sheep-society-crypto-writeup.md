---
title: "AngstromCTF 2019 Secret Sheep Society Crypto 120 Writeup"
author: "CHERIEF Yassine (omega-coder)"
date: 2019-04-24T09:27:58+02:00
subtitle: "AES CBC Mode bit-flipping attack"
tags: ["ctf", "crypto", "bit-flipping", "aes", "security"]
categories: ["security", "writeups"]
---

# TL;DR

1. Spotting the weakness (AES with CBC Mode).
2. Get token
3. Flipping specific bytes in session json (turn **`flase`** to **`true `**).
4. manipulate token with flipped bytes.
5. Send manipulated token to page.
6. Get the flag. 

<hr>

## Challenge details 

| **Category**  | **Points** | **Solves** |
|-----------|--------|--------|
| Crypto | 120    | 0     |

<hr>

## Challenge Description

> The sheep are up to no good. They have a [web portal](https://secretsheepsociety.2019.chall.actf.co/) for their secret society, which we have the source for. It seems fairly easy to join the organization, but climbing up its ranks is a different story.<br> Author: defund










