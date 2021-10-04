## OCI运行时Spec
> The Open Container Initiative Runtime Specification aims to specify the configuration, execution environment, and lifecycle of a container.

> A container's configuration is specified as the config.json for the supported platforms and details the fields that enable the creation of a container. The execution environment 
> is specified to ensure that applications running inside a container have a consistent environment between runtimes along with common actions defined for the container's lifecycle.

包括三部分内容
- 配置文件（多平台Linux, Windows，Solaris...）
- 执行环境 (bundle)
- 生命周期和操作 (lifecycle)

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
    * **`cgroup`** 容器有自己隔离的cgroup层级
 
* **`path`** *(string可选的)* - namespace文件.
    必须是绝对地址 [runtime mount namespace](glossary.md#runtime-namespace).
    运行时必须让容器的进程加入到这个path所在的Namespace， 典型应用是k8spod的pause容器。

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

**`devices`** （可选的) 列举容器内必须包含的设备

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

#### 控制组（Control groups）
**`cgroupsPath`** (string, 可选) cgroups路径.
It can be used to either control the cgroups hierarchy for containers or to run a new process in an existing container.
**`devices`** (array of objects, 可选) configures the [allowed device list][cgroup-v1-devices].
**`memory`** (object, 可选) represents the cgroup subsystem `memory` and it's used to set limits on the container's memory usage.
**`cpu`** (object, 可选) represents the cgroup subsystems `cpu` and `cpusets`.
```json
"cgroupsPath": "/myRuntime/myContainer",
"resources": {
    "memory": {
    "limit": 100000,
    "reservation": 200000
    },
    "cpu": {
    "shares": 1024,
    "quota": 1000000,
    "period": 500000,
    "realtimeRuntime": 950000,
    "realtimePeriod": 1000000,
    "cpus": "2-3",
    },
    "devices": [
        {
            "allow": false,
            "access": "rwm"
        }
    ]
}
```

#### 系统设置（Sysctl）
**`sysctl`** (object, 可选) allows kernel parameters to be modified at runtime for the container.


```json
"sysctl": {
    "net.ipv4.ip_forward": "1",
    "net.core.somaxconn": "256"
}
```


#### 安全计算（Seccomp）

The following parameters can be specified to set up seccomp:
* **`syscalls`** *(array of objects, OPTIONAL)* - match a syscall in seccomp.
    * **`names`** *(array of strings, REQUIRED)* - the names of the syscalls.
        `names` MUST contain at least one entry.
    * **`action`** *(string, REQUIRED)* - the action for seccomp rules.
        A valid list of constants as of libseccomp v2.5.0 is shown below.
        * `SCMP_ACT_KILL`
        * `SCMP_ACT_KILL_PROCESS`
        * `SCMP_ACT_KILL_THREAD`
        * `SCMP_ACT_TRAP`
        * `SCMP_ACT_ERRNO`
        * `SCMP_ACT_TRACE`
        * `SCMP_ACT_ALLOW`
        * `SCMP_ACT_LOG`
        * `SCMP_ACT_NOTIFY`
* **`defaultAction`** *(string, REQUIRED)* - the default action for seccomp. Allowed values are the same as `syscalls[].action`.
* **`listenerPath`** *(string, OPTIONAL)* - specifies the path of UNIX domain socket over which the runtime will send the [container process state](#containerprocessstate) data structure when the `SCMP_ACT_NOTIFY` action is used.
```json
"seccomp": {
    "defaultAction": "SCMP_ACT_ALLOW",
    "architectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "getcwd",
                "chmod"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
```
#### rootfs挂载传播（Mount Propagation）
**`rootfsPropagation`** (string, 可选) 设置rootfs的挂载传播.
取值范围： `shared`, `slave`, `private` or `unbindable`.
注意，一个对等组（peer group） 是一组可以相互传播挂载/卸载信息的挂载点

* **`shared`**: the rootfs mount belongs to a new peer group.
    This means that further mounts (e.g. nested containers) will also belong to that peer group and will propagate events to the rootfs.
    Note this does not mean that it's shared with the host.
* **`slave`**: the rootfs mount receives propagation events from the host (e.g. if something is mounted on the host it will also appear in the container) but not the other way around.
* **`private`**: the rootfs mount doesn't receive mount propagation events from the host and further mounts in nested containers will be isolated from the host and from the rootfs (even if the nested container `rootfsPropagation` option is shared).
* **`unbindable`**: the rootfs mount is a private mount that cannot be bind-mounted.

```json
"rootfsPropagation": "slave",
```

#### 屏蔽路径（Masked Paths）

**`maskedPaths`** (array of strings, OPTIONAL) will mask over the provided paths inside the container so that they cannot be read.

```json
"maskedPaths": [
    "/proc/kcore"
]
```

#### 只读路径（Readonly Paths）

**`readonlyPaths`** (array of strings, 可选) will set the provided paths as readonly inside the container.
```json
"readonlyPaths": [
    "/proc/sys"
]
```

#### 挂载点标签（Mount Label）

**`mountLabel`** (string, 可选) 设置挂载点的Selinux context.
```json
"mountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c715,c811"
```

### 容器bundle
文件系统Bundle是一个容器格式，它包含了容器运行需要的 - 根文件系统和config，可以在bundle上执行spec规定的操作.

A Standard Container bundle contains all the information needed to load and run a container.
This includes the following artifacts:

1. <a name="containerFormat01" />`config.json`: contains configuration data.
    This REQUIRED file MUST reside in the root of the bundle directory and MUST be named `config.json`.
    See [`config.json`](config.md) for more details.

2. <a name="containerFormat02" />container's root filesystem: the directory referenced by [`root.path`](config.md#root), if that property is set in `config.json`.

### 运行时和生命周期
- 状态（State）
* **`ociVersion`** (string, REQUIRED) is version of the Open Container Initiative Runtime Specification with which the state complies.
* **`id`** (string, REQUIRED) is the container's ID.
    This MUST be unique across all containers on this host.
    There is no requirement that it be unique across hosts.
* **`status`** (string, REQUIRED) is the runtime state of the container.
    * `creating`: the container is being created (step 2 in the [lifecycle](#lifecycle))
    * `created`: the runtime has finished the [create operation](#create) (after step 2 in the [lifecycle](#lifecycle)), and the container process has neither exited nor executed the user-specified program
    * `running`: the container process has executed the user-specified program but has not exited (after step 8 in the [lifecycle](#lifecycle))
    * `stopped`: the container process has exited (step 10 in the [lifecycle](#lifecycle))

* **`pid`** (int, REQUIRED when `status` is `created` or `running` on Linux, OPTIONAL on other platforms) is the ID of the container process.
  For hooks executed in the runtime namespace, it is the pid as seen by the runtime.
  For hooks executed in the container namespace, it is the pid as seen by the container.
* **`bundle`** (string, REQUIRED) is the absolute path to the container's bundle directory.
    This is provided so that consumers can find the container's configuration and root filesystem on the host.
* **`annotations`** (map, OPTIONAL) contains the list of annotations associated with the container.
    If no annotations were provided then this property MAY either be absent or an empty map.
    
```json
{
    "ociVersion": "0.2.0",
    "id": "oci-container1",
    "status": "running",
    "pid": 4422,
    "bundle": "/containers/redis",
    "annotations": {
        "myKey": "myValue"
    }
}
```

- 生命周期（lifecycle）

1. OCI compliant runtime's [`create`](runtime.md#create) command is invoked with a reference to the location of the bundle and a unique identifier.
2. The container's runtime environment MUST be created according to the configuration in [`config.json`](config.md).
    If the runtime is unable to create the environment specified in the [`config.json`](config.md), it MUST [generate an error](#errors).
    While the resources requested in the [`config.json`](config.md) MUST be created, the user-specified program (from [`process`](config.md#process)) MUST NOT be run at this time.
    Any updates to [`config.json`](config.md) after this step MUST NOT affect the container.
3. The [`prestart` hooks](config.md#prestart) MUST be invoked by the runtime.
    If any `prestart` hook fails, the runtime MUST [generate an error](#errors), stop the container, and continue the lifecycle at step 12.
4. The [`createRuntime` hooks](config.md#createRuntime-hooks) MUST be invoked by the runtime.
    If any `createRuntime` hook fails, the runtime MUST [generate an error](#errors), stop the container, and continue the lifecycle at step 12.
5. The [`createContainer` hooks](config.md#createContainer-hooks) MUST be invoked by the runtime.
    If any `createContainer` hook fails, the runtime MUST [generate an error](#errors), stop the container, and continue the lifecycle at step 12.
6. Runtime's [`start`](runtime.md#start) command is invoked with the unique identifier of the container.
7. The [`startContainer` hooks](config.md#startContainer-hooks) MUST be invoked by the runtime.
    If any `startContainer` hook fails, the runtime MUST [generate an error](#errors), stop the container, and continue the lifecycle at step 12.
8. The runtime MUST run the user-specified program, as specified by [`process`](config.md#process).
9. The [`poststart` hooks](config.md#poststart) MUST be invoked by the runtime.
    If any `poststart` hook fails, the runtime MUST [log a warning](#warnings), but the remaining hooks and lifecycle continue as if the hook had succeeded.
10. The container process exits.
    This MAY happen due to erroring out, exiting, crashing or the runtime's [`kill`](runtime.md#kill) operation being invoked.
11. Runtime's [`delete`](runtime.md#delete) command is invoked with the unique identifier of the container.
12. The container MUST be destroyed by undoing the steps performed during create phase (step 2).
13. The [`poststop` hooks](config.md#poststop) MUST be invoked by the runtime.
    If any `poststop` hook fails, the runtime MUST [log a warning](#warnings), but the remaining hooks and lifecycle continue as if the hook had succeeded.

### 操作（Operations）

- Query State

`state <container-id>`

This operation MUST [generate an error](#errors) if it is not provided the ID of a container.
Attempting to query a container that does not exist MUST [generate an error](#errors).
This operation MUST return the state of a container as specified in the [State](#state) section.

- Create

`create <container-id> <path-to-bundle>`

This operation MUST [generate an error](#errors) if it is not provided a path to the bundle and the container ID to associate with the container.
If the ID provided is not unique across all containers within the scope of the runtime, or is not valid in any other way, the implementation MUST [generate an error](#errors) and a new container MUST NOT be created.
This operation MUST create a new container.

All of the properties configured in [`config.json`](config.md) except for [`process`](config.md#process) MUST be applied.
[`process.args`](config.md#process) MUST NOT be applied until triggered by the [`start`](#start) operation.
The remaining `process` properties MAY be applied by this operation.
If the runtime cannot apply a property as specified in the [configuration](config.md), it MUST [generate an error](#errors) and a new container MUST NOT be created.

The runtime MAY validate `config.json` against this spec, either generically or with respect to the local system capabilities, before creating the container ([step 2](#lifecycle)).
Runtime callers who are interested in pre-create validation can run [bundle-validation tools](implementations.md#testing--tools) before invoking the create operation.

Any changes made to the [`config.json`](config.md) file after this operation will not have an effect on the container.
- Start
`start <container-id>`

This operation MUST [generate an error](#errors) if it is not provided the container ID.
Attempting to `start` a container that is not [`created`](#state) MUST have no effect on the container and MUST [generate an error](#errors).
This operation MUST run the user-specified program as specified by [`process`](config.md#process).
This operation MUST generate an error if `process` was not set.
- Kill
`kill <container-id> <signal>`

This operation MUST [generate an error](#errors) if it is not provided the container ID.
Attempting to send a signal to a container that is neither [`created` nor `running`](#state) MUST have no effect on the container and MUST [generate an error](#errors).
This operation MUST send the specified signal to the container process.

- Delete
`delete <container-id>`

This operation MUST [generate an error](#errors) if it is not provided the container ID.
Attempting to `delete` a container that is not [`stopped`](#state) MUST have no effect on the container and MUST [generate an error](#errors).
Deleting a container MUST delete the resources that were created during the `create` step.
Note that resources associated with the container, but not created by this container, MUST NOT be deleted.
Once a container is deleted its ID MAY be used by a subsequent container.

- Hooks
Many of the operations specified in this specification have "hooks" that allow for additional actions to be taken before or after each operation.


