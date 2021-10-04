## OCI运行时Spec
> The Open Container Initiative Runtime Specification aims to specify the configuration, execution environment, and lifecycle of a container.

> A container's configuration is specified as the config.json for the supported platforms and details the fields that enable the creation of a container. The execution environment > is specified to ensure that applications running inside a container have a consistent environment between runtimes along with common actions defined for the container's lifecycle.

包括三部分内容
- 配置文件（多平台Linux, Windows，Solaris...）
- 执行环境
- 生命周期和操作

### 配置文件（configuration）
配置是一个平台相关的config.json文件，这里这介绍Linux平台

#### 缺省文件系统 (filesystem)
在Linux 容器的文件系统里，下面这些是必须具备的（可以更多）。

| Path     | Type   |
| -------- | ------ |
| /proc    | [proc][] |
| /sys     | [sysfs][]  |
| /dev/pts | [devpts][] |
| /dev/shm | [tmpfs][]  |

```json
       "mounts": [
                {
                        "destination": "/proc",
                        "type": "proc",
                        "source": "proc"
                },
                {
                        "destination": "/dev",
                        "type": "tmpfs",
                        "source": "tmpfs",
                        "options": [
                                "nosuid",
                                "strictatime",
                                "mode=755",
                                "size=65536k"
                        ]
                },
                {
                        "destination": "/dev/pts",
                        "type": "devpts",
                        "source": "devpts",
                        "options": [
                                "nosuid",
                                "noexec",
                                "newinstance",
                                "ptmxmode=0666",
                                "mode=0620",
                                "gid=5"
                        ]
                },
                {
                        "destination": "/dev/shm",
                        "type": "tmpfs",
                        "source": "shm",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "mode=1777",
                                "size=65536k"
                        ]
                },
               {
                        "destination": "/dev/mqueue",
                        "type": "mqueue",
                        "source": "mqueue",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev"
                        ]
                },
                {
                        "destination": "/sys",
                        "type": "sysfs",
                        "source": "sysfs",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "ro"
                        ]
                },
                {
                        "destination": "/sys/fs/cgroup",
                        "type": "cgroup",
                        "source": "cgroup",
                        "options": [
                                "nosuid",
                                "noexec",
                                "nodev",
                                "relatime",
                                "ro"
                        ]
                }
        ],
```
#### 命名空间（namespace )

* **`type`** *(string, REQUIRED)* - namespace类型. Linux容器应该支持下面这些Namespace，
    * **`pid`** 容器进程应该只看见自己容器内的进程.
    * **`network`** 容器有自己的网络协议栈.
    * **`mount`** 容器有自己隔离的Mount表
    * **`ipc`** 只允许容器内的进程间通过IPC交互
    * **`uts`** 容器有自己的Host name和Domain name.
    * **`user`** 可以把host的用户和组ID映射到容器
    * **`cgroup`** 容器有自己隔离的cgroup hierarchy
 
* **`path`** *(string可选的)* - namespace文件.
    必须是绝对地址 [runtime mount namespace](glossary.md#runtime-namespace).
    运行时必须让容器的进程加入到这个path所在的Namespace.

   如果没有path，表明需要创建一个新的指定类型的[container namespace](glossary.md#container-namespace).

```json
"namespaces": [
    {
        "type": "pid",
        "path": "/proc/1234/ns/pid"
    },
    {
        "type": "network",
        "path": "/var/run/netns/neta"
    },
    {
        "type": "mount"
    },
    {
        "type": "ipc"
    },
    {
        "type": "uts"
    },
    {
        "type": "user"
    },
    {
        "type": "cgroup"
    }
]
```


#### 用户namespace的映射（host -> container）
**`uidMappings`** (array of objects, OPTIONAL) describes the user namespace uid mappings from the host to the container.
**`gidMappings`** (array of objects, OPTIONAL) describes the user namespace gid mappings from the host to the container.

每个映射包含下面的字段
* **`containerID`** *(uint32, REQUIRED)* - 容器内起始uid/gid编号.
* **`hostID`** *(uint32, REQUIRED)* - 主机内起始uid/gid编号（和上面ContainerID对应）
* **`size`** *(uint32, REQUIRED)* - 映射的条数.

```json
"uidMappings": [
    {
        "containerID": 0,
        "hostID": 1000,
        "size": 32000
    }
],
"gidMappings": [
    {
        "containerID": 0,
        "hostID": 1000,
        "size": 32000
    }
]
```

#### 设备（devices）

**`devices`** 可选的) 列举容器内必须包含的设备

Each entry has the following structure:

* **`type`** *(string, REQUIRED)* - type of device: `c`, `b`, `u` or `p`.
    More info in [mknod(1)][mknod.1].
* **`path`** *(string, REQUIRED)* - full path to device inside container.
    If a [file][] already exists at `path` that does not match the requested device, the runtime MUST generate an error.
* **`major, minor`** *(int64, REQUIRED unless `type` is `p`)* - [major, minor numbers][devices] for the device.
* **`fileMode`** *(uint32, OPTIONAL)* - file mode for the device.
    You can also control access to devices [with cgroups](#configLinuxDeviceAllowedlist).
* **`uid`** *(uint32, OPTIONAL)* - id of device owner in the [container namespace](glossary.md#container-namespace).
* **`gid`** *(uint32, OPTIONAL)* - id of device group in the [container namespace](glossary.md#container-namespace).

`type`, `major` and `minor`组合仅仅用在单个设备上，不能共享.

```json
"devices": [
    {
        "path": "/dev/fuse",
        "type": "c",
        "major": 10,
        "minor": 229,
        "fileMode": 438,
        "uid": 0,
        "gid": 0
    },
    {
        "path": "/dev/sda",
        "type": "b",
        "major": 8,
        "minor": 0,
        "fileMode": 432,
        "uid": 0,
        "gid": 0
    }
]
```
