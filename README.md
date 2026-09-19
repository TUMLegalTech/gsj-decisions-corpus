# gsj-decisions-corpus

Precomputed sentence-embedding vectors for **33,979 decisions of the German
Federal Court of Justice** (Bundesgerichtshof, BGH), published as release
assets so that any site running the GSJ legal-case workbench can import the
*same* vectors instead of deriving its own.

There is no code and no corpus text in this repository. Everything is in the
[releases](../../releases).

| | |
|---|---|
| decisions | 33,979 (court: `BGH`) |
| chunk vectors | 1,141,170 |
| dimensions | 768 |
| storage | `float16`, gzip, seven shards |
| total download | 1,616,286,454 bytes (1.505 GiB) + a 2,373-byte manifest |
| corpus fingerprint | `f93c956f1e6d2214b0073a388a750b1d6043265adc4e49a313ccf3b6a7438f72` |
| sidecar fingerprint | `4d3188c0e3624fd7f4227a9d22b071ad5fb17fa17b4ce95b701d922850f07533` |

---

## Where the source came from

The decisions are the XML records published by
**[rechtsprechung-im-internet.de](https://www.rechtsprechung-im-internet.de/)**
(the JURE dataset), the German federal justice administration's open
publication of decisions of the federal courts. That dataset is declared
`accessRights: public`.
Court decisions are *amtliche Werke* under **§ 5 UrhG** and carry no copyright,
so there is no licensing barrier to redistributing anything derived from them.

Nothing else went in. No commentary, no headnotes from commercial publishers,
no party data beyond what the published decision itself contains.

What is published **here** is not the decision text — it is the numeric
embedding of it. The text is available from the source above.

---

## Which model produced these vectors

A site running a **different model, revision, quantization, pooling rule or
dimension must not use these vectors.** The importer refuses such a mismatch by
name; this table is what it refuses against.

| field | value |
|---|---|
| `model` | `Snowflake/snowflake-arctic-embed-m-v2.0` |
| `revision` | `95c2741480856aa9666782eb4afe11959938017f` |
| runtime | ONNX Runtime, CPU, `onnx/model_int8.onnx` (dynamic INT8) |
| model-files digest (`manifest_sha256`) | `926bfa3a8788ddcc68cde3f706c92d7251781f768d0b01cf40066dd612d2bd42` |
| `dimensions` | 768 |
| `distance` | cosine |
| pooling | **CLS**, never mean |
| normalization | L2, emitted as float32 then stored as float16 |
| query prefix | `query: ` on **queries only**; documents get no prefix |
| token cap | 2,048 |
| batching at build time | 8 |
| `encoding` stamp | `snowflake-m-v2-onnx-int8-cls-l2-f32-768-max2048-query-v1` |

Chunking (the text these vectors are of): 500 characters with 80 characters of
overlap, `parser_version` 1, collection
`gsj_snowflake_m_v2_int8_2048_decisions`.

A query must be embedded by **that** encoder under **that** recipe — the
`query: ` prefix included — or it lands in a different part of the space and
the neighbours are meaningless.

---

## ⚠ These vectors cannot be reproduced by re-embedding

**This artifact *is* the corpus. Re-deriving it locally produces a different
store and different search rankings — with the same model, the same revision
and the same text.**

The reason is the quantization, and it is measured, not suspected. The pinned
ONNX export is *dynamically* quantized: 48 `DynamicQuantizeLinear` nodes feed
48 `MatMulInteger` nodes (two per projection block across 24 layers). Every
activation entering a `MatMulInteger` is quantized at run time on a scale
computed from the **min and max of the whole input tensor** — and that tensor
is `[batch, sequence, hidden]`. So the scale that quantizes one text's
activations depends on every *other* row in the batch and on every padded
position. The attention mask hides pad positions from attention; it does not
hide them from the min/max that sets the quantization step.

Measured on the build machine, 40 random chunks, cosine of the batched vector
against the same text encoded alone:

| how the same text was encoded | cosine vs. encoded alone (min / mean) |
|---|---|
| alone, twice | bit-identical |
| in a batch of 8 **identical copies** | 1.00000 / 1.00000 (40/40 bit-identical) |
| with 7 **random** batchmates — the build path | **0.95397 / 0.97477** |
| with one long batchmate (mostly padding) | 0.96531 / 0.97893 |
| a batch of ONE, hand-padded to ≥512 tokens | 0.97535 / 0.98110 |
| with 7 distinct batchmates of **identical token length — zero padding** | 0.96529 / 0.97312 |

The last row is the one that settles it: with no padding at all, the vectors
still move. Padding is the common way the tensor's range changes; it is not the
cause. The cause is the per-tensor activation scale itself.

Per `(text, batch)` the encoder is deterministic and portable — the same text
with the same batchmates gives bit-identical vectors on different machines of
the same class. Per *text* it is not. A fresh encode of one shard against these
vectors measured mean cosine 0.981, **min 0.907**.

### What follows from that

- **Two sites that both re-embed do not see the same neighbours.** Retrieval
  differs at the margins — rank ties, the tenth hit. Importing this artifact is
  the only way two sites agree.
- **The provenance contract proves identity of origin, not reproducibility.**
  The six embedding fields (`model`, `revision`, `manifest_sha256`,
  `dimensions`, `distance`, `encoding`) prove a store was made by *this*
  encoder under *this* recipe. They cannot prove two stores hold the same
  numbers, because they do not.
- **Verify identity, never equality.** Check that what you downloaded is what
  this release published (below). Do not expect a local re-embed to match it,
  and do not treat a mismatch against a re-embed as corruption here.
- If you want reproducible re-embedding you must change the encoder — batch
  size one with no padding, or a statically quantized export. Either produces a
  *different* vector space, and every existing store would have to be replaced
  rather than migrated. That is a different corpus generation and would be a
  different release here.

**If you read this far and re-embed anyway, you have been warned in the one
place you would look.**

---

## The files, and how to verify them

Each release carries **eight assets**: one manifest and seven shards.

| asset | bytes | vectors |
|---|---:|---:|
| `vectors.json` | 2,373 | — |
| `vectors-01.f16.gz` | 161,151,835 | 113,790 |
| `vectors-02.f16.gz` | 335,721,431 | 237,019 |
| `vectors-03.f16.gz` | 373,936,398 | 263,992 |
| `vectors-04.f16.gz` | 276,141,730 | 194,965 |
| `vectors-05.f16.gz` | 157,823,994 | 111,437 |
| `vectors-06.f16.gz` | 168,616,997 | 119,043 |
| `vectors-07.f16.gz` | 142,894,069 | 100,924 |

They are split because GitHub's per-asset limit is 2 GB and the set is already
float16 — there is no compression headroom left to buy. The largest shard is
19% of that ceiling, a failed download costs one shard rather than all of them,
and the corpus can grow without hitting a wall.

`vectors.json` is the manifest and the root of trust. Verify it first:

```sh
TAG=corpus-1.snowflake-m-v2-int8-768.f93c956f
BASE=https://github.com/TUMLegalTech/gsj-decisions-corpus/releases/download/$TAG

curl -fL -o vectors.json "$BASE/vectors.json"
sha256sum vectors.json
# 15bf3fb0530af8e2cb3941f0b35bf2454cc3d21bad55a467465f5939a502399c  vectors.json
```

That digest is published in the release notes and is the value you put in
`corpus.vectors_sha256`. Every other digest you need is *inside* the manifest,
so this one hash covers the whole 1.5 GiB set.

Then fetch and check each shard against the manifest:

```sh
python3 - <<'PY'
import hashlib, json, pathlib, urllib.request
TAG  = "corpus-1.snowflake-m-v2-int8-768.f93c956f"
BASE = f"https://github.com/TUMLegalTech/gsj-decisions-corpus/releases/download/{TAG}"
m = json.loads(pathlib.Path("vectors.json").read_bytes())
for s in m["shards"]:
    p = pathlib.Path(s["archive"])
    if not p.is_file() or p.stat().st_size != s["bytes"]:
        urllib.request.urlretrieve(f"{BASE}/{s['archive']}", p)
    d = hashlib.sha256(p.read_bytes()).hexdigest()
    print(f"{s['archive']} {'OK' if d == s['sha256'] else 'MISMATCH'}")
PY
```

The manifest also carries a `fingerprint` over its own canonical form, so it
cannot be edited without either breaking that self-check or being re-stamped
(which the importer refuses against the corpus it derived itself). Its fields:

`format`, `fingerprint`, `corpus_fingerprint`, `core_commit`, `parser_version`,
`chunk_chars`, `chunk_overlap`, `collection`, `dtype`, `codec`, `vectors`,
`embedding` (`model`, `revision`, `manifest_sha256`, `dimensions`, `distance`,
`encoding`), and `shards[]` — each with `id`, `archive`, `bytes`, `sha256`,
`vectors` and `ids_sha256`.

`ids_sha256` is the one that makes this more than a checksum: it binds a
shard's vectors to the exact chunk ids the importing site derives *itself* from
its own copy of the decision XML. A shard whose bytes are intact but which
describes different text is refused.

---

## How the installer consumes it

Point a site at the **manifest**; the shards are fetched as its siblings from
the same release.

```json
"corpus": {
  "vectors_url": "https://github.com/TUMLegalTech/gsj-decisions-corpus/releases/download/corpus-1.snowflake-m-v2-int8-768.f93c956f/vectors.json",
  "vectors_sha256": "15bf3fb0530af8e2cb3941f0b35bf2454cc3d21bad55a467465f5939a502399c"
}
```

The installer then:

1. downloads `vectors.json` and refuses unless it hashes to
   `vectors_sha256` — the manifest is never parsed before it is proven;
2. downloads each shard the manifest names and refuses unless it hashes to
   that shard's `sha256`;
3. stages the verified files into the corpus volume, checking size and digest
   again inside the cluster and publishing `vectors.json` **last**, so a
   half-arrived set is never mistaken for a complete one;
4. verifies the manifest a third time against the corpus the site parsed
   itself — `corpus_fingerprint`, `core_commit`, `parser_version`, every
   `embedding` field, the collection, the encoding, the shard coverage and the
   vector total — and each shard against the ids that site derived, before a
   single vector is stored.

Any disagreement stops the install and names the field that disagreed. Nothing
is ever re-stamped to make a mismatch go away, and a failed check is never
downgraded into a silent local re-embed.

### No egress?

Some clusters cannot reach the internet at all. Download the eight assets
somewhere that can, copy them to the installer host in one directory, make the
manifest private (`chmod 600 vectors.json` — the installer refuses a
world-readable protected input), and point the site at the manifest **on disk**
instead:

```json
"corpus": {
  "vectors_path": "/srv/gsj-vectors/vectors.json",
  "vectors_sha256": "15bf3fb0530af8e2cb3941f0b35bf2454cc3d21bad55a467465f5939a502399c"
}
```

Same artifact, same digests, same verification — only the transport differs.
This path never touches GitHub.

### If the download fails

The two failure modes are treated differently on purpose.

**An interrupted transfer resumes.** Partial bytes are kept and the next
attempt continues them with an HTTP range request; a server that does not
support ranges gets one fresh full attempt. Shards are cached by their own
digest, so a run that dies at shard six does not re-download the first five —
re-running fetches only what is missing.

**A complete download whose digest is wrong refuses, by name, and is not
silently re-fetched.** A transfer that finished and still hashes wrong is not a
transient error: it is a corrupted mirror or a substituted object, and quietly
retrying it would defeat the point of publishing digests at all. The run stops
and names the shard. Re-running does try again from scratch — but the operator
sees that it happened rather than waiting out a loop.

If a shard keeps failing, do not work around it. The published digests are
fixed. Fetch the eight assets over a route you trust and use the no-egress path
below.

---

## Release naming

```
corpus-1 . snowflake-m-v2-int8-768 . f93c956f
    │              │                    └── first 8 of the corpus fingerprint
    │              └── the embedding model generation and encoding
    └── the corpus generation (which decisions, which parser)
```

A new model means a new middle segment and a **different** corpus that must not
be confused with this one — its vectors are not interchangeable with these, and
mixing them is exactly the mistake the naming is there to prevent. A new
scrape, a new parser version or different chunking means a new leading segment.
The trailing fingerprint is the machine-checkable tie to the exact corpus the
importer verifies against.

## Updating

Because of the caveat above, sites cannot refresh a corpus by rebuilding it.
A corpus update is: a new release here, under a new tag, with a new manifest —
and every site re-importing from it.

## Licence

The decisions are *amtliche Werke* under § 5 UrhG and carry no copyright. This
repository's own content, and the vector data published in its releases, are
released under **CC0 1.0 Universal** — see [LICENSE](LICENSE). Attribution is
not required; naming rechtsprechung-im-internet.de as the source of the
underlying decisions is appreciated.
