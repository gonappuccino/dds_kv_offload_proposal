# A Design Proposal for a Key-Value Store Leveraging the DDS Offload API

## 1. Project Overview

This document presents a simple design proposal to demonstrate the practical application of the Offload API introduced in "DDS: DPU-optimized Disaggregated Storage" (Zhang et al., VLDB 2024). To illustrate the core concepts, we specify a Key-Value (KV) store use case, grounding the theoretical API from Section 6.1 of the paper in a tangible application, similar to the case study in Section 9.2.

## 2. Key-Value Store Use Case 

This design assumes a simple KV store with the following characteristics:

* Storage Model: All Key-Value pairs are stored in a single, append-only log file (`log.dat`) on an SSD managed by the host server.

* `PUT(key, value)` Operation: This operation is always executed on the Host. It appends the new KV pair to the end of `log.dat` and returns the physical storage location (`file_id`, `offset`, `size`) of the new record.

* `GET(key)` Operation: This operation is the primary target for DPU offloading. To be processed by the DPU, the physical location corresponding to the `key` must already exist in the `Cache Table` residing in the DPU's memory.

## 3. DDS Offload API Implementation Design (Pseudo-code)

The following Python-style pseudo-code outlines how the DDS offload API could be implemented for our KV hypothetical store.

### 3.1. 'OffPred': Request Distribution

This function acts as the initial traffic director, deciding whether a request should be handled by the DPU or forwarded to the host.

```python
def OffPred(Msg, CacheTable):
    """
    Offload Predicate for a batched KV store.

    Given a network message potentially containing a batch of KV requests, this function inspects each request and classifies them into two lists:
    - `host_requests`: to be handled by the host CPU
    - `dpu_requests`: to be offloaded to the DPU

    Offload policy:
    - A GET request is offloaded only if the key exists in the DPU-side cache.
    - PUT and DELETE requests always go to the host to ensure data consistency and leverage the host's superior processing power for writes (as discussed in §2 of the paper).
    """

    host_requests = []
    dpu_requests = []

    # A single network message can contain multiple requests, a common optimization
    # mentioned in the DDS paper to improve throughput.
    individual_requests = parse_message_into_requests(Msg)

    for req in individual_requests:
        if req.op_type == 'GET':
            # GET requests are offloadable ONLY if its key exists in the DPU cache table.
            if req.key in CacheTable:
                # Cache hit: The DPU can serve the read directly.
                dpu_requests.append(req)
            else:
                # Cache miss: The request must be handled by the host.
                host_requests.append(req)
        
        elif req.op_type in ['PUT', 'DELETE']:
            # Writes and Deletes must be processed by the host for durability and consistency.
            host_requests.append(req)
        
        else:
            # Unknown or control-plane requests should default to the host for safety.
            host_requests.append(req)
    
    return (host_requests, dpu_requests)

```

### 3.2. Cache: Populating the DPU Cache Table

This function enables the 'cache-on-write' mechanism, which is fundamental to DDS. It populates the DPU's cache table with the necessary metadata after a write operation completes on the host.

```python
def Cache(WriteOp, ttl_ms=10000):
    """
    Processes a batch of write results to produce cache updates for the DPU.

    Each cache entry includes:
    - key: logical key from the KV store
    - location: (file_id, offset, size) tuple that points to physical data
    - version: optional version number or write timestamp to track freshness
    - ttl: time-to-live in milliseconds for managing cache eviction

    This aligns with the DDS paper's design principles in §6.1 and the KV store integration in §9.2.
    """

    cache_items = []

    for result in WriteOp:
        key = result.key

        location = (
            result.file_id,
            result.offset,
            result.size
        )

        # If the write result includes versioning info, use it.
        version = getattr(result, 'version', None)

        # Construct the cache entry as a dictionary for extensibility.
        cache_entry = {
            "location": location,
            "ttl_ms": ttl_ms
        }

        if version is not None:
            cache_entry["version"] = version
        
        # Append the final (key,entry) pair to the update list
        cache_items.append((key, cache_entry))
    
    return cache_items
```

### 3.3 OffFunc: Logical-to-Physical Request Translation

The Offload Function is the core of the execution engine. It translates a logical request into a physical file operation that the DPU's file service can execute.

```python
    def OffFunc(Req, CacheTable):
        """
        Translates a logical GET request into a physical file read operation using two-level mapping abstraction described in the DDS paper.

        - Level 1 (DPU-side): Logical key -> (file.id, offset, size)
        - Level 2 (File Service): (file.id, offset, size) -> disk block access

        Fallback to host if cache entry is missing (should not occur if OffPred works correctly).
        """
        
        key = getattr(Req, 'key', None)
        if key is None:
            # Handle malformed requests 
            log_warning("GET request missing key field")
            return ForwardToHost(Req)
        
        physical_address = CacheTable.get(key)
        
        if physical_address:
            file_id, offset, size = physical_address
            return CreateReadOp(file_id=file_id, offset=offset, size=size)
        else:
            # fallback
            log_warning(f"Cache miss in OffFunc for key: {key}. Forwarding to host.")
            return ForwardToHost(Req)
```

### 3.4 Invalidate: Maintaining Cache Coherency

This function is crucial for maintaining data consistency between the host and the DPU cache, implementing an 'invalidate-on-read' policy.

```python
    def Invalidate(ReadOp):
        """
        Evicts an entry from the DPU cache when a file read occurs on the host.

        This function implements the 'invalidate-on-read' policy, which is a precautionary measure described in the 
        DDS paper. It assumes that data read by the host might be modified in the host's memory, so its DPU cache entry 
        should be removed to prevent serving potentially stale data.
        """

        # for each read, return the keys to be removed.
        keys_to_invalidate = ReadOp.keys
        return keys_to_invalidate
```


## 4. Expected Benefits

By adopting this design, a KV store can leverage the DDS architecture to achieve significant performance and efficiency gains, as demonstrated in the source paper. The primary benefits include:


* Drastic CPU Cost Reduction: The host CPU consumption for read operations can be effectively eliminated, saving up to tens of CPU cores per storage server.
* Order-of-Magnitude Latency Improvement: By bypassing the host I/O stack, end-to-end read latency can be reduced by up to an order of magnitude, for instance, from 11ms down to 850µs at high throughput.
* Ease of Adoption: The integration requires minimal changes to the existing data system, with the original paper reporting that similar production integrations required only a few hundred lines of code.