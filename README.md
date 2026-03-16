
## What is VectorFaces?

[**Try the live demo**](https://www.elastic.co/demo-gallery/vector-search-celebrity-faces) with `start-local` on a single CPU.

![](doc/vectorfaces.gif)

VectorFaces is a real-time face recognition demo that searches through **1.2 million celebrity face embeddings** using Elasticsearch vector similarity search. Point your webcam or upload a photo, and it instantly finds matching faces using vector search. What is your celebrity lookalike?

The dataset is an excerpt from the [IMDB-WIKI](https://data.vision.ee.ethz.ch/cvl/rrothe/imdb-wiki/) dataset, which contains celebrity images from IMDb and Wikipedia. We generated 512-dimensional embeddings using InsightFace.

The demo leverages **four quantization methods in parallel** in four different indices with 318,526 vector embeddings each (totaling 1.2 million vectors), letting you compare their performance directly:

- **BBQ (Better Binary Quantization)** - Elasticsearch's binary quantization combining high recall with significant compression
- **DiskBBQ** - BBQ optimized for disk storage, reading vectors directly from disk during rescoring
- **int8** - 8-bit integer quantization reducing each dimension from 4 bytes to 1 byte
- **int4** - 4-bit quantization providing maximum compression (8× smaller than float32)

All quantization introduces some accuracy loss. In practice, **int8** typically needs little to no rescoring, **int4** often benefits from 1.5×–2× oversampling, and **BBQ** commonly requires 3×–5× oversampling for optimal recall. Each method balances accuracy, speed, memory footprint, and storage differently.

## What to Compare

Switch between the four indices using the sidebar and observe:
- **Search latency** - Response times for each quantization method
- **Relevance** - Do results differ between methods?
- **Resource usage** - Index sizes and memory in Kibana monitoring

### Understanding Search Parameters

Experiment with these parameters to balance speed and accuracy:

- **`k`**: Number of nearest neighbors to return globally across all shards. The final top `k` results are merged from all shards.
- **`num_candidates`**: Number of approximate neighbors to gather per shard before selecting the top `k`. Higher values improve recall and accuracy but increase latency. This is the primary knob for optimizing the latency/recall trade-off.
- **`size`**: Number of results that we bring back from Elasticsearch. Must be ≤ `k`.

For quantized indices, increasing `num_candidates` effectively oversamples the search space, which helps recover accuracy lost during quantization.

Inspect how much memory and disk each index is using:
```
GET faces-bbq_hnsw-10.15/_stats?filter_path=*.primaries.dense_vector

GET faces-disk_bbq-10.15/_stats?filter_path=*.primaries.dense_vector

GET faces-int8_hnsw-10.15/_stats?filter_path=*.primaries.dense_vector

GET faces-int4_hnsw-10.15/_stats?filter_path=*.primaries.dense_vector
```

## Changing more parameters

While `k` and `num_candidates` can be changed for all index types, some quantization techique have special flags we can tune. Take a look at the full list of settings [here](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector#dense-vector-index-options), some of them are applicable only at index time (like `m`, `ef_construction`), which would require a re-index, but some are applicable for query time as well, so we can have some extra fun by changing them.

Perhaps the most important one is `oversample`. It controls how many candidates are rescored using the original float vectors. With `oversample: 3.0`, the search retrieves `num_candidates` per shard using quantized vectors, then rescores the top `k * oversample` candidates (30 in this case) with the original vectors before returning the final top `k` results. Higher values improve accuracy at the cost of additional computation.

For BBQ we can change it in the index mapping at `rescore_vector.oversample`:

```json
PUT faces-bbq_hnsw-10.15/_mapping
{
  "properties": {
    "face_embeddings": {
      "type": "dense_vector",
      "dims": 512,
      "index": true,
      "similarity": "cosine",
      "index_options": {
        "type": "bbq_hnsw",
        "m": 16,
        "ef_construction": 100,
        "rescore_vector": {
          "oversample": 3
        }
      }
    }
  }
}
```

For DiskBBQ we can change `rescore_vector.oversample` and also `default_visit_percentage` which controls what percentage of clusters to visit during the search phase. Lower values trade accuracy for speed by visiting fewer clusters.

```json
PUT faces-disk_bbq-10.15/_mapping
{
    "properties": {
        "face_embeddings": {
          "type": "dense_vector",
          "dims": 512,
          "index": true,
          "similarity": "cosine",
          "index_options": {
            "type": "bbq_disk",
            "cluster_size": 384,
            "default_visit_percentage": 0,
            "rescore_vector": {
              "oversample": 3
            }
          }
        }
    }
}
```


## Setup

### 1. Create Your Elasticsearch Cluster

Create a cluster in [Elastic Cloud](https://cloud.elastic.co/home) or use [start-local](https://github.com/elastic/start-local). You will need just the Elasticsearch endpoint and the provided API key.

![Elastic Cloud Setup](doc/kibana-1.webp)

### 2. Create the Indices

Run `setup.py` to create the four indices plus one index for storing uploads:

```bash
pip install elasticsearch

python setup.py \
  --es_url https://your-deployment.us-west2.gcp.elastic-cloud.com:443 \
  --es_apikey your-api-key
```

This creates:
- `faces-bbq_hnsw-10.15`
- `faces-disk_bbq-10.15`
- `faces-int8_hnsw-10.15`
- `faces-int4_hnsw-10.15`
- `faces-bbq_hnsw-uploads` to store your picture uploads

### 3. Index the Data

Download the [index dump](https://storage.googleapis.com/faces-dataset/vectorfaces-index-dump.tar.gz) (318,526 celebrity face embeddings) and extract it.

**Option A: Index Each Method**

```bash
python index.py faces-bbq_hnsw-10.15 \
  --es_url https://your-deployment.us-west2.gcp.elastic-cloud.com:443 \
  --es_apikey your-api-key

python index.py faces-disk_bbq-10.15 \
  --es_url https://your-deployment.us-west2.gcp.elastic-cloud.com:443 \
  --es_apikey your-api-key

python index.py faces-int8_hnsw-10.15 \
  --es_url https://your-deployment.us-west2.gcp.elastic-cloud.com:443 \
  --es_apikey your-api-key

python index.py faces-int4_hnsw-10.15 \
  --es_url https://your-deployment.us-west2.gcp.elastic-cloud.com:443 \
  --es_apikey your-api-key
```

**Option B: Index Once, Reindex to Others**

Index BBQ first, verify all 318,526 documents are indexed, then reindex to the other indices.

Open **Kibana > Dev Tools** and run:

```
POST _reindex?requests_per_second=-1
{
  "source": {
    "index": "faces-bbq_hnsw-10.15"
  },
  "dest": {
    "index": "faces-int8_hnsw-10.15"
  }
}

POST _reindex?requests_per_second=-1
{
  "source": {
    "index": "faces-bbq_hnsw-10.15"
  },
  "dest": {
    "index": "faces-disk_bbq-10.15"
  }
}

POST _reindex?requests_per_second=-1
{
  "source": {
    "index": "faces-bbq_hnsw-10.15"
  },
  "dest": {
    "index": "faces-int4_hnsw-10.15"
  }
}
```

## Run the Demo

Configure `env.local` with your Elasticsearch credentials. Change only `ES_HOST` and `ES_API_KEY`; leave the rest as is:

```bash
ES_INDEX="faces"
ES_INDICES="faces-int4_hnsw-10.15,faces-int8_hnsw-10.15,faces-disk_bbq-10.15,faces-bbq_hnsw-10.15,faces-bbq_hnsw-10.15"
ES_UPLOADS_INDEX="faces-bbq_hnsw-uploads"
ES_HOST=https://es-host:port
ES_API_KEY=apikey
```

Start the application:

```bash
docker-compose up
```

Open `http://localhost:16700` and start testing!
