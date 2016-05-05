+++
categories = ["general"]
date = "2016-05-05T11:59:13+09:00"
tags = ["document"]
title = "Specification"

+++

## Consul Kv Datastore

**Fireap** stores its working data on _Consul Kv_.  
All kinds of data appear in the table below:

| path                                    | data type  | description          |
|:--------------------------------------- | ---------- |:-------------------- |
| `fireap/<app>/lock`                     | String     | Task execution lock to avoid multiple execution at the same time |
| `fireap/<app>/nodes/<node>/version`     | String     | Last executed task version on `<node>` |
| `fireap/<app>/nodes/<node>/semaphore`   | Integer    | How many _subscribers_ are acceptable for `<node>` |
| `fireap/<app>/nodes/<node>/update_info` | JSON     | Includes date when last task execution completed and from which host `<node>` "pulled" the update |

Here are some supplementary information:

- **Fireap** does not memorize version history but only last executed version.
- When a _subscriber_ "pulls" updates from a remote node, it decrements the
`semaphore` number of the remote node. This reduction is done by Consul HTTP API
with `?cas=` parameter to safely update the value.
- Format of `update_info` goes like below:
  - `{"updated_at":"2016-03-20 22:24:09 +0900","remote_node":"xxx01"}`.
  - This data is only for `monitor` use.

## References

- [Key/Value store (HTTP) - Consul by HashiCorp](https://www.consul.io/docs/agent/http/kv.html)
