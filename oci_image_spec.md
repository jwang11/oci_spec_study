# OCI Image Spec
> This specification defines an OCI Image, consisting of a manifest, an image index (optional), a set of filesystem layers, and a configuration.
> The goal of this specification is to enable the creation of interoperable tools for building, transporting, and preparing a container image to run.

从上层划分，spec包括了
* [Image Manifest](manifest.md) - 描述镜像组成元素的文档
* [Image Index](image-index.md) - 按照平台分类的Manifest列表
* [Filesystem Layer](layer.md) - 容器文件系统的变化集合（changeset）。
* [Image Configuration](config.md) - 定义了容器的层顺序，以及生成runtime bundle的镜像配置。
* [Descriptor](descriptor.md) - 描述类型，元数据以及内容地址的结构


## 镜像清单（Image Manifest）


### 属性描述

- **`schemaVersion`** *int*

  This REQUIRED property specifies the image manifest schema version.
  For this version of the specification, this MUST be `2` to ensure backward compatibility with older versions of Docker. The value of this field will not change. This field MAY be removed in a future version of the specification.

- **`mediaType`** *string*

  This property is *reserved* for use, to [maintain compatibility](media-types.md#compatibility-matrix).
  When used, this field contains the media type of this document, which differs from the [descriptor](descriptor.md#properties) use of `mediaType`.

- **`config`** *[descriptor](descriptor.md)*

    This REQUIRED property references a configuration object for a container, by digest.
    Beyond the [descriptor requirements](descriptor.md#properties), the value has the following additional restrictions:

    - **`mediaType`** *string*

        This [descriptor property](descriptor.md#properties) has additional restrictions for `config`.
        Implementations MUST support at least the following media types:

        - [`application/vnd.oci.image.config.v1+json`](config.md)

        Manifests concerned with portability SHOULD use one of the above media types.

- **`layers`** *array of objects*

    Each item in the array MUST be a [descriptor](descriptor.md).
    The array MUST have the base layer at index 0.
    Subsequent layers MUST then follow in stack order (i.e. from `layers[0]` to `layers[len(layers)-1]`).
    The final filesystem layout MUST match the result of [applying](layer.md#applying-changesets) the layers to an empty directory.
    The [ownership, mode, and other attributes](layer.md#file-attributes) of the initial empty directory are unspecified.

    Beyond the [descriptor requirements](descriptor.md#properties), the value has the following additional restrictions:

    - **`mediaType`** *string*

        This [descriptor property](descriptor.md#properties) has additional restrictions for `layers[]`.
        Implementations MUST support at least the following media types:

        - [`application/vnd.oci.image.layer.v1.tar`](layer.md)
        - [`application/vnd.oci.image.layer.v1.tar+gzip`](layer.md#gzip-media-types)
        - [`application/vnd.oci.image.layer.nondistributable.v1.tar`](layer.md#non-distributable-layers)
        - [`application/vnd.oci.image.layer.nondistributable.v1.tar+gzip`](layer.md#gzip-media-types)

        Manifests concerned with portability SHOULD use one of the above media types.
        An encountered `mediaType` that is unknown to the implementation MUST be ignored.


        Entries in this field will frequently use the `+gzip` types.

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for the image manifest.
    This OPTIONAL property MUST use the [annotation rules](annotations.md#rules).

    See [Pre-Defined Annotation Keys](annotations.md#pre-defined-annotation-keys).


*Example showing an image manifest:*
```json,title=Manifest&mediatype=application/vnd.oci.image.manifest.v1%2Bjson
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

## 镜像索引（Manifest Index）

### 属性描述

- **`schemaVersion`** *int*

  This REQUIRED property specifies the image manifest schema version.
  For this version of the specification, this MUST be `2` to ensure backward compatibility with older versions of Docker.
  The value of this field will not change.
  This field MAY be removed in a future version of the specification.

- **`mediaType`** *string*

  This property is *reserved* for use, to [maintain compatibility][matrix].
  When used, this field contains the media type of this document, which differs from the [descriptor](descriptor.md#properties) use of `mediaType`.

- **`manifests`** *array of objects*

  This REQUIRED property contains a list of [manifests](manifest.md) for specific platforms.
  While this property MUST be present, the size of the array MAY be zero.

  Each object in `manifests` includes a set of [descriptor properties](descriptor.md#properties) with the following additional properties and restrictions:

  - **`mediaType`** *string*

    This [descriptor property](descriptor.md#properties) has additional restrictions for `manifests`.
    Implementations MUST support at least the following media types:

    - [`application/vnd.oci.image.manifest.v1+json`](manifest.md)

    Also, implementations SHOULD support the following media types:

    - `application/vnd.oci.image.index.v1+json` (nested index)

    Image indexes concerned with portability SHOULD use one of the above media types.
    Future versions of the spec MAY use a different mediatype (i.e. a new versioned format).
    An encountered `mediaType` that is unknown to the implementation MUST be ignored.

  - **`platform`** *object*

    This OPTIONAL property describes the minimum runtime requirements of the image.
    This property SHOULD be present if its target is platform-specific.

    - **`architecture`** *string*

        This REQUIRED property specifies the CPU architecture.
        Image indexes SHOULD use, and implementations SHOULD understand, values listed in the Go Language document for [`GOARCH`][go-environment2].

    - **`os`** *string*

        This REQUIRED property specifies the operating system.
        Image indexes SHOULD use, and implementations SHOULD understand, values listed in the Go Language document for [`GOOS`][go-environment2].

    - **`os.version`** *string*

        This OPTIONAL property specifies the version of the operating system targeted by the referenced blob.
        Implementations MAY refuse to use manifests where `os.version` is not known to work with the host OS version.
        Valid values are implementation-defined. e.g. `10.0.14393.1066` on `windows`.

    - **`os.features`** *array of strings*

        This OPTIONAL property specifies an array of strings, each specifying a mandatory OS feature.
        When `os` is `windows`, image indexes SHOULD use, and implementations SHOULD understand the following values:

        - `win32k`: image requires `win32k.sys` on the host (Note: `win32k.sys` is missing on Nano Server)

        When `os` is not `windows`, values are implementation-defined and SHOULD be submitted to this specification for standardization.

    - **`variant`** *string*

        This OPTIONAL property specifies the variant of the CPU.
        Image indexes SHOULD use, and implementations SHOULD understand, `variant` values listed in the [Platform Variants](#platform-variants) table.

    - **`features`** *array of strings*

        This property is RESERVED for future versions of the specification.

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for the image index.
    This OPTIONAL property MUST use the [annotation rules](annotations.md#rules).

    See [Pre-Defined Annotation Keys](annotations.md#pre-defined-annotation-keys).

## Platform Variants

When the variant of the CPU is not listed in the table, values are implementation-defined and SHOULD be submitted to this specification for standardization.

| ISA/ABI         | `architecture` | `variant`   |
|-----------------|----------------|-------------|
| ARM 32-bit, v6  | `arm`          | `v6`        |
| ARM 32-bit, v7  | `arm`          | `v7`        |
| ARM 32-bit, v8  | `arm`          | `v8`        |
| ARM 64-bit, v8  | `arm64`        | `v8`        |


*Example showing a simple image index pointing to image manifests for two platforms:*
```json,title=Image%20Index&mediatype=application/vnd.oci.image.index.v1%2Bjson
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

### 文件系统层

### `+gzip` Media Types

* The media type `application/vnd.oci.image.layer.v1.tar+gzip` represents an `application/vnd.oci.image.layer.v1.tar` payload which has been compressed with [gzip][rfc1952_2].

### 变化类型

Types of changes that can occur in a changeset are:

* Additions
* Modifications
* Removals

Additions and Modifications are represented the same in the changeset tar archive.

Removals are represented using "[whiteout](#whiteouts)" file entries (See [Representing Changes](#representing-changes)).

### 文件类型

* regular files
* directories
* sockets
* symbolic links
* block devices
* character devices
* FIFOs


### 创建文件系统层

#### Initial Root Filesystem

The initial root filesystem is the base or parent layer.

For this example, an image root filesystem has an initial state as an empty directory.
The name of the directory is not relevant to the layer itself, only for the purpose of producing comparisons.

Here is an initial empty directory structure for a changeset, with a unique directory name `rootfs-c9d-v1`.

```
rootfs-c9d-v1/
```

#### Populate Initial Filesystem

Files and directories are then created:

```
rootfs-c9d-v1/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

The `rootfs-c9d-v1` directory is then created as a plain [tar archive][tar-archive] with relative path to `rootfs-c9d-v1`.
Entries for the following files:

```
./
./etc/
./etc/my-app-config
./bin/
./bin/my-app-binary
./bin/my-app-tools
```

#### Populate a Comparison Filesystem

Create a new directory and initialize it with a copy or snapshot of the prior root filesystem.
Example commands that can preserve [file attributes](#file-attributes) to make this copy are:
* [cp(1)](http://linux.die.net/man/1/cp): `cp -a rootfs-c9d-v1/ rootfs-c9d-v1.s1/`
* [rsync(1)](http://linux.die.net/man/1/rsync):  `rsync -aHAX rootfs-c9d-v1/ rootfs-c9d-v1.s1/`
* [tar(1)](http://linux.die.net/man/1/tar): `mkdir rootfs-c9d-v1.s1 && tar --acls --xattrs -C rootfs-c9d-v1/ -c . | tar -C rootfs-c9d-v1.s1/ --acls --xattrs -x` (including `--selinux` where supported)

Any [changes](#change-types) to the snapshot MUST NOT change or affect the directory it was copied from.

For example `rootfs-c9d-v1.s1` is an identical snapshot of `rootfs-c9d-v1`.
In this way `rootfs-c9d-v1.s1` is prepared for updates and alterations.

**Implementor's Note**: *a copy-on-write or union filesystem can efficiently make directory snapshots*

Initial layout of the snapshot:

```
rootfs-c9d-v1.s1/
    etc/
        my-app-config
    bin/
        my-app-binary
        my-app-tools
```

See [Change Types](#change-types) for more details on changes.

For example, add a directory at `/etc/my-app.d` containing a default config file, removing the existing config file.
Also a change (in attribute or file content) to `./bin/my-app-tools` binary to handle the config layout change.

Following these changes, the representation of the `rootfs-c9d-v1.s1` directory:

```
rootfs-c9d-v1.s1/
    etc/
        my-app.d/
            default.cfg
    bin/
        my-app-binary
        my-app-tools
```

#### Determining Changes

When two directories are compared, the relative root is the top-level directory.
The directories are compared, looking for files that have been [added, modified, or removed](#change-types).

For this example, `rootfs-c9d-v1/` and `rootfs-c9d-v1.s1/` are recursively compared, each as relative root path.

The following changeset is found:

```
Added:      /etc/my-app.d/
Added:      /etc/my-app.d/default.cfg
Modified:   /bin/my-app-tools
Deleted:    /etc/my-app-config
```

This reflects the removal of `/etc/my-app-config` and creation of a file and directory at `/etc/my-app.d/default.cfg`.
`/bin/my-app-tools` has also been replaced with an updated version.

#### Representing Changes

A [tar archive][tar-archive] is then created which contains *only* this changeset:

- Added and modified files and directories in their entirety
- Deleted files or directories marked with a [whiteout file](#whiteouts)

The resulting tar archive for `rootfs-c9d-v1.s1` has the following entries:

```
./etc/my-app.d/
./etc/my-app.d/default.cfg
./bin/my-app-tools
./etc/.wh.my-app-config
```

To signify that the resource `./etc/my-app-config` MUST be removed when the changeset is applied, the basename of the entry is prefixed with `.wh.`.

## 镜像配置

This section defines the `application/vnd.oci.image.config.v1+json` [media type](media-types.md).

### [Layer](layer.md)

* Image filesystems are composed of *layers*.
* Each layer represents a set of filesystem changes in a tar-based [layer format](layer.md), recording files to be added, changed, or deleted relative to its parent layer.
* Layers do not have configuration metadata such as environment variables or default arguments - these are properties of the image as a whole rather than any particular layer.
* Using a layer-based or union filesystem such as AUFS, or by computing the diff from filesystem snapshots, the filesystem changeset can be used to present a series of image layers as if they were one cohesive filesystem.

### Image JSON

* Each image has an associated JSON structure which describes some basic information about the image such as date created, author, as well as execution/runtime configuration like its entrypoint, default arguments, networking, and volumes.
* The JSON structure also references a cryptographic hash of each layer used by the image, and provides history information for those layers.
* This JSON is considered to be immutable, because changing it would change the computed [ImageID](#imageid).
* Changing it means creating a new derived image, instead of changing the existing image.

### Layer DiffID

A layer DiffID is the digest over the layer's uncompressed tar archive and serialized in the descriptor digest format, e.g., `sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9`.
Layers SHOULD be packed and unpacked reproducibly to avoid changing the layer DiffID, for example by using [tar-split][] to save the tar headers.

NOTE: Do not confuse DiffIDs with [layer digests](manifest.md#image-manifest-property-descriptions), often referenced in the manifest, which are digests over compressed or uncompressed content.

### Layer ChainID

For convenience, it is sometimes useful to refer to a stack of layers with a single identifier.
While a layer's `DiffID` identifies a single changeset, the `ChainID` identifies the subsequent application of those changesets.
This ensures that we have handles referring to both the layer itself, as well as the result of the application of a series of changesets.
Use in combination with `rootfs.diff_ids` while applying layers to a root filesystem to uniquely and safely identify the result.

#### Definition

The `ChainID` of an applied set of layers is defined with the following recursion:

```
ChainID(L₀) =  DiffID(L₀)
ChainID(L₀|...|Lₙ₋₁|Lₙ) = Digest(ChainID(L₀|...|Lₙ₋₁) + " " + DiffID(Lₙ))
```

For this, we define the binary `|` operation to be the result of applying the right operand to the left operand.
For example, given base layer `A` and a changeset `B`, we refer to the result of applying `B` to `A` as `A|B`.

Above, we define the `ChainID` for a single layer (`L₀`) as equivalent to the `DiffID` for that layer.
Otherwise, the `ChainID` for a set of applied layers (`L₀|...|Lₙ₋₁|Lₙ`) is defined as the recursion `Digest(ChainID(L₀|...|Lₙ₋₁) + " " + DiffID(Lₙ))`.

#### Explanation

Let's say we have layers A, B, C, ordered from bottom to top, where A is the base and C is the top.

Let's expand the definition of `ChainID(A|B|C)` to explore its internal structure:

```
ChainID(A) = DiffID(A)
ChainID(A|B) = Digest(ChainID(A) + " " + DiffID(B))
ChainID(A|B|C) = Digest(ChainID(A|B) + " " + DiffID(C))
```

### ImageID

Each image's ID is given by the SHA256 hash of its [configuration JSON](#image-json).
It is represented as a hexadecimal encoding of 256 bits, e.g., `sha256:a9561eb1b190625c9adb5a9513e72c4dedafc1cb2d4c5236c9a6957ec7dfd5a9`.
Since the [configuration JSON](#image-json) that gets hashed references hashes of each layer in the image, this formulation of the ImageID makes images content-addressable.

## Properties

Note: Any OPTIONAL field MAY also be set to null, which is equivalent to being absent.

- **created** *string*, OPTIONAL

  An combined date and time at which the image was created, formatted as defined by [RFC 3339, section 5.6][rfc3339-s5.6].

- **author** *string*, OPTIONAL

  Gives the name and/or email address of the person or entity which created and is responsible for maintaining the image.

- **architecture** *string*, REQUIRED

  The CPU architecture which the binaries in this image are built to run on.
  Configurations SHOULD use, and implementations SHOULD understand, values listed in the Go Language document for [`GOARCH`][go-environment].

- **os** *string*, REQUIRED

  The name of the operating system which the image is built to run on.
  Configurations SHOULD use, and implementations SHOULD understand, values listed in the Go Language document for [`GOOS`][go-environment].

- **os.version** *string*, OPTIONAL

  This OPTIONAL property specifies the version of the operating system targeted by the referenced blob.
  Implementations MAY refuse to use manifests where `os.version` is not known to work with the host OS version.
  Valid values are implementation-defined. e.g. `10.0.14393.1066` on `windows`.

- **os.features** *array of strings*, OPTIONAL

  This OPTIONAL property specifies an array of strings, each specifying a mandatory OS feature.
  When `os` is `windows`, image indexes SHOULD use, and implementations SHOULD understand the following values:

  - `win32k`: image requires `win32k.sys` on the host (Note: `win32k.sys` is missing on Nano Server)

- **variant** *string*, OPTIONAL

  The variant of the specified CPU architecture.
  Configurations SHOULD use, and implementations SHOULD understand, `variant` values listed in the [Platform Variants](image-index.md#platform-variants) table.

- **config** *object*, OPTIONAL

  The execution parameters which SHOULD be used as a base when running a container using the image.
  This field can be `null`, in which case any execution parameters should be specified at creation of the container.

   - **User** *string*, OPTIONAL

     The username or UID which is a platform-specific structure that allows specific control over which user the process run as.
     This acts as a default value to use when the value is not specified when creating a container.
     For Linux based systems, all of the following are valid: `user`, `uid`, `user:group`, `uid:gid`, `uid:group`, `user:gid`.
     If `group`/`gid` is not specified, the default group and supplementary groups of the given `user`/`uid` in `/etc/passwd` from the container are applied.

   - **ExposedPorts** *object*, OPTIONAL

     A set of ports to expose from a container running this image.
     Its keys can be in the format of:
`port/tcp`, `port/udp`, `port` with the default protocol being `tcp` if not specified.
     These values act as defaults and are merged with any specified when creating a container.
     **NOTE:** This JSON structure value is unusual because it is a direct JSON serialization of the Go type `map[string]struct{}` and is represented in JSON as an object mapping its keys to an empty object.

   - **Env** *array of strings*, OPTIONAL

     Entries are in the format of `VARNAME=VARVALUE`.
     These values act as defaults and are merged with any specified when creating a container.

   - **Entrypoint** *array of strings*, OPTIONAL

     A list of arguments to use as the command to execute when the container starts.
     These values act as defaults and may be replaced by an entrypoint specified when creating a container.

   - **Cmd** *array of strings*, OPTIONAL

     Default arguments to the entrypoint of the container.
     These values act as defaults and may be replaced by any specified when creating a container.
     If an `Entrypoint` value is not specified, then the first entry of the `Cmd` array SHOULD be interpreted as the executable to run.

   - **Volumes** *object*, OPTIONAL

     A set of directories describing where the process is likely to write data specific to a container instance.
     **NOTE:** This JSON structure value is unusual because it is a direct JSON serialization of the Go type `map[string]struct{}` and is represented in JSON as an object mapping its keys to an empty object.

   - **WorkingDir** *string*, OPTIONAL

     Sets the current working directory of the entrypoint process in the container.
     This value acts as a default and may be replaced by a working directory specified when creating a container.

   - **Labels** *object*, OPTIONAL

     The field contains arbitrary metadata for the container.
     This property MUST use the [annotation rules](annotations.md#rules).

  - **StopSignal** *string*, OPTIONAL

    The field contains the system call signal that will be sent to the container to exit. The signal can be a signal name in the format `SIGNAME`, for instance `SIGKILL` or `SIGRTMIN+3`.

  - **Memory** *integer*, OPTIONAL

    This property is *reserved* for use, to [maintain compatibility](media-types.md#compatibility-matrix).

  - **MemorySwap** *integer*, OPTIONAL

    This property is *reserved* for use, to [maintain compatibility](media-types.md#compatibility-matrix).

  - **CpuShares** *integer*, OPTIONAL

    This property is *reserved* for use, to [maintain compatibility](media-types.md#compatibility-matrix).

  - **Healthcheck** *object*, OPTIONAL

    This property is *reserved* for use, to [maintain compatibility](media-types.md#compatibility-matrix).

- **rootfs** *object*, REQUIRED

   The rootfs key references the layer content addresses used by the image.
   This makes the image config hash depend on the filesystem hash.

    - **type** *string*, REQUIRED

       MUST be set to `layers`.
       Implementations MUST generate an error if they encounter a unknown value while verifying or unpacking an image.

    - **diff_ids** *array of strings*, REQUIRED

       An array of layer content hashes (`DiffIDs`), in order from first to last.

- **history** *array of objects*, OPTIONAL

  Describes the history of each layer.
  The array is ordered from first to last.
  The object has the following fields:

    - **created** *string*, OPTIONAL

       A combined date and time at which the layer was created, formatted as defined by [RFC 3339, section 5.6][rfc3339-s5.6].

    - **author** *string*, OPTIONAL

       The author of the build point.

    - **created_by** *string*, OPTIONAL

       The command which created the layer.

    - **comment** *string*, OPTIONAL

       A custom message set when creating the layer.

    - **empty_layer** *boolean*, OPTIONAL

       This field is used to mark if the history item created a filesystem diff.
       It is set to true if this history item doesn't correspond to an actual layer in the rootfs section (for example, Dockerfile's [ENV](https://docs.docker.com/engine/reference/builder/#/env) command results in no change to the filesystem).

Any extra fields in the Image JSON struct are considered implementation specific and MUST be ignored by any implementations which are unable to interpret them.

Whitespace is OPTIONAL and implementations MAY have compact JSON with no whitespace.

```json,title=Image%20JSON&mediatype=application/vnd.oci.image.config.v1%2Bjson
{
    "created": "2015-10-31T22:22:56.015925234Z",
    "author": "Alyssa P. Hacker <alyspdev@example.com>",
    "architecture": "amd64",
    "os": "linux",
    "config": {
        "User": "alice",
        "ExposedPorts": {
            "8080/tcp": {}
        },
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "FOO=oci_is_a",
            "BAR=well_written_spec"
        ],
        "Entrypoint": [
            "/bin/my-app-binary"
        ],
        "Cmd": [
            "--foreground",
            "--config",
            "/etc/my-app.d/default.cfg"
        ],
        "Volumes": {
            "/var/job-result-data": {},
            "/var/log/my-app-logs": {}
        },
        "WorkingDir": "/home/alice",
        "Labels": {
            "com.example.project.git.url": "https://example.com/project.git",
            "com.example.project.git.commit": "45a939b2999782a3f005621a8d0f29aa387e1d6b"
        }
    },
    "rootfs": {
      "diff_ids": [
        "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
        "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      ],
      "type": "layers"
    },
    "history": [
      {
        "created": "2015-10-31T22:22:54.690851953Z",
        "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
      },
      {
        "created": "2015-10-31T22:22:55.613815829Z",
        "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
        "empty_layer": true
      }
    ]
}
```

## 内容描述符（Content Descriptors）

This section defines the `application/vnd.oci.descriptor.v1+json` [media type](media-types.md).

### 属性

- **`mediaType`** *string*

  This REQUIRED property contains the media type of the referenced content.
  Values MUST comply with [RFC 6838][rfc6838], including the [naming requirements in its section 4.2][rfc6838-s4.2].

  The OCI image specification defines [several of its own MIME types](media-types.md) for resources defined in the specification.

- **`digest`** *string*

  This REQUIRED property is the _digest_ of the targeted content, conforming to the requirements outlined in [Digests](#digests).
  Retrieved content SHOULD be verified against this digest when consumed via untrusted sources.

- **`size`** *int64*

  This REQUIRED property specifies the size, in bytes, of the raw content.
  This property exists so that a client will have an expected size for the content before processing.
  If the length of the retrieved content does not match the specified length, the content SHOULD NOT be trusted.

- **`urls`** *array of strings*

  This OPTIONAL property specifies a list of URIs from which this object MAY be downloaded.
  Each entry MUST conform to [RFC 3986][rfc3986].
  Entries SHOULD use the `http` and `https` schemes, as defined in [RFC 7230][rfc7230-s2.7].

- **`annotations`** *string-string map*

    This OPTIONAL property contains arbitrary metadata for this descriptor.
    This OPTIONAL property MUST use the [annotation rules](annotations.md#rules).


### SHA-512

[SHA-512][rfc4634-s4.2] is a collision-resistant hash function which [may be more perfomant][sha256-vs-sha512] than [SHA-256](#sha-256) on some CPUs.
Implementations MAY implement SHA-512 digest verification for use in descriptors.


```json,title=Content%20Descriptor&mediatype=application/vnd.oci.descriptor.v1%2Bjson
{
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "size": 7682,
  "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270"
}
```

In the following example, the descriptor indicates that the referenced manifest is retrievable from a particular URL:

```json,title=Content%20Descriptor&mediatype=application/vnd.oci.descriptor.v1%2Bjson
{
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "size": 7682,
  "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
  "urls": [
    "https://example.com/example-manifest"
  ]
}
```
