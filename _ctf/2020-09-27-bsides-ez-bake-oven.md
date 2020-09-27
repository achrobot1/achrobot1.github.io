---
title: "BSides Boston 2020: EZ Bake Oven"
tags: [CTF, web, cookies]
date: 2020-09-27
---


# Challenge
Prompt:
![png](/images/bsides-writeups/ez-bake-oven_prompt.png)

We are given a webpage that looks like this:
![png](/images/bsides-writeups/ez-bake-initial.png)

If we select any option on the left, the timer begins to count down.
Of notable interest is the _Magic Cookie_ , however it would take 7200 minutes for the timer to complete.

When we do place something in the oven, a cookie is set:
![png](/images/bsides-writeups/ez-bake-with-cookie.png)

Looks like base64
![png](/images/bsides-writeups/ez-bake-decoded-cookie.png)

So let's go ahead and write our own cookie and base64 encode it. I changed the date on the cookie to several weeks prior to the current date. I went back to the webpage and set this as the cookie value and got the flag:

![png](/images/bsides-writeups/ez-bake-completed.png)


