# OCI Image Spec
> This specification defines an OCI Image, consisting of a manifest, an image index (optional), a set of filesystem layers, and a configuration.
> The goal of this specification is to enable the creation of interoperable tools for building, transporting, and preparing a container image to run.

从上层划分，spec包括了
* [Image Manifest](manifest.md) - 描述镜像组成元素的文档
* [Image Index](image-index.md) - 按照平台分类的Manifest列表
* [Filesystem Layer](layer.md) - 容器文件系统的变化集合（changeset）。
* [Image Configuration](config.md) - a document determining layer ordering and configuration of the image suitable for translation into a [runtime bundle][runtime-spec]
* [Conversion](conversion.md) - a document describing how this translation should occur
* [Descriptor](descriptor.md) - a reference that describes the type, metadata and content address of referenced content
