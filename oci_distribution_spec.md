# OCI Distribution Spec
> defines an API protocol to facilitate and standardize the distribution of content.


## 术语

- **Registry**: a service that handles the required APIs defined in this specification
- **Client**: a tool that communicates with Registries
- **Push**: the act of uploading Blobs and Manifests to a Registry
- **Pull**: the act of downloading Blobs and Manifests from a Registry
- **Blob**: the binary form of content that is stored by a Registry, addressable by a Digest
- **Manifest**: a JSON document which defines an Artifact. Manifests are defined under the OCI Image Spec <sup>[apdx-2](#appendix)</sup>
- **Config**: a blob referenced in the Manifest which contains Artifact metadata. Config is defined under the OCI Image Spec <sup>[apdx-4](#appendix)</sup>
- **Artifact**: one conceptual piece of content stored as Blobs with an accompanying Manifest containing a Config
- **Digest**: a unique identifier created from a cryptographic hash of a Blob's content. Digests are defined under the OCI Image Spec <sup>[apdx-3](#appendix)</sup>
- **Tag**: a custom, human-readable Manifest identifier


## Pull操作

The process of pulling an artifact centers around retrieving two components: the manifest and one or more blobs.

- ***检查是否在Registry存在***

In order to verify that a repository contains a given manifest or blob, make a `HEAD` request to a URL in the following form:

`/v2/<name>/manifests/<reference>` <sup>[end-3](#endpoints)</sup> (for manifests), or

`/v2/<name>/blobs/<digest>` <sup>[end-2](#endpoints)</sup> (for blobs).

A HEAD request to an existing blob or manifest URL MUST return `200 OK`. A successful response SHOULD contain the digest
of the uploaded blob in the header `Docker-Content-Digest`.

If the blob or manifest is not found in the registry, the response code MUST be `404 Not Found`.


- ***Pulling manifests***

To pull a manifest, perform a `GET` request to a URL in the following form:
`/v2/<name>/manifests/<reference>` <sup>[end-3](#endpoints)</sup>

`<name>` refers to the namespace of the repository. `<reference>` MUST be either (a) the digest of the manifest or (b) a tag.
The `<reference>` MUST NOT be in any other format. Throughout this document, `<name>` MUST match the following regular expression:

`[a-z0-9]+([._-][a-z0-9]+)*(/[a-z0-9]+([._-][a-z0-9]+)*)*`

Throughout this document, `<reference>` as a tag MUST be at most 128 characters in length and MUST match the following regular expression:

`[a-zA-Z0-9_][a-zA-Z0-9._-]{0,127}`

The client SHOULD include an `Accept` header indicating which manifest content types it supports.
In a successful response, the `Content-Type` header will indicate the type of the returned manifest.
For more information on the use of `Accept` headers and content negotiation, please see [Content Negotiation](./content-negotiation.md)

A GET request to an existing manifest URL MUST provide the expected manifest, with a response code that MUST be `200 OK`.
A successful response SHOULD contain the digest of the uploaded blob in the header `Docker-Content-Digest`.

The `Docker-Content-Digest` header, if present on the response, returns the canonical
digest of the uploaded blob which MAY differ from the provided digest. If the digest does differ, it MAY be the case that
the hashing algorithms used do not match. See [Content Digests](https://github.com/opencontainers/image-spec/blob/v1.0.1/descriptor.md#digests) <sup>[apdx-3](#appendix)</sup> for information on how to detect the hashing
algorithm in use. Most clients MAY ignore the value, but if it is used, the client MUST verify the value against the uploaded
blob data.

If the manifest is not found in the registry, the response code MUST be `404 Not Found`.

- ***Pulling blobs***

To pull a blob, perform a `GET` request to a URL in the following form:
`/v2/<name>/blobs/<digest>` <sup>[end-2](#endpoints)</sup>

`<name>` is the namespace of the repository, and `<digest>` is the blob's digest.

A GET request to an existing blob URL MUST provide the expected blob, with a response code that MUST be `200 OK`.
A successful response SHOULD contain the digest of the uploaded blob in the header `Docker-Content-Digest`. If present,
the value of this header MUST be a digest matching that of the response body.

If the blob is not found in the registry, the response code MUST be `404 Not Found`.



## Push操作

Pushing an artifact typically works in the opposite order as a pull: the blobs making up the artifact are uploaded first,
and the manifest last. Strictly speaking, content can be uploaded to the registry in any order, but a registry MAY reject
a manifest if it references blobs that are not yet uploaded, resulting in a `BLOB_UNKNOWN` error <sup>[code-1](#error-codes)</sup>.

- ***Pushing blobs***
有两个方法来push blobs: chunked or monolithic，这里只介绍一下chunks

#### Pushing a blob in chunks

A chunked blob upload is accomplished in three phases:
1. Obtain a session ID (upload URL) (`POST`)
2. Upload the chunks (`PATCH`)
3. Close the session (`PUT`)

For information on obtaining a session ID, reference the above section on pushing a blob monolithically via the `POST`/`PUT` method. The process remains unchanged for chunked upload, except that the post request MUST include the following header:

```
Content-Length: 0
```

Please reference the above section for restrictions on the `<location>`.

---
To upload a chunk, issue a `PATCH` request to a URL path in the following format, and with the following headers and body:

URL path: `<location>` <sup>[end-5](#endpoints)</sup>
```
Content-Type: application/octet-stream
Content-Range: <range>
Content-Length: <length>
```
```
<upload byte stream of chunk>
```

The `<location>` refers to the URL obtained from the preceding `POST` request.

The `<range>` refers to the byte range of the chunk, and MUST be inclusive on both ends. The first chunk's range MUST begin with `0`. It MUST match the following regular expression:

```regexp
^[0-9]+-[0-9]+$
```

The `<length>` is the content-length, in bytes, of the current chunk.

Each successful chunk upload MUST have a `202 Accepted` response code, and MUST have the following header:

```
Location <location>
```

Each consecutive chunk upload SHOULD use the `<location>` provided in the response to the previous chunk upload.

Chunks MUST be uploaded in order, with the first byte of a chunk being the last chunk's `<end-of-range>` plus one. If a chunk is uploaded out of order, the registry MUST respond with a `416 Requested Range Not Satisfiable` code.

The final chunk MAY be uploaded using a `PATCH` request or it MAY be uploaded in the closing `PUT` request. Regardless of how the final chunk is uploaded, the session MUST be closed with a `PUT` request.

---

To close the session, issue a `PUT` request to a url in the following format, and with the following headers (and optional body, depending on whether or not the final chunk was uploaded already via a `PATCH` request):

`<location>?digest=<digest>`
```
Content-Length: <length of chunk, if present>
Content-Range: <range of chunk, if present>
Content-Type: application/octet-stream <if chunk provided>
```
```
OPTIONAL: <final chunk byte stream>
```

The closing `PUT` request MUST include the `<digest>` of the whole blob (not the final chunk) as a query parameter.

The response to a successful closing of the session MUST be `201 Created`, and MUST contain the following header:
```
Location: <blob-location>
```

Here, `<blob-location>` is a pullable blob URL.

- ***Pushing Manifests***

To push a manifest, perform a `PUT` request to a path in the following format, and with the following headers
and body:
`/v2/<name>/manifests/<reference>` <sup>[end-7](#endpoints)</sup>
```
Content-Type: application/vnd.oci.image.manifest.v1+json
```
```
<manifest byte stream>
```

`<name>` is the namespace of the repository, and the `<reference>` MUST be either a) a digest or b) a tag.

The uploaded manifest MUST reference any blobs that make up the artifact. However, the list of blobs MAY
be empty. Upon a successful upload, the registry MUST return response code `201 Created`, and MUST have the
following header:

```
Location: <location>
```

The `<location>` is a pullable manifest URL.

An attempt to pull a nonexistent repository MUST return response code `404 Not Found`

