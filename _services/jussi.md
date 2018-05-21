---
title: jussi
position: 3
description: A reverse proxy that forwards json-rpc requests.
---

Jussi is a custom-built caching layer for use with `steemd` and other various services (such as [SBDS](/services/#services-sbds)).

#### Installation

To run `jussi` locally:

```bash
git clone https://github.com/steemit/jussi.git
cd jussi
docker build -t="$USER/jussi:$(git rev-parse --abbrev-ref HEAD)" .
docker run -itp 9000:8080 "$USER/jussi:$(git rev-parse --abbrev-ref HEAD)"
```

Try out your local configuration:

```bash
curl -s --data '{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":[8675309], "id":1}' http://localhost:9000
```

See: [Running Condenser, Jussi and a new service locally + adding feature flags to Condenser](https://steemit.com/steemdev/@maitland/running-condenser-jussi-and-a-new-service-locally-adding-feature-flags-to-condenser)

---

### Benefits of jussi

#### Time To Live

Jussi can be configured with various `TTL` (Time To Live) schemes. A `TTL` is an integer value in seconds.  Integers equal to or less than `0` have special meaning.  The current default:

| Upstream   | API                     | Method                          | Parameters         | TTL (seconds) |
|------------|-------------------------|---------------------------------|--------------------|---------------|
| `steemd`   | `login_api`             | _all_                           | _all_              | -1            |
| `steemd`   | `network_broadcast_api` | _all_                           | _all_              | -1            |
| `steemd`   | `follow_api`            | _all_                           | _all_              | 10            |
| `steemd`   | `market_history_api`    | _all_                           | _all_              | 1             |
| `steemd`   | `database_api`          | _all_                           | _all_              | 3             |
| `steemd`   | `database_api`          | `get_block`                     | _all_              | -2            |
| `steemd`   | `database_api`          | `get_block_header`              | _all_              | -2            |
| `steemd`   | `database_api`          | `get_content`                   | _all_              | 1             |
| `steemd`   | `database_api`          | `get_state`                     | _all_              | 1             |
| `steemd`   | `database_api`          | `get_state`                     | `'/trending'`      | 30            |
| `steemd`   | `database_api`          | `get_state`                     | `'trending'`       | 30            |
| `steemd`   | `database_api`          | `get_state`                     | `'/hot'`           | 30            |
| `steemd`   | `database_api`          | `get_state`                     | `'/welcome'`       | 30            |
| `steemd`   | `database_api`          | `get_state`                     | `'/promoted'`      | 30            |
| `steemd`   | `database_api`          | `get_state`                     | `'/created'`       | 10            |
| `steemd`   | `database_api`          | `get_dynamic_global_properties` | _all_              | 1             |
| `overseer` | _all_                   | _all_                           | _all_              | 5             |
| `conveyor` | _all_                   | _all_                           | _all_              | -1            |
| `sbds`     | _all_                   | _all_                           | _all_              | 3             |
| `hivemind` | _all_                   | _all_                           | _all_              | 3             |
| `yo`       | _all_                   | _all_                           | _all_              | 3             |

In this case, requests for `login_api` and `network_broadcast_api` have a `TTL` of `-1`, which means requests with those namespaces are not cached, whereas `follow_api` request have a `TTL` of `10` seconds.

Some methods and parameters have their own `TTL` that overrides the general default, like `database_api.get_block`, which overrides `database_api.*`.

##### Time to Live Special Meaning

* `0` won't expire
* `-1` won't be cached
* `-2` will be cached without expiration only if it is `irreversible` in terms of blockchain consensus

If you have a local copy of jussi (see: [Installation](#installation)), you can change these defaults by modifying `PROD_UPSTREAM_CONFIG.json`.

#### json-rpc batch

Normally, a request is made with a JSON Object (`{}`).  But jussi also supports batch requests, which is constructed with a JSON Array of Objects (`[{}]`).
  
For example, this would be a typical, non-batched JSON Object request that asks for a single block:

```bash
curl -s --data '{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":[1], "id":1}' https://api.steemit.com
```

```json
{
   "id":1,
   "jsonrpc":"2.0",
   "result":{
      "previous":"0000000000000000000000000000000000000000",
      "timestamp":"2016-03-24T16:05:00",
      "witness":"initminer",
      "transaction_merkle_root":"0000000000000000000000000000000000000000",
      "extensions":[

      ],
      "witness_signature":"204f8ad56a8f5cf722a02b035a61b500aa59b9519b2c33c77a80c0a714680a5a5a7a340d909d19996613c5e4ae92146b9add8a7a663eef37d837ef881477313043",
      "transactions":[

      ],
      "block_id":"0000000109833ce528d5bbfb3f6225b39ee10086",
      "signing_key":"STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX",
      "transaction_ids":[

      ]
   }
}
```

To request more than one block using the batch construct, wrap each call in a JSON Array, that asks for two blocks in one request:

```bash
curl -s --data '[{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":[1], "id":1},{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":[2], "id":2}]' https://api.steemit.com
```

```json
[
   {
      "id":1,
      "jsonrpc":"2.0",
      "result":{
         "previous":"0000000000000000000000000000000000000000",
         "timestamp":"2016-03-24T16:05:00",
         "witness":"initminer",
         "transaction_merkle_root":"0000000000000000000000000000000000000000",
         "extensions":[

         ],
         "witness_signature":"204f8ad56a8f5cf722a02b035a61b500aa59b9519b2c33c77a80c0a714680a5a5a7a340d909d19996613c5e4ae92146b9add8a7a663eef37d837ef881477313043",
         "transactions":[

         ],
         "block_id":"0000000109833ce528d5bbfb3f6225b39ee10086",
         "signing_key":"STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX",
         "transaction_ids":[

         ]
      }
   },
   {
      "id":2,
      "jsonrpc":"2.0",
      "result":{
         "previous":"0000000109833ce528d5bbfb3f6225b39ee10086",
         "timestamp":"2016-03-24T16:05:36",
         "witness":"initminer",
         "transaction_merkle_root":"0000000000000000000000000000000000000000",
         "extensions":[

         ],
         "witness_signature":"1f3e85ab301a600f391f11e859240f090a9404f8ebf0bf98df58eb17f455156e2d16e1dcfc621acb3a7acbedc86b6d2560fdd87ce5709e80fa333a2bbb92966df3",
         "transactions":[

         ],
         "block_id":"00000002ed04e3c3def0238f693931ee7eebbdf1",
         "signing_key":"STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX",
         "transaction_ids":[

         ]
      }
   }
]
```

Error responses are returned in the JSON Array response as well.  Notice the `"WRONG"` parameter in the second element.  The first block is returned as expected, the second one generates an error.

```bash
curl -s --data '[{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":[1], "id":1},{"jsonrpc":"2.0", "method":"condenser_api.get_block", "params":["WRONG"], "id":2}]' https://api.steemit.com
```

```json
[
   {
      "jsonrpc":"2.0",
      "result":{
         "previous":"0000000000000000000000000000000000000000",
         "timestamp":"2016-03-24T16:05:00",
         "witness":"initminer",
         "transaction_merkle_root":"0000000000000000000000000000000000000000",
         "extensions":[

         ],
         "witness_signature":"204f8ad56a8f5cf722a02b035a61b500aa59b9519b2c33c77a80c0a714680a5a5a7a340d909d19996613c5e4ae92146b9add8a7a663eef37d837ef881477313043",
         "transactions":[

         ],
         "block_id":"0000000109833ce528d5bbfb3f6225b39ee10086",
         "signing_key":"STM8GC13uCZbP44HzMLV6zPZGwVQ8Nt4Kji8PapsPiNq1BK153XTX",
         "transaction_ids":[

         ]
      },
      "id":1
   },
   {
      "jsonrpc":"2.0",
      "error":{
         "code":-32000,
         "message":"Parse Error:Couldn't parse uint64_t",
         "data":{
            "code":4,
            "name":"parse_error_exception",
            "message":"Parse Error",
            "stack":[
               {
                  "context":{
                     "level":"error",
                     "file":"string.cpp",
                     "line":113,
                     "method":"to_uint64",
                     "hostname":"",
                     "timestamp":"2018-05-21T18:02:41"
                  },
                  "format":"Couldn't parse uint64_t",
                  "data":{

                  }
               },
               {
                  "context":{
                     "level":"warn",
                     "file":"string.cpp",
                     "line":116,
                     "method":"to_uint64",
                     "hostname":"",
                     "timestamp":"2018-05-21T18:02:41"
                  },
                  "format":"",
                  "data":{
                     "i":"WRONG"
                  }
               },
               {
                  "context":{
                     "level":"warn",
                     "file":"variant.cpp",
                     "line":405,
                     "method":"as_uint64",
                     "hostname":"",
                     "timestamp":"2018-05-21T18:02:41"
                  },
                  "format":"",
                  "data":{
                     "*this":"WRONG"
                  }
               }
            ]
         }
      },
      "id":2
   }
]
```
**Note:** Batch requests are limited to a maximum of 50 request elements.

*See: [json-rpc batch specification](http://www.jsonrpc.org/specification#batch)*

---

*Latin*

>     jussi
> 
>     noun
> 
>     declension: 2nd declension
>     gender: neuter
> 
>     Definitions:
>       1. order, command, decree, ordinance, law