# Offline Inference Traces

This directory contains anonymized traces of offline LLM inference workloads collected from Alibaba Cloud Model Studio (百炼), as described in our SOSP 2026 paper, *"Batched in Back: Characterizing and Optimizing Offline LLM Inference in Production with ACDC"*.

## Data Structure

The traces are organized as follows:

```
offline/
├── trace_a.jsonl
├── trace_b.jsonl
├── trace_c.jsonl
├── trace_d.jsonl
├── trace_e.jsonl
└── trace_f.jsonl
```

- `trace_a.jsonl` – `trace_c.jsonl`: Content-hash traces: per-request input length plus prefix block hashes (and anonymized image IDs) for cache-reuse analysis
- `trace_d.jsonl` – `trace_f.jsonl`: Stat traces: per-request full input/output lengths and end-to-end execution duration

## Workloads

Each trace corresponds to one complete batch task submitted to the platform, i.e., the full set of requests in a single production batch job. Unlike online serving traces, batch tasks are submitted as a whole, so there is no per-request arrival timestamp; `request_id` preserves the order of requests in the submitted input file. Traces `a`/`b`/`c` expose the prefix structure of inputs (`hash_ids`), while traces `d`/`e`/`f` provide complete input/output lengths and request execution durations.

| Workload | File | Requests | Type |
| -------- | ---- | -------- | ---- |
| Dictionary translation | `trace_a.jsonl` | 51,429 | text |
| Text generation | `trace_b.jsonl` | 51,047 | text |
| Image captioning | `trace_c.jsonl` | 50,954 | multimodal |
| Generation-heavy (short in, long out) | `trace_d.jsonl` | 49,986 | text |
| Scoring / classification (long in, short out) | `trace_e.jsonl` | 49,999 | text |
| Multimodal balanced | `trace_f.jsonl` | 49,985 | multimodal |

All input and output lengths in these traces are in token unit.

## Trace Format

Traces are in JSONL format, one request per line.

**Content-hash traces (`trace_a`/`b`/`c`)** :

```jsonl
{
  "request_id": 0, 
  "type": "multimodal", 
  "input_length": 446, 
  "hash_ids": [655296, 655297, 655298, 655299, 655300, 655301, 655302, 655303, 655304, 655305, 655306, 655307, 655308, 655309, 655310, 655311, 655312, 655313, 655314, 655315, 655316, 655317, 655318, 655319, 655320, 655321, 655322, 655323], 
  "img_ids": ["img_00001"]
}
{
  "request_id": 1, 
  "type": "multimodal", 
  "input_length": 420, 
  "hash_ids": [655296, 655297, 655298, 655299, 655300, 655301, 655302, 655303, 655304, 655305, 655306, 655307, 655308, 655309, 655310, 655311, 655312, 655313, 655314, 655315, 655316, 655317, 655318, 655324, 655325, 655326, 655327], 
  "img_ids": ["img_00002"]
}
```

**Stat traces (`trace_d`/`e`/`f`)** :

```jsonl
{
  "request_id": 0, 
  "type": "text", 
  "input_length": 61, 
  "output_length": 2370, 
  "duration": 14000
}
{
  "request_id": 1, 
  "type": "text", 
  "input_length": 54, 
  "output_length": 2031, 
  "duration": 11630
}
```

### Fields

- `request_id`: Sequential ID of the request within its batch task, following the original order in the submitted input file.
- `type`: Request modality, either `text` or `multimodal`.
- `input_length`: Number of input (prompt) tokens.
- `output_length` (stat traces only): Number of generated output tokens.
- `duration` (stat traces only): End-to-end request completion time in milliseconds.
- `hash_ids` (content-hash traces only): Remapped hash IDs of the input's prefix blocks, with a block size of 16 tokens. For example, in the two content-hash requests above, both share the same first 23 blocks (hash IDs 655296–655318), i.e., a common 368-token prefix whose KV cache can be reused; they diverge from the 24th block onward. Hash IDs use one global namespace consistent across all content-hash traces (same ID = same content block), enabling both within- and cross-trace prefix-sharing analysis; in practice, cross-trace overlap is naturally rare since the traces are distinct task types.
- `img_ids` (content-hash traces only): Anonymized image IDs of the request. Within a task, identical image URLs are mapped to the same ID, enabling image/multimodal cache-reuse analysis.

## Privacy

These traces contain only anonymized token counts, remapped content hashes, and execution durations. They do not include any original text, real token IDs, or user identifiers. Prefix block hashes were generated using a salted hash and then globally remapped to consecutive integers, making recovery of the original content computationally infeasible. Image URLs have been replaced with anonymized IDs.
