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

Typically, the first step in pulling an artifact is to retrieve the manifest.

### Pulling manifests文件

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

### Pulling blobs文件

To pull a blob, perform a `GET` request to a URL in the following form:
`/v2/<name>/blobs/<digest>` <sup>[end-2](#endpoints)</sup>

`<name>` is the namespace of the repository, and `<digest>` is the blob's digest.

A GET request to an existing blob URL MUST provide the expected blob, with a response code that MUST be `200 OK`.
A successful response SHOULD contain the digest of the uploaded blob in the header `Docker-Content-Digest`. If present,
the value of this header MUST be a digest matching that of the response body.

If the blob is not found in the registry, the response code MUST be `404 Not Found`.

### 检查是否内容在Registry存在
In order to verify that a repository contains a given manifest or blob, make a `HEAD` request to a URL in the following form:

`/v2/<name>/manifests/<reference>` <sup>[end-3](#endpoints)</sup> (for manifests), or

`/v2/<name>/blobs/<digest>` <sup>[end-2](#endpoints)</sup> (for blobs).

A HEAD request to an existing blob or manifest URL MUST return `200 OK`. A successful response SHOULD contain the digest
of the uploaded blob in the header `Docker-Content-Digest`.

If the blob or manifest is not found in the registry, the response code MUST be `404 Not Found`.