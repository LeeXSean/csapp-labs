---
title: Proxy Lab
description: A 70/70 HTTP proxy — URI parsing, detached workers, and an approximate-LRU response cache.
---

# Proxy Lab · Building a Web Proxy

<p class="article-meta">Network programming <span class="dot">·</span> Score 70/70 <span class="dot">·</span> <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c">proxy.c</a></p>

!!! success "Verified locally"
    `NO_PROXY= no_proxy= ./driver.sh` → **70 / 70**: Basic 40/40, Concurrency 15/15, Cache 15/15.

The proxy accepts an HTTP request from a browser, opens a second connection to the origin, rewrites the request into the form the origin expects, relays the response byte-for-byte, and may cache the full response.

!!! abstract "The assignment"
    Build the proxy in three steps:

    1. A **sequential** proxy that forwards HTTP `GET` requests correctly.
    2. A **concurrent** proxy so one slow origin cannot stall every client.
    3. A **cache** for objects up to 100 KiB, with total cached payload bytes capped at 1,049,000 and eviction ordered by approximate LRU.

The cache is the only shared state. Parsing, request rewriting, origin I/O, and the temporary response copy stay private to one worker thread.

## One request path

``` text
                  GET http://host/path                 hit
   +---------+ ---------------------->  +-------+ ------>  +--------------+
   | browser |                          | proxy |          | object cache |
   +---------+ <----------------------  +-------+ <------  +--------------+
                   response (cached)         |  ^
                                        miss |  | response
                                             v  |
                                        +---------------+
                                        | origin server |
                                        +---------------+
```

---

## Part 1 · Forward one request correctly

A browser sends an **absolute URI** to the proxy; the origin server expects only the path. `doit` therefore splits the request line into `hostname`, `port`, and `path`, opens the origin connection, and rebuilds an HTTP/1.0 request.

### Parse the absolute URI in place { data-toc-label="URI parsing" }

The lab traffic uses URIs of the form `http://host[:port]/path`. `parse_uri` edits that string directly:

``` c
void parse_uri(char *uri, char *hostname, char *path, char *port) {
    char *head; char *tail;

    head = strstr(uri, "//");
    head += 2;
    if ((tail = strchr(head, ':')) == NULL) {
        strcpy(port, "80");
        tail = strchr(head, '/');
        *tail = '\0';
        strcpy(hostname, head);
    } else {
        *tail++ = '\0';
        strcpy(hostname, head);
        head = tail;
        tail = strchr(head, '/');
        *tail = '\0';
        strcpy(port, head);
    }
    *tail = '/';
    strcpy(path, tail); return;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L151-L175"><code>proxy.c</code> L151–L175</a> (full function) (reformatted to fit)</p>

A request like

``` text
http://www.example.com:8080/assets/logo.png
       └── hostname ──┘ └port┘└──── path ────┘
```

becomes:

``` text
hostname = www.example.com
port     = 8080
path     = /assets/logo.png
```

If the URI omits a port, the proxy supplies `80`. The cache key is then normalized as `hostname:port/path`, so `http://x/a` and `http://x:80/a` map to the same entry. This parser is intentionally lab-sized: it handles the handout's HTTP form, not general HTTPS or IPv6 URLs.

### Rebuild the request; do not tunnel the client's headers blindly { data-toc-label="Request headers" }

`doit` is the whole request path — accept a line, reject anything but `GET`, resolve the URI, serve a hit from the cache, and otherwise open the origin. The outgoing request always starts in one fixed shape:

``` c
void doit(int clientfd) {
    char buf[MAXLINE], request[2 * MAXLINE];
    char method[MAXLINE], uri[MAXLINE], version[MAXLINE];
    char hostname[MAXLINE], path[MAXLINE], port[MAXLINE];
    char key[3 * MAXLINE], object[MAX_OBJECT_SIZE]; rio_t client_rio, server_rio;
    int serverfd, cacheable = 1;
    ssize_t n;
    size_t object_size = 0;

    /* Read request line and headers */
    Rio_readinitb(&client_rio, clientfd);
    if (!Rio_readlineb(&client_rio, buf, MAXLINE))
        return;
    sscanf(buf, "%s %s %s", method, uri, version);
    if (strcasecmp(method, "GET")) {
        clienterror(clientfd, method, "501", "Not Implemented",
                    "Proxy does not implement this method");
        return;
    }
    parse_uri(uri, hostname, path, port);
    snprintf(key, sizeof(key), "%s:%s%s", hostname, port, path);
    if (cache_get(key, object, &object_size)) {
        Rio_writen(clientfd, object, object_size); return;
    }

    serverfd = Open_clientfd(hostname, port);

    /* Build proxy request headers */
    snprintf(request, sizeof(request), "GET %s HTTP/1.0\r\n", path);
    Rio_writen(serverfd, request, strlen(request));
    snprintf(request, sizeof(request), "Host: %s\r\n", hostname);
    Rio_writen(serverfd, request, strlen(request));
    Rio_writen(serverfd, user_agent_hdr, strlen(user_agent_hdr));
    Rio_writen(serverfd, "Connection: close\r\n",
              strlen("Connection: close\r\n"));
    Rio_writen(serverfd, "Proxy-Connection: close\r\n",
              strlen("Proxy-Connection: close\r\n"));

    /* Forward client request */
    forward_request(&client_rio, serverfd);

    /* Forward server request */
    Rio_readinitb(&server_rio, serverfd);
    while ((n = Rio_readnb(&server_rio, buf, MAXBUF)) > 0) {
        Rio_writen(clientfd, buf, n);
        if (cacheable && object_size + (size_t)n <= MAX_OBJECT_SIZE) {
            memcpy(object + object_size, buf, n);
            object_size += n;
        } else {
            cacheable = 0;
        }
    }
    Close(serverfd);
    if (cacheable)
        cache_put(key, object, object_size);
}
```

Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L67-L126"><code>proxy.c</code> L67–L126</a> (full function)
{: .code-source }

Then `forward_request` reads the remaining client headers and filters the four headers this proxy handles specially:

| Header from client | Proxy action | Why |
|---|---|---|
| `Host` | Discard the client's copy | The proxy already emitted its own `Host` line |
| `User-Agent` | Replace | The handout requires one fixed value |
| `Connection` | Replace with `close` | Make EOF delimit the origin response |
| `Proxy-Connection` | Replace with `close` | Keep the client-proxy hop short-lived |
| Everything else | Forward unchanged | Preserve end-to-end metadata |

The header loop stops on the blank line `\r\n`, and the proxy writes one final blank line to terminate the request. For this lab, forcing HTTP/1.0 plus `Connection: close` is the simplest correct rule: the origin's response ends at EOF, so the proxy does not need to manage persistent upstream connections.

### Relay bytes, not strings

The response path is binary-safe because it uses the byte count returned by `Rio_readnb` rather than a string length. That `n` matters: a JPEG, PDF, or executable can contain `\0` bytes long before the end of the response, so a `strlen`-based copy would truncate it.

---

## Part 2 · One detached worker per connection

A correct sequential proxy still fails the concurrency part of the lab. One slow origin would occupy the only control path and block every later client. The smallest fix is one detached thread per accepted connection:

``` c
int main(int argc, char **argv) {
    int listenfd, *connfd; socklen_t clientlen; struct sockaddr_storage clientaddr;
    pthread_t tid;

    /* Check command line args */
    if (argc != 2) {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(1);
    }
    Signal(SIGPIPE, SIG_IGN);
    listenfd = Open_listenfd(argv[1]);

    while (1) {
        clientlen = sizeof(clientaddr);
        connfd = Malloc(sizeof(int));
        *connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
        Pthread_create(&tid, NULL, thread, connfd);
    }
    return 0;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L30-L52"><code>proxy.c</code> L30–L52</a> (<code>main</code>, the accept loop) (reformatted to fit)</p>

``` c
void *thread(void *vargp) {
    Pthread_detach(pthread_self());

    int connfd = *((int *)vargp);
    Free(vargp);

    doit(connfd); Close(connfd);

    return NULL;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L54-L65"><code>proxy.c</code> L54–L65</a> (<code>thread</code>, one detached worker per connection) (reformatted to fit)</p>

The heap-allocated `connfd` is an ownership handoff, not shared state. The accept loop produces one descriptor, the worker consumes it, and the temporary heap cell disappears immediately. Passing `&connfd` from the loop stack would race as soon as the next `Accept` overwrote the same slot.

Two details matter here:

- `Pthread_detach` lets finished workers clean themselves up; the main loop never needs `join`.
- `Signal(SIGPIPE, SIG_IGN)` converts a write to a closed socket from a process-killing signal into an `EPIPE` error.

---

## Part 3 · Cache complete responses

The cache stores the entire origin response — headers and body together — under the normalized `hostname:port/path` key.

``` c
typedef struct cache_object {
    char *key; char *data; size_t size; unsigned long recent; struct cache_object *next;
} cache_object;

static cache_object *cache; static size_t cache_size; static unsigned long cache_clock;
static pthread_rwlock_t cache_lock = PTHREAD_RWLOCK_INITIALIZER;
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L7-L18"><code>proxy.c</code> L7–L18</a> (<code>cache_object</code> and its global state) (reformatted to fit)</p>

Each object owns exact-sized heap copies of its key and response bytes. That makes the cache budget precise: `cache_size` counts stored response bytes only, which is exactly what the handout limits.

### Cache invariants

| Invariant | How the code enforces it |
|---|---|
| One object ≤ `MAX_OBJECT_SIZE` | Stop accumulating once the response would exceed 100 KiB |
| Total payload ≤ `MAX_CACHE_SIZE` | Evict least-recently used entries until the new object fits |
| Binary-safe storage | Copy each chunk with `memcpy(..., n)` |
| One key stored once | Re-check for an existing key while holding the write lock |

On a miss, the worker relays the response to the client and simultaneously accumulates a **private** cache candidate in its stack buffer `object[MAX_OBJECT_SIZE]`, in the relay loop of `doit` above. Once the response grows past 100 KiB, the proxy stops extending the private copy but keeps forwarding the rest of the bytes to the client. Oversized responses still reach the client intact; only the cache copy is abandoned.

### Many readers, one writer { data-toc-label="Reader-writer locking" }

The cache is read-mostly, so the shared state sits behind one `pthread_rwlock_t`.

A cache hit proceeds in two phases:

``` text
read lock → find matching key → copy bytes to caller buffer → unlock
write lock → find key again → bump recent timestamp → unlock
```

The worker copies cached bytes into its own local buffer under the read lock, then releases the lock before writing to the client socket. A slow client therefore does not pin the cache.

The first lookup's `object` pointer cannot be reused after releasing the read lock: another writer may evict and free that entry before the write lock is acquired. The second phase therefore searches by `key` again before updating recency.

`recent` is mutable metadata, so `cache_get` reacquires the write side to update it:

``` c
int cache_get(const char *key, char *data, size_t *size) {
    cache_object *object;
    int found = 0;

    pthread_rwlock_rdlock(&cache_lock);
    for (object = cache; object; object = object->next) {
        if (!strcmp(object->key, key)) {
            memcpy(data, object->data, object->size);
            *size = object->size;
            found = 1;
            break;
        }
    }
    pthread_rwlock_unlock(&cache_lock);

    if (!found)
        return 0;

    pthread_rwlock_wrlock(&cache_lock);
    for (object = cache; object; object = object->next) {
        if (!strcmp(object->key, key)) {
            object->recent = ++cache_clock;
            break;
        }
    }
    pthread_rwlock_unlock(&cache_lock); return 1;
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L177-L205"><code>proxy.c</code> L177–L205</a> (full function) (reformatted to fit)</p>

The order is approximate LRU: `cache_clock` increases monotonically, and eviction removes the object with the smallest `recent` value.

Insertion and eviction stay entirely under the write lock:

``` c
void cache_put(const char *key, const char *data, size_t size) {
    cache_object *object;

    if (size == 0 || size > MAX_OBJECT_SIZE)
        return;

    pthread_rwlock_wrlock(&cache_lock);
    for (object = cache; object; object = object->next) {
        if (!strcmp(object->key, key)) {
            object->recent = ++cache_clock;
            pthread_rwlock_unlock(&cache_lock); return;
        }
    }

    while (cache && cache_size + size > MAX_CACHE_SIZE)
        cache_evict_lru();

    object = Malloc(sizeof(*object));
    object->key = Malloc(strlen(key) + 1);
    object->data = Malloc(size);
    strcpy(object->key, key); memcpy(object->data, data, size);
    object->size = size;
    object->recent = ++cache_clock;
    object->next = cache;
    cache = object;
    cache_size += size;
    pthread_rwlock_unlock(&cache_lock);
}
```

<p class="code-source">Source: <a href="https://github.com/LeeXSean/csapp-labs/blob/main/Proxy_Lab/proxylab-handout/proxy.c#L207-L237"><code>proxy.c</code> L207–L237</a> (full function) (reformatted to fit)</p>

`cache_evict_lru` linearly scans the linked list. Under a 1 MiB total-byte budget, that `O(n)` eviction cost is acceptable.

---

## Verification

| Check | Result |
|---|---|
| Basic text and binary forwarding | **40/40** |
| Slow origin does not block another client | **15/15** |
| Cached object survives origin shutdown | **15/15** |
| 16 simultaneous hits after origin shutdown | **16/16 identical** |
| Recently read entry survives later eviction pressure | **Pass** |
| Response larger than 100 KiB is not cached | **Pass** |
