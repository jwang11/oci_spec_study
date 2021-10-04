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
