# AT Protocol 101

The Authenticated Transfer Protocol, aka atproto, is a federated protocol for large-scale distributed social applications. This notebook introduces how to interact with the data on the protocol, all of which is publicly available.

We'll use Python, without an SDK, so you can see how it works behind the scenes, but SDKs for multiple languages have been developed, including [Typescript](https://github.com/bluesky-social/atproto/tree/main/packages/api), [Python](https://atproto.blue/), [Dart](https://atprotodart.com/), and [Go](https://github.com/bluesky-social/indigo/tree/main).

For more in-depth information, check out:
- [Bluesky's documentation](https://docs.bsky.app)
- Skygaze's [starter repositories](https://github.com/skygaze-ai), which include bot and feed generator templates


```python
!pip install requests;

import pprint
import requests

pp = pprint.PrettyPrinter()
```

---
## Identity

Your DID, or Decentralized Identifier, is your universal ID across atproto. You can change your handle, but your DID remains the same. You can read more on DIDs <a href="https://atproto.com/specs/did">here</a>.

If you have a user's handle and you need to resolve it to their DID, you can use the `com.atproto.identity.resolveHandle` lexicon:


```python
handle = "foobar.bsky.social"  # Input your handle here

resolved_handle = requests.get(
    "https://bsky.social/xrpc/com.atproto.identity.resolveHandle",
    params={"handle": handle}
).json()

pp.pprint(resolved_handle)
```

    {'did': 'did:plc:yjmf4d5se4m23vgs7h5lyfgf'}


<div class="alert alert-block alert-info">
The HTTP API endpoint above used the atproto lexicon <code>com.atproto.identity.resolveHandle</code>

<br />
<br />
    
Lexicon is a schema system used to define RPC methods and record types. We'll use lexicons for the rest of this tutorial; you can read more about Lexicon <a href="https://atproto.com/guides/lexicon">here</a> and see the HTTP API reference for all atproto and Bluesky lexicons <a href="https://docs.bsky.app/docs/category/http-reference">here</a>.
</div>


```python
# You can get the DID for any user, like Paul:

handle = "pfrazee.com"  # Your handle can be your domain, too!

resolved_handle = requests.get(
    "https://bsky.social/xrpc/com.atproto.identity.resolveHandle",
    params={"handle": handle}
).json()

pp.pprint(resolved_handle)
```

    {'did': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd'}


---
## Data repository

A user's data is stored in their signed data repository (repo). Their repo holds the collections of all of their records, which include posts, comments, likes, follows, media blobs, etc. All currently active records are stored in the repository, and current repository contents are publicly available.

To access any user's data repository, you can use the `com.atproto.repo.describeRepo` lexicon:


```python
did = 'did:plc:ragtjsm2j2vknwkz3zp4oxrd'  # Get your DID from the example above

data_repository = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.describeRepo",
    params={"repo": did}
)

pp.pprint(data_repository.json())
```

    {'collections': ['app.bsky.actor.profile',
                     'app.bsky.feed.like',
                     'app.bsky.feed.post',
                     'app.bsky.feed.repost',
                     'app.bsky.feed.threadgate',
                     'app.bsky.graph.block',
                     'app.bsky.graph.follow',
                     'app.bsky.graph.list',
                     'app.bsky.graph.listitem'],
     'did': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd',
     'didDoc': {'@context': ['https://www.w3.org/ns/did/v1',
                             'https://w3id.org/security/multikey/v1',
                             'https://w3id.org/security/suites/secp256k1-2019/v1'],
                'alsoKnownAs': ['at://pfrazee.com'],
                'id': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd',
                'service': [{'id': '#atproto_pds',
                             'serviceEndpoint': 'https://morel.us-east.host.bsky.network',
                             'type': 'AtprotoPersonalDataServer'}],
                'verificationMethod': [{'controller': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd',
                                        'id': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd#atproto',
                                        'publicKeyMultibase': 'zQ3shbTzUCq5zuk7oSj5zaJndqWhjwGDaGuvBXpjg8C19qssW',
                                        'type': 'Multikey'}]},
     'handle': 'pfrazee.com',
     'handleIsCorrect': True}


### Records

The records in your data repository correspond exclusively to your "outbound" actions. For example, if you follow Paul or like one of Paul's posts, those records will be included in your data repository. However, if Paul were to follow you, there would be no record of that in your data repository; only Paul's.

<div class="alert alert-block alert-info">
The records in your data repository alone can answer the question "who do you follow?" because all of your follow records exist in your data repository. However, in order to answer the question "who follows you?" we would need to check every single user's repo across the entire network and see who has a follow record pointing to your DID. This is also known as having a "global view" of the network, and we'll cover how that works in a bit.
</div>

Posts, along with all other types of records, are identified using their <code>uri</code> and <code>cid</code>. 
- The `uri` can be thought of the path to that record, using the following format: `at://[did]/[record-type]/[record-key]`.
- The `cid` is the record's commit hash value and is used to cryptographically validate the record.
    
See an in-depth explanation of post records <a href="https://docs.bsky.app/docs/advanced-guides/posts">here</a>

The `collections` array in a user's data repo indicates all of the record types that that user has created. To retrieve all of a user's records of a given type, like all of their posts or all of their follows, you can use the `com.atproto.repo.listRecords` lexicon.


```python
my_posts = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.listRecords",
    params={
        "repo": did,
        "collection": "app.bsky.feed.post",
        "limit": 1
    }
)

pp.pprint(my_posts.json())
```

    {'cursor': '3klfrd6r7y32d',
     'records': [{'cid': 'bafyreiejy7dvbwpeafg5reqcl5utbae2thtgru6q33lzou7e3vip7eschq',
                  'uri': 'at://did:plc:p7flpn65bzf3kzjrp2xftshq/app.bsky.feed.post/3klfrd6r7y32d',
                  'value': {'$type': 'app.bsky.feed.post',
                            'createdAt': '2024-02-14T21:01:57.919Z',
                            'langs': ['en'],
                            'reply': {'parent': {'cid': 'bafyreiam25c245puhavkbp3xmxqpxnkfzlixvz5p46o2caxoa2m5f7gava',
                                                 'uri': 'at://did:plc:vwzwgnygau7ed7b7wt5ux7y2/app.bsky.feed.post/3kle5tg7iu325'},
                                      'root': {'cid': 'bafyreiam25c245puhavkbp3xmxqpxnkfzlixvz5p46o2caxoa2m5f7gava',
                                               'uri': 'at://did:plc:vwzwgnygau7ed7b7wt5ux7y2/app.bsky.feed.post/3kle5tg7iu325'}},
                            'text': '__innit__'}}]}


If you haven't posted on Bluesky yet, your `app.bsky.feed.post` collection will be an empty array. 

However, you can just as easily see any user's posts on the network. Let's get Paul's posts:


```python
paul_did = 'did:plc:ragtjsm2j2vknwkz3zp4oxrd'

paul_posts = requests.get(
    "https://bsky.social/xrpc/com.atproto.repo.listRecords",
    params={
        "repo": paul_did,
        "collection": "app.bsky.feed.post",
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
)

pp.pprint(paul_posts.json())
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
    }
)

pp.pprint(paul_follows.json())
```

    {'cursor': '3klstnu522s2o',
     'records': [{'cid': 'bafyreigdnspbjbytlqgsotaw7mdaqpj3m3at7stxlnocf6tgubgwhdf5la',
                  'uri': 'at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.graph.follow/3klstnu522s2o',
                  'value': {'$type': 'app.bsky.graph.follow',
                            'createdAt': '2024-02-20T01:48:19.526Z',
                            'subject': 'did:plc:uhfmcrnunkr3whev3momfchq'}}]}


---
## Authentication

Most of the data on the protocol is publicly available. However, to access data within the App View (as well as any of your own private data, like mutes) you must authenticate with your regular Bluesky credentials. You can protect your credentials by creating an [App Password](https://bsky.app/settings/app-passwords) for your project.

### Create a session
Once you authenticate, you receive a session object. This object includes your `accessJwt`, which is used to authenticate requests and is valid for 2 hours. Your `refreshJwt` lasts longer and is used only to update the session with a new access token. The session object also includes some basic account information, like your `did`, `handle`, and `email`. 


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

---
## App View

As mentioned, data repositories only include a user's "outbound" actions. In order to have a global view, like "which users liked this post?", an App View aggregates records across all data repositories on the network.

The `app.bsky.*` endpoints pull information from the global App View:


```python
# Get the likes for a given post
post_uri = "at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.feed.post/3klvirybw6c2b"

post_likes = requests.get(
    "https://bsky.social/xrpc/app.bsky.feed.getLikes",
    params={
        "uri": post_uri,
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
)

pp.pprint(post_likes.json())
```

    {'cursor': 'did:plc:mm6g3tgaumdqvfvlij526zz7',
     'likes': [{'actor': {'avatar': 'https://cdn.bsky.app/img/avatar/plain/did:plc:mm6g3tgaumdqvfvlij526zz7/bafkreicflmhdf27hqiker6u77k32yw72rbiqwsufnkj7ttzicloquejzt4@jpeg',
                          'description': '#Toronto Her/She. 🏳️\u200d⚧️ TRANS 👉🏻 '
                                         'https://gofund.me/fab678d0 or '
                                         'https://ko-fi.com/monicaellerose\n'
                                         '\n'
                                         'Socials 🔗 twitch.tv/monicaellerose \n'
                                         '\n'
                                         'User: #44620  ',
                          'did': 'did:plc:mm6g3tgaumdqvfvlij526zz7',
                          'displayName': 'Monica Rose',
                          'handle': 'monicarose.ca',
                          'indexedAt': '2024-02-23T00:32:58.929Z',
                          'labels': [],
                          'viewer': {'blockedBy': False,
                                     'followedBy': 'at://did:plc:mm6g3tgaumdqvfvlij526zz7/app.bsky.graph.follow/3kgu62s5sfm2w',
                                     'muted': False}},
                'createdAt': '2024-02-21T03:38:15.557Z',
                'indexedAt': '2024-02-21T03:38:15.557Z'}],
     'uri': 'at://did:plc:ragtjsm2j2vknwkz3zp4oxrd/app.bsky.feed.post/3klvirybw6c2b'}



```python
# Get the followers of a certain account
did = "did:plc:ragtjsm2j2vknwkz3zp4oxrd"

followers = requests.get(
    "https://bsky.social/xrpc/app.bsky.graph.getFollowers",
    params={
        "actor": did,
        "limit": 1
    },
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
)

pp.pprint(followers.json())
```

    {'cursor': '3km7wfn7of523',
     'followers': [{'did': 'did:plc:3gukbz6l2gmzbxjtguukh4tq',
                    'displayName': '',
                    'handle': 'rukmini.bsky.social',
                    'indexedAt': '2024-02-25T05:04:16.982Z',
                    'labels': [],
                    'viewer': {'blockedBy': False, 'muted': False}}],
     'subject': {'avatar': 'https://cdn.bsky.app/img/avatar/plain/did:plc:ragtjsm2j2vknwkz3zp4oxrd/bafkreihhpqdyntku66himwor5wlhtdo44hllmngj2ofmbqnm25bdm454wq@jpeg',
                 'description': 'Developer at Bluesky. The one who puts bugs in '
                                'this app. s/acc (shitpost accelerationist). Turbo '
                                'dude. He/him',
                 'did': 'did:plc:ragtjsm2j2vknwkz3zp4oxrd',
                 'displayName': 'Paul Frazee (hogfather arc) 🦋',
                 'handle': 'pfrazee.com',
                 'indexedAt': '2024-02-22T23:36:25.729Z',
                 'labels': [],
                 'viewer': {'blockedBy': False, 'muted': False}}}



```python
# Get your preferences (private -- can only view your own)
preferences = requests.get(
    "https://bsky.social/xrpc/app.bsky.actor.getPreferences",
    headers={"Authorization": f"Bearer {session['accessJwt']}"},
)

pp.pprint(preferences.json())
```

    {'preferences': [{'$type': 'app.bsky.actor.defs#adultContentPref',
                      'enabled': False},
                     {'$type': 'app.bsky.actor.defs#feedViewPref',
                      'feed': 'home',
                      'hideQuotePosts': False,
                      'hideReplies': False,
                      'hideRepliesByLikeCount': 2,
                      'hideRepliesByUnfollowed': False,
                      'hideReposts': False},
                     {'$type': 'app.bsky.actor.defs#savedFeedsPref',
                      'pinned': ['at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/whats-hot',
                                 'at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/with-friends',
                                 'at://did:plc:wqowuobffl66jv3kpsvo7ak4/app.bsky.feed.generator/the-algorithm'],
                      'saved': ['at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/bsky-team',
                                'at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/with-friends',
                                'at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/whats-hot',
                                'at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.generator/hot-classic',
                                'at://did:plc:wqowuobffl66jv3kpsvo7ak4/app.bsky.feed.generator/the-algorithm']}]}

