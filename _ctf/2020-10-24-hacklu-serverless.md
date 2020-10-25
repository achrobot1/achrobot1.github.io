---
title: "Hack.lu CTF 2020: FluxCloud Serverless"
tags: [CTF, web, waf]
date: 2020-10-24
---


# Challenge
Prompt:

![png](/images/hacklu-writeups/serverless-prompt.png)

We are presented with a static page with a 'Demo' button

![png](/images/hacklu-writeups/main-page.png)

Clicking on the Demo button sends us to this page:
![png](/images/hacklu-writeups/demo-page.png)

Not much going on. There no cookies set, url parameters, user input, or anything else that we can explore in the web app. 

We are luckily provided with some source code, so let's take a look there.

The source code is structured:

```
public/
└── app
    ├── Dockerfile
    ├── index.js
    ├── package.json
    ├── package-lock.json
    ├── redis.js
    ├── serverless
    │   ├── billing.js
    │   ├── functions
    │   │   ├── app.js
    │   │   └── waf.js
    │   └── index.js
    └── static
        ├── index.html
        └── tailwind.min.css

4 directories, 11 files
```
## Source code inspection
Looking at <i>app/serverless/functions/app.js</i> I was immediately drawn to the flag endpoint:

```
/**
 * A demo application.
 */

const express = require('express');
const morgan = require('morgan');

const FLAG = process.env.FLAG || 'fakeflag{}';

const router = express.Router();
router.use(morgan('dev'));

router.get('/', (req, res) => {
    res.send('<code><h1>Hello World!</h1><p>Your FluxCloud Serverless deployment works!<p>And it is unhackable™ :)').end();
});

router.get('/flag', (req, res) => {
    res.send(FLAG).end();
});

router.get('/blocked', (req, res) => {
    res.send('<body style="margin:0"><img style="width:100%;height:100%" alt="wait. thats illegal." src="https://i.kym-cdn.com/entries/icons/original/000/028/207/Screen_Shot_2019-01-17_at_4.22.43_PM.jpg"/></body>').end();
});

// wrap in named function for better logging
module.exports = function app(...args) {
    return router(...args);
};
```
However, navigating to demo/:id/flag gave me this:
![png](/images/hacklu-writeups/waf-violation.png)

As hinted by the existence of waf.js, there is a web app firewall that is acting as gatekeeper. 
We can also see in the response that we get a 'X-WAF-Blocked' header that complains about a WAF violation found in the requested URL:
```
HTTP/1.1 403 Forbidden
Server: nginx
Date: Sat, 24 Oct 2020 13:59:18 GMT
Content-Type: text/html; charset=utf-8
Connection: close
X-Powered-By: Express
X-WAF-Blocked: url
ETag: W/"c6-n0KR/wCN9SFgXKAZficOHY67Sqc"
Vary: Accept-Encoding
Content-Length: 198

<body style="margin:0"><img style="width:100%;height:100%" alt="wait. thats illegal." src="https://i.kym-cdn.com/entries/icons/original/000/028/207/Screen_Shot_2019-01-17_at_4.22.43_PM.jpg"/></body>
```

## WAF bypass
Let's look at waf.js and figure out how to bypass any restrictions:
```
/**
 * A demo Web Application Firewall (WAF).
 */

const badStrings = [
    'X5O!P%@AP[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*',
    'woyouyizhixiaomaol',
    'conglaiyebuqi',
    'UNION',
    'SELECT',
    'SLEEP',
    'BENCHMARK',
    'alert(1)',
    '<script>',
    'onerror',
    'flag',
];

function checkRecursive(value) {
    // don't get bypassed by double-encoding
    const hasPercentEncoding = /%[a-fA-F0-9]{2}/.test(value);
    if (hasPercentEncoding) {
        return checkRecursive(decodeURIComponent(value));
    }

    // check for any bad word
    for (const badWord of badStrings) {
        if (value.includes(badWord)) {
            return true;
        }
    }
    return false;
}

function isBad(req) {
    const toCheck = ['url', 'body'];
    for (const key of toCheck) {
        const value = req[key];
        if (!value) {
            continue;
        }
        if (checkRecursive(String(value))) {            
            return key;
        }
    }
    return null;
}

// use named function for better logging
module.exports = async function waf(req, res, next) {
    const blockReason = isBad(req);
    if (blockReason !== null) {
        res.status(403);
        res.set('X-WAF-Blocked', blockReason);
        // internal redirect to /blocked so the app can show a custom 403 page
        req.url = '/blocked';
        return;
    }
};
```

We have 'flag' in the array of <i>badWord</i>s. Let's try some variations.
Perhaps sending 'FlAg' will do the trick:
![png](/images/hacklu-writeups/waf-bypass.png)


## Flag
```
flag{ca$h_ov3rfl0w}
```
