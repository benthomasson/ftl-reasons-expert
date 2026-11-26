---
model: "openai:rits/deepseek-ai/DeepSeek-V4.1-Flash-1017"
date: 2026-11-26
---

# Topic: How `cluster_cache` is reused across `--exhaust` rounds and whether the "Loading embedding model..." message is printed once or per-call.

## How `cluster_cache` is reused across `--exhaust` rounds

`cluster_cache` is a stateful `ClusterCache` object, not a per-round value.

- `ClusterCache` is defined in `reasons/cluster.py` around the class docstring at line 35.
- Its constructor creates one `SentenceTransformer` model and initializes an internal cache:
  - `self.model = SentenceTransformer(model_name)`
  - `self._cache = {}`
- `ClusterCache.embed` caches embeddings by `(node_id, sha256(text).hexdigest()[:16])`.
  - It finds which belief ids/texts are not already in `self._cache`.
  - It calls `self.model.encode(...)` only for those uncached texts.
  - It stores the new embeddings and returns embeddings for all requested ids in sorted id order.

So across rounds, the cache reuses embeddings when the same `node_id` has the same text. New beliefs, or beliefs whose text changed, are encoded incrementally. KMeans clustering itself is still re-run each round.

In the CLI path:

- `reasons/cli.py:1486` shows the derive helper accepts `cluster_cache=None`.
- `reasons/cli.py:1522` forwards `cluster_cache` into the derive call.
- `reasons/cli.py:1688` initializes `cluster_cache = None`.
- `reasons/cli.py:1693` constructs `ClusterCache(...)`.
- `reasons/cli.py:1731` branches on `args.exhaust`.
- `reasons/cli.py:1740` passes the same `cluster_cache` into the exhaust-round call.
- `reasons/cli.py:1756` also passes it for the non-exhaust path.
- `reasons/derive.py:187` and `reasons/derive.py:442` accept `cluster_cache`, and `reasons/derive.py:215`, `:223`, and `:534` forward it into clustering helpers.

Therefore, in `--exhaust` mode, the same `ClusterCache` instance is threaded through each round. The expensive model instance is reused, and only uncached embeddings are recomputed.

## Is “Loading embedding model...” printed once or per-call?

It is printed once per `cmd_derive` invocation when the cache-construction block runs, not once per `--exhaust` round.

The only print for `"Loading embedding model..."` is at:

- `reasons/cli.py:1692`: `print("Loading embedding model...", file=sys.stderr)`

It is immediately followed by the single construction site:

- `reasons/cli.py:1693`: `cluster_cache = ClusterCache(`

That construction happens before the `if args.exhaust:` branch at `reasons/cli.py:1731`, and the exhaust loop passes the already-created object at `reasons/cli.py:1740`. `ClusterCache.embed` itself prints nothing. So repeated rounds do not reprint the message.

Caveat: if a cluster helper is called with `cache=None`, it will construct a new `ClusterCache` internally per call. That reloads the model but does not print the CLI message. The CLI exhaust path avoids that by passing the non-`None` `cluster_cache`.

The observation set does not show the exact condition guarding lines `1688–1693`, so the precise flag combination that triggers cache creation is not visible here. The ordering clearly shows a single construction site before the exhaust loop.

## Topics to Explore

- [file] `reasons/cli.py` — Inspect `cmd_derive` around lines 1680–1760 to see the exact guard around cache creation and the exhaust-round loop.
- [function] `reasons/cluster.py:ClusterCache.embed` — Shows the content-addressed cache key and incremental encoding behavior.
- [function] `reasons/cluster.py:cluster_beliefs_intra` — Uses `round_num % k` to rotate cluster focus while reusing the same embedding cache.
- [file] `reasons/derive.py` — Trace how `cluster_cache` flows from derive entry points into clustering helpers.
- [file] `tests/test_cluster.py` — Contains reuse and incremental-cache tests around lines 72 and 83.

## Beliefs

- `cluster-cache-keys-on-node-id-and-text-hash` — `ClusterCache.embed` caches each embedding under `(node_id, sha256(text)[:16])`.
- `cluster-cache-reuses-model-instance` — `ClusterCache.__init__` creates one `SentenceTransformer`, and `embed` only calls `model.encode` for uncached texts.
- `exhaust-passes-same-cache-to-each-round` — `reasons/cli.py` creates `cluster_cache` at line 1693 and passes it into each exhaust round at line 1740.
- `loading-message-has-one-print-site` — `"Loading embedding model..."` is printed only at `reasons/cli.py:1692`, adjacent to the single `ClusterCache` construction.
- `kmeans-is-not-cached` — `cluster_beliefs` and `cluster_beliefs_intra` run `KMeans.fit_predict` every call even when embeddings are reused from `ClusterCache`.
