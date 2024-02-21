# AT Protocol Crash Course

The Authenticated Transfer Protocol, aka atproto, is a federated protocol for large-scale distributed social applications. [What this notebook is- this notebook will introduce how to interact with the data on the protocol, all of which is publicly available].

We'll use Python, without an SDK, so you can see how it works behind the scenes, but SDKs for multiple languages have been developed, including [Typescript](https://github.com/bluesky-social/atproto/tree/main/packages/api), [Python](https://atproto.blue/), [Dart](https://atprotodart.com/), and [Go](https://github.com/bluesky-social/indigo/tree/main).

[If you're looking for further etc. etc.- Link to bot and feed generator templates]

---
## Identity

In order to access any data on the protocol, you need to be authenticated. You can sign in with your regular Bluesky credentials, and you can protect your credentials by creating an [App Password](https://bsky.app/settings/app-passwords) for your project.

### Create a session
Once you authenticate, you receive a session object. This object includes your `accessJwt`, which is used to authenticate requests and is valid for 2 hours. Your `refreshJwt` lasts longer and is used only to update the session with a new access token. The session object also includes some basic account information, like your `did`, `handle`, and `email`. 

<div class="alert alert-block alert-info">
Your DID, or Decentralized Identifier, is your universal ID across atproto. You can read more on them <a href="https://atproto.com/specs/did">here</a>.
</div>


```python
# Create a Bluesky account at bsky.app
bluesky_username = "<username>"
bluesky_password = "<password>"

session = requests.post(
    "https://bsky.social/xrpc/com.atproto.server.createSession",
    json={"identifier": bluesky_username, "password": bluesky_password},
).json()

pp.pprint(session)
```

    {'accessJwt': '...',
     'did': 'did:plc:wqowuobffl66jv3kpsvo7ak4',
     'didDoc': {'@context': ['https://www.w3.org/ns/did/v1',
                             'https://w3id.org/security/multikey/v1',
                             'https://w3id.org/security/suites/secp256k1-2019/v1'],
                'alsoKnownAs': ['at://skygaze.io'],
                'id': 'did:plc:wqowuobffl66jv3kpsvo7ak4',
                'service': [{'id': '#atproto_pds',
                             'serviceEndpoint': 'https://inkcap.us-east.host.bsky.network',
                             'type': 'AtprotoPersonalDataServer'}],
                'verificationMethod': [...]},
     'email': 'skygazefeeds@gmail.com',
     'emailConfirmed': True,
     'handle': 'skygaze.io',
     'refreshJwt': '...'}


<div class="alert alert-block alert-info">
The HTTP API endpoint above used the atproto lexicon <code>com.atproto.server.createSession</code>

<br />
<br />
    
Lexicon is a schema system used to define RPC methods and record types. We'll use lexicons for the rest of this tutorial; you can read more about Lexicon <a href="https://atproto.com/guides/lexicon">here</a> and see the HTTP API reference for all atproto and Bluesky lexicons <a href="https://docs.bsky.app/docs/category/http-reference">here</a>.
</div>

### Handle resolution

If you have a user's handle and you need to resolve it to their DID, you can use the `com.atproto.identity.resolveHandle` lexicon:


```python
paul_handle = "pfrazee.com"  # Your handle can be your own domain, too!

paul_identity = requests.get(
    "https://bsky.social/xrpc/com.atproto.identity.resolveHandle",
    params={"handle": paul_handle},
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
).json()

pp.pprint(paul_identity)
```

    {'did': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd'}


---
## Data repository

A user's data is stored in their signed data repository (repo). Their repo holds the collections of all of their records, which include posts, comments, likes, follows, media blobs, etc.

To access any user's data repository, you can use the `com.atproto.repo.describeRepo` lexicon:


```python
did = session["did"]

data_repository = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.describeRepo",
    params={"repo": did},
    headers={"Authorization": session["accessJwt"]},
).json()

pp.pprint(data_repository)
```

    {'collections': ['app.bsky.actor.profile',
                     'app.bsky.feed.generator',
                     'app.bsky.feed.like',
                     'app.bsky.feed.post',
                     'app.bsky.feed.repost',
                     'app.bsky.graph.block',
                     'app.bsky.graph.follow'],
     'did': 'did:plc:wqowuobffl66jv3kpsvo7ak4',
     'didDoc': {'@context': ['https://www.w3.org/ns/did/v1',
                             'https://w3id.org/security/multikey/v1',
                             'https://w3id.org/security/suites/secp256k1-2019/v1'],
                'alsoKnownAs': ['at://skygaze.io'],
                'id': 'did:plc:wqowuobffl66jv3kpsvo7ak4',
                'service': [{'id': '#atproto_pds',
                             'serviceEndpoint': 'https://inkcap.us-east.host.bsky.network',
                             'type': 'AtprotoPersonalDataServer'}],
                'verificationMethod': [{'controller': 'did:plc:wqowuobffl66jv3kpsvo7ak4',
                                        'id': 'did:plc:wqowuobffl66jv3kpsvo7ak4#atproto',
                                        'publicKeyMultibase': 'zQ3shgVBaFgscLHqXhN339HBuZ2WMbMeLE5GGtUBKxkxCA1C5',
                                        'type': 'Multikey'}]},
     'handle': 'skygaze.io',
     'handleIsCorrect': True}


### Records

A user's data repository stores all of their records, which represent any public user action: posts, likes, reposts, blocks, follows etc. All currently active records are stored in the repository, and current repository contents are publicly available.

<div class="alert alert-block alert-info">
All of the records stored in a user's repo are "outbound" i.e. they only represent actions that the user has performed. For example, the records in Paul's data repository alone can answer the question "who does Paul follow?" because all of his follow records exist in his data repository. However, in order to answer the question "who follows Paul?" we would need to check every single user's repo across the entire network and see who has a follow record for Paul. This is also known as having a "global view" of the network, and we'll cover how that works in a bit.
</div>

The `collections` array in a user's data repo indicates all of the record types that that user has created. To retrieve all of a user's records of a given type, like all of their posts or all of their follows, you can use the `com.atproto.repo.listRecords` lexicon.


```python
did = session["did"]

my_posts = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.listRecords",
    params={
        "repo": did,
        "collection": "app.bsky.feed.post",
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
).json()

pp.pprint(my_posts)
```

    {'cursor': '3klqk3u246c2e',
     'records': [{'cid': 'bafyreiht7rc6r5ivfx2hejnhf6hilnjv2vprkc4sfgb2peuzpeb2krxzdq',
                  'uri': 'at://did:plc:wqowuobffl66jv3kpsvo7ak4/app.bsky.feed.post/3klqk3u246c2e',
                  'value': {'$type': 'app.bsky.feed.post',
                            'createdAt': '2024-02-19T03:51:52.796Z',
                            'embed': {'$type': 'app.bsky.embed.external',
                                      'external': {'description': 'Last week, '
                                                                  'Bluesky opened '
                                                                  'its doors, and '
                                                                  'now, it has ~5M '
                                                                  'users! In '
                                                                  'celebration of '
                                                                  'Bluesky’s '
                                                                  'public launch, '
                                                                  'we’re hosting a '
                                                                  'hackathon at '
                                                                  'the YC office '
                                                                  'in SF judged by '
                                                                  'Bluesky CEO Jay '
                                                                  'Graber and '
                                                                  'several...',
                                                   'thumb': {'$type': 'blob',
                                                             'mimeType': 'image/jpeg',
                                                             'ref': {'$link': 'bafkreib4pc6sjnhogtaoluipc33ewpsxdvd34r42slma5d75d6x6sm6the'},
                                                             'size': 954165},
                                                   'title': 'RSVP to Bluesky AI '
                                                            'Hackathon | Partiful',
                                                   'uri': 'https://partiful.com/e/AiscT5PsNTIrxNq58M5q'}},
                            'langs': ['en'],
                            'text': 'Bluesky hackathon coming up in SF on the 25th '
                                    '🥳'}}]}


If you haven't posted on Bluesky yet, your `app.bsky.feed.post` collection will be an empty array. 

However, you can just as easily see any user's posts on the network. Let's get Paul's posts:


```python
paul_did = paul_identity['did']

paul_posts = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.listRecords",
    params={
        "repo": paul_did,
        "collection": "app.bsky.feed.post",
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
).json()

pp.pprint(paul_posts)
```

    {'cursor': '3klvirybw6c2b',
     'records': [{'cid': 'bafyreidnqpmg6knx6o3mkbonjeasne75hvmfnpevl4gyfcjohmnamwtkwm',
                  'uri': 'at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.feed.post/3klvirybw6c2b',
                  'value': {'$type': 'app.bsky.feed.post',
                            'createdAt': '2024-02-21T03:11:46.550Z',
                            'langs': ['en'],
                            'reply': {'parent': {'cid': 'bafyreienacnu743ohgg5tw36q77gbd4dpr74cwvytyser53odpjtp42kdi',
                                                 'uri': 'at://did:plc:i3ycqqigla52z3pc6b24w3ku/app.bsky.feed.post/3klvipy765k2o'},
                                      'root': {'cid': 'bafyreihkpwdhddycutoqel6sya6vbpgjmgkq66kykocsctvbzvlttcwoo4',
                                               'uri': 'at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.feed.post/3klutu6krgk2o'}},
                            'text': 'oh lol'}}]}


<div class="alert alert-block alert-info">
Posts, along with all other types of records, are identified using their <code>uri</code> and <code>cid</code>. 
    <ul>
    <li>The <code>uri</code> can be thought of the path to that record, using the following format: <code>at://[did]/[record-type]/[record-key]</code>. </li>
    <li>The `cid` is the record's commit hash value and is used to cryptographically validate the record. </li>
    </ul>
    
See an in-depth explanation of post records <a href="https://docs.bsky.app/docs/advanced-guides/posts">here</a>
</div>

### Scrolling
A maximum of 100 records can be returned per request. The `cursor` can be used to scroll through all of the records in a given collection, regardless of its size:


```python
all_paul_posts = []

more_posts = True
cursor = ''

while more_posts:
    paul_posts_batch = requests.get(
        "https://bsky.social/xrpc/com.atproto.repo.listRecords",
        params={
            "repo": paul_did,
            "collection": "app.bsky.actor.post",
            "cursor": cursor
        },
        headers={"Authorization": f"Bearer {session['accessJwt']}"},
    ).json()

    all_paul_posts.extend(paul_posts_batch['records'])

    if 'cursor' in paul_posts_batch:
        cursor = paul_posts_batch['cursor']
    else:
        more_posts = False

    # Paul has a lot of posts 😅
    if len(all_paul_posts) > 500:
        more_posts = False
```

### Other records

Just like posts, you can access other record type for any user, like:

- `app.bsky.actor.profile`
- `app.bsky.feed.like`
- `app.bsky.feed.repost`
- `app.bsky.graph.block`
- `app.bsky.graph.follow`

Check out all of the other atproto lexicons [here](https://docs.bsky.app/docs/category/http-reference). 


```python
paul_follows = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.listRecords",
    params={
        "repo": paul_did,
        "collection": "app.bsky.graph.follow",
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
).json()

pp.pprint(paul_follows)
```

    {'cursor': '3klstnu522s2o',
     'records': [{'cid': 'bafyreigdnspbjbytlqgsotaw7mdaqpj3m3at7stxlnocf6tgubgwhdf5la',
                  'uri': 'at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.graph.follow/3klstnu522s2o',
                  'value': {'$type': 'app.bsky.graph.follow',
                            'createdAt': '2024-02-20T01:48:19.526Z',
                            'subject': 'did:plc:uhfmcrnunkr3whev3momfchq'}}]}


## App View

As mentioned, data repositories only include a user's "outbound" actions. In order to have a global view, like "which users liked this post?", an App View aggregates records across all data repositories on the network.

The `app.bsky.*` endpoints 


```python
# Example of calling https://bsky.social/xrpc/app.bsky.feed.getLikes
```


```python
# Example of calling https://bsky.social/xrpc/app.bsky.graph.getFollowers
```


```python
# Example of calling https://bsky.social/xrpc/app.bsky.actor.getProfile (gives you more info than their profile record)
```

---
## Feed Generators

[Link to feed generator examples in Python and Typescript]

---
## Firehose

How does the App View index all of the content from the data repositories? It uses the global firehose, or Relay, which streams updates from each data repository.

[Link to firehose examples in Python and Typescript]


```python

```


<style>
.output {
    max-height: 150px; /* Adjust the height as needed */
    overflow-y: auto; /* Enables vertical scrolling */
}
</style>




```python

```
