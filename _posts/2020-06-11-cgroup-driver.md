---
layout: post
title: 'Cgroup Driver 선택하기'
author: ssup.2
date: 2020-06-11 10:00
tags: [cgroup, container, docker, kubernetes]
image:
---

카카오에서는 DKOS라고 불리는 Kubernetes 기반 Container Flatform을 개발 & 운영하고 있습니다. DKOS를 안정적으로 운영하기 위해서 Kubernetes를 분석하는 도중, Kubernetes의 [Cgroup Driver 문서](https://kubernetes.io/docs/setup/production-environment/#cgroup-drivers)에서 다음과 같은 문구를 확인할 수 있었습니다.

> We have seen cases in the field where nodes that are configured to use cgroupfs for the kubelet and Docker, and systemd for the rest of the processes running on the node becomes unstable under resource pressure.

문구의 내용은 systemd를 이용하는 Linux 환경에서 kubelet과 Docker가 Cgroup Driver로 cgroupfs Driver를 이용하도록 설정되어 있다면 System이 불안정해 진다는 내용입니다. System이 불안전해 진다는 내용 때문에 위의 문구를 완전히 파악할 필요가 있었고, Cgroup Driver를 분석하는 계기가 되었습니다. 본 글에서는 Cgroup 소개 및 Cgroup Driver의 역활 및 동작 구조를 소개하고, 분석 내용을 바탕으로 DKOS에서는 어떤 Cgroup Driver를 선택하였는지 소개하려고 합니다.

## Cgroup 이란?

Cgroup Driver를 소개하기 전에 먼저 Cgroup을 소개하려고 합니다. Cgroup은 Control Group의 약자로 다수의 Process가 포함되어 있는 Process Group 단위로 Resource 사용량을 제한하고 격리시키는 Linux의 기능입니다. 여기서 Resource는 CPU, Memory, Disk, Network를 의미합니다. Cgroup은 주로 Container의 Resource 제어를 위해서 많이 이용됩니다.

![Cgroup과 Container의 관계](/files/Cgroup_Driver_Container_Cgroup.PNG)

위의 예제는 Container와 Cgroup의 관계를 나타내고 있습니다. Container가 생성된다면 생성된 Container의 Process들을 담당하는 Container Cgroup이 생성됩니다. Container의 모든 Process들은 해당 Container Cgroup에 소속됩니다. Container 내부 Process들이 Fork System Call을 통해서 Child Process들을 생성해도, 생성된 Child Process들 모두 Container Cgroup에 소속됩니다. 따라서 Container의 Resource를 제어하기 위해서는 Container Cgroup을 제어하면 됩니다.

## Cgroup 제어하기

그렇다면 Docker와 같이 Container를 관리하는 도구는 Cgroup을 어떻게 제어할까요? Cgroup을 제어하는 방법은 cgroupfs을 이용하는 방법과 systemd에서 제공하는 API를 이용하는 방법 2가지가 존재합니다.

### cgroupfs를 이용하여 Cgroup 제어

Linux에서는 Cgroup을 제어하는 방법으로 cgroupfs을 제공합니다. cgroupfs은 이름에서 알 수 있는것 처럼 Cgroup 제어를 위해 탄생한 특수 File System 입니다. Cgroup 정보는 Linux Kernel 내부에 관리하고 있으며, Linux Kernel은 Cgroup 제어를 위해서 별도의 System Call 제공하는 방식이 아닌 cgroupfs을 제공하고 있습니다. Linux Kernel이 갖고 있는 Cgroup 정보는 cgroupfs의 Directory와 File로 나타나며, Cgroup 제어는 Directory 생성,삭제 및 File의 내용 변경을 통해서 이루어집니다. 따라서 cgroupfs으로 Cgroup을 제어할때는 "mkdir, rmdir, echo"와 같은 명령어들을 이용할 수 있습니다.

```
(root)# mount 
...
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
...
```

systemd는 cgroupfs을 /sys/fs/cgroup 하위 경로에 Mount합니다. 따라서 systemd를 이용하는 Linux 환경에서는 User가 별도로 cgroupfs을 Mount하지 않았어도 /sys/fs/cgroup 하위 경로에 cgroupfs들이 Mount되어 있습니다. mount 명령어를 통해서 Mount되어 있는 cgroupfs들을 확인할 수 있습니다. 실제 Cgroup은 제어하는 Resource Type 별로 존재합니다. 따라서 cgroupfs은 하나만 Mount 되어 있지 않고 Resource Type 별로 별도로 Mount 되어 있습니다.

각 cgroupfs이 담당하는 Resource Type은 Mount Option을 통해서 알 수 있습니다. 예를들어 위 예제의 cgroupfs 중에서 /sys/fs/cgroup/memory 경로에 Mount된 마지막 cgroupfs의 Mount Option에는 memory가 있기 때문에, 마지막 cgroupfs은 Memory를 담당하는 cgroupfs이라는 의미입니다.

```
(root)# cd /sys/fs/cgroup/memory
(root)# mkdir group_a
(root)# cd group_a && ls
cgroup.clone_children       memory.kmem.max_usage_in_bytes      memory.max_usage_in_bytes        memory.usage_in_bytes
cgroup.event_control        memory.kmem.slabinfo                memory.move_charge_at_immigrate  memory.use_hierarchy
cgroup.procs                memory.kmem.tcp.failcnt             memory.numa_stat                 notify_on_release
cgroup.sane_behavior        memory.kmem.tcp.limit_in_bytes      memory.oom_control               release_agent
memory.failcnt              memory.kmem.tcp.max_usage_in_bytes  memory.pressure_level            system.slice
memory.force_empty          memory.kmem.tcp.usage_in_bytes      memory.soft_limit_in_bytes       tasks
memory.kmem.failcnt         memory.kmem.usage_in_bytes          memory.stat                      user.slice
memory.kmem.limit_in_bytes  memory.limit_in_bytes               memory.swappiness
```

cgroupfs을 통해서 Cgroup을 생성하는 방법은 cgroupfs에 Directory를 생성하면 됩니다. 즉 cgroupfs안의 Directory는 하나의 Cgroup이라는 의미이며, Cgroup 사이의 관계는 Directory 구조처럼 Tree 형태를 갖는다는 의미입니다. 위의 예제에서는 Memory를 담당하는 cgroupfs으로 이동하여 mkdir로 명령어를 통해서 group_a라는 Memory Cgroup을 생성하는 과정을 나타내고 있습니다. group_a라는 Directory만 생성했을뿐 group_a Directory 내부에는 어떠한 File도 생성하지 않았지만, 실제로 보면 다수의 File들이 생성되어 있는걸 확인할 수 있습니다. 이러한 File들은 group_a Memory Cgroup의 상태 확인 및 설정에 이용되는 File입니다.

```
(root)# sleep infinity &
[1] 16661
(root)# echo 16661 > cgroup.procs
(root)# cat cgroup.procs
16661
```

위의 예제는 생성한 group_a Memory Cgroup에 sleep Process를 넣고 확인하는 과정을 나타내고 있습니다. sleep Process의 PID를 echo 명령어를 통해서 cgroup.procs File에 쓰면 sleep Process는 group_a Memory Cgroup에 소속됩니다. 반대로 group_a Memory Cgroup에 설정된 PID를 확인하기 위해서는 cat 명령어를 통해서 cgroup.procs 파일의 내용을 확인하면 됩니다. 이와 같은 방식으로 Linux Kernel은 cgroupfs을 통해서 Cgroup을 제어할 수 있도록 합니다.

### systemd를 이용하여 Cgroup 제어

systemd는 Linux의 Init Process로써 Daemon Process 제어 역활과 더불어 Cgroup을 제어하는 역활도 수행합니다. systemd의 Cgroup 제어 기능은 systemd가 제어하는 Unit의 일부 기능으로 포함되어 있습니다. 여기서 systemd의 Unit은 systemd가 제어하는 Daemon Process라고 이해하시면 됩니다. 즉 systemd의 Cgroup 제어 기능은 자신이 제어하는 Daemon Process의 Resource 사용량을 제어하기 위해서 이용됩니다.

```
(root)# systemd-run --unit=myunit sleep infinity
Running as unit: myunit.service
(root)# ps -ef | grep sleep
5255
(root)# cd /sys/fs/cgroup/memory/system.slice
(root)# ls
...
myunit.service
...
(root)# cd myunit.service
(root)# cat cgroup.procs
5255
```

systemd-run 명령어는 systemd가 제공하는 API를 이용하여 Unit을 생성하고 실행하는 명령어 입니다. 위의 예제는 systemd-run 명령어를 통해서 sleep Process를 실행하는 myunit이라는 Unit을 만들고, 실제 Cgroup이 생성되었는지 확인하는 과정을 나타내고 있습니다. systemd는 생성한 Unit의 Cgroup을 기본적으로 /sys/fs/cgroup/memory/system.slice 경로의 하위에 생성합니다. myunit.service라는 Cgroup이 생성되어 있는것을 확인할 수 있고, 해당 Cgroup에 5255 PID를 갖고 있는 sleep Process가 소속되어 있는 것을 확인할 수 있습니다.

## Cgroup Driver 분석

Cgroup Driver는 Cgroup을 관리하는 Module을 의미합니다. Cgroup Driver는 cgroupfs Driver와 systemd Driver가 존재합니다. 이름에서 알수 있는것 처럼 cgroupfs Driver는 자신이 직접 cgroupfs을 통해서 Cgroup을 제어합니다. 반면 systemd Driver는 systemd를 통해서 Cgroup을 제어합니다.

![Cgroup Driver에 따른 Cgroup 설정 과정](/files/Cgroup_Driver_Cgroup_Driver.PNG)

위의 그림은 Kubernetes Cluster에서 Docker를 이용할 경우, Cgroup Driver의 위치와 동작 과정을 나타내고 있습니다. Cgroup Driver는 libcontainer Library에 포함되어 있으며, libcontainer Library는 kubelet과 runc에서 이용됩니다. 어떤 Cgroup Driver를 이용할지는 kubelet과 Docker Daemon에 각각 설정되며, 두 Cgroup Driver는 반드시 동일해야 합니다. Docker Daemon에 설정된 Cgroup Driver는 Docker Daemon과 containerd의 명령에 따라서 실제로 Container 생성을 담당하는 runc가 이용합니다.

systemd의 API는 systemd에서 제공하는 IPC 기법인 D-Bus를 통해서 다른 Process들에게 노출됩니다. 따라서 systemd Driver는 D-Bus를 통해서 systemd와 통신합니다. systemd는 DBUS를 통해서 전달된 systemd Driver의 요청에 따라서 Cgroup을 제어합니다. systemd가 Cgroup을 제어할때는 cgroupfs Driver와 동일하게 cgroupfs을 이용합니다. 본 글의 cgroupfs을 소개하는 부분에서 systemd가 /sys/fs/cgroup 경로에 cgroupfs을 mount한다고 언급했었는데, systemd가 cgroupfs을 이용하여 Cgroup을 제어해야 하기 때문에 systemd가 cgroupfs의 Mount도 관리하는 것입니다.

> /sys/fs/cgroup/memory/kubepods/besteffort/pod0798b0e6_e445_4738_87b6_91bc5cc5e57f/33dff8e50378bb9baa47bccda0b21d57162ac63411e92cd1e0ff6a9a155470a6

cgroupfs Driver를 통해서 생성되는 Cgroup의 경로와 systemd Driver를 통해서 생성되는 Cgroup의 경로는 서로의 정책으로 인해 다릅니다. 위의 경로는 cgroupfs Driver를 통해서 생성된 특정 Container의 Memory Cgroup 경로를 나타내고 있습니다. Pod의 QoS, Pod의 ID, Container ID로 구성되어 있는걸 확인할 수 있습니다.

> /sys/fs/cgroup/memory/kubepods.slice/kubepods-besteffort.slice/kubepods-besteffort-pod0798b0e6_e445_4738_87b6_91bc5cc5e57f.slice/docker-3dff8e50378bb9baa47bccda0b21d57162ac63411e92cd1e0ff6a9a155470a6.scope

위의 경로는 동일한 Container이지만 systemd Driver를 이용할 경우 생성되는 Memory Cgroup의 경로를 나타내고 있습니다. cgroupfs Driver와 동일하게 Pod의 QoS, Pod의 ID, Container ID로 구성되어 있지만 이름이 조금씩 다른걸 확인할 수 있습니다. 경로에 slice, scope가 붙어 있는걸 알 수 있는데, slice와 scope는 systemd에서 Cgroup을 관리하기 위한 Unit의 Type을 나타냅니다. 이처럼 systemd가 생성하는 Cgroup에는 반드시 Unit의 Type이 붙게 됩니다.

## Cgroup Drvier 선택

Cgroup Driver를 분석하고 나니 systemd를 이용하는 환경에서 cgroupfs을 이용할 경우 System이 불안전해 질수 있다는 Kubernetes의 Cgroup Driver 문서에 더욱더 공감할 수 없었습니다. cgroupfs Driver가 cgroupfs을 통해서 설정한 Cgroup이나, systemd가 cgroupfs을 통해서 설정한 Cgroup이나 Linux Kernel의 입장에서는 다를게 없다고 판단되었기 때문입니다.

* https://github.com/kubernetes/kubeadm/issues/1394#issuecomment-627068509
* https://github.com/kubernetes/website/commit/43764bd6fba6e6a4004a47e65ce0223942766f33#commitcomment-39111396
* https://github.com/systemd/systemd/issues/8645#issuecomment-628326452

좀더 명확한 원인을 알고 싶어 관련 질문들을 Github에 남겼습니다. 위의 링크들은 제가 Github에 남긴 질문과 답변이 있는 Link입니다. 결론적으로 cgroupfs Driver와 systemd를 같이 쓰면 문제가 되는 점은 Cgroup 관리 정책의 충돌 부분이었지, System의 안전성과는 관계가 없다는 결론을 내릴수 있었습니다. 따라서 다음과 같은 이유들 때문에 DKOS에서는 기존에 이용하던 cgorupfs Driver를 그대로 이용하도록 결정하였습니다.

1. DKOS가 구성하는 Kubernetes Cluster의 Node들은 Kubernetes를 위한 전용 VM/PM Node입니다. 해당 Node들은 Kubernetes 구동만 고려하면 되기 때문에, cgroupfs Driver와 systemd의 Cgroup 관리 정책의 충돌이 실제로 관리를 어렵게 하는 요소가 되지 않았습니다.
1. Cgroup을 설정하는데 User Level App인 systemd를 이용하는 systemd Driver 보다는, cgroupfs을 통해서 Cgroup을 직접 설정하는 cgroupfs Driver가 더 안정적인 확률이 높다고 판단하였습니다.
1. DKOS는 Kubernetes Cluster 구성시 지금까지 cgroupfs Driver를 이용했었고, cgroupfs Driver로 인한 이슈를 경험한적이 없습니다.

## 마치며

결론은 단순하게 "현재 이용하고 있는 cgroupfs Driver를 그대로 이용하자" 이지만, Cgroup Driver를 분석하고 선택하는 과정을 통해서 Kubernetes를 좀더 깊게 이해할 수 있었습니다.

클라우드네이티브2셀에서는 DKOS를 기반으로 사내에 안정적인 Kubernetes Cluster를 제공하기 위해서 꾸준히 노력하고 있습니다. 같이 DKOS를 발전해 나가고 싶으신 분들은 아래의 모집 공고를 참고하여 많은 관심과 지원 부탁드립니다.
* [컨테이너 클라우드 개발자 모집](https://careers.kakao.com/jobs/P-11556)
