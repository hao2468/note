# docker对比虚拟机优势
虚拟机由hypervisor管理虚拟机，虚拟机需运行Guest OS，有资源消耗
虚拟机对系统的调用需经过虚拟化软件拦截处理，有资源消耗

## 缺点
docker中容器共享同一个系统内核，docker容器中可以挂载不同版本的操作系统文件，但需要能被宿主机内核支持，如无法在linux内核的宿主机的docker中挂载windows，或挂载更高版本的linux系统
容器使用namespace技术进行隔离，但有些资源无法被隔离，如在容器进行系统时间的修改，会对整个宿主机生效。因此，安全性不足。
由于共用系统资源，容器的资源可能被宿主机上的其他进程占用，也可能容器占用了其他进程的资源

## Cgroups
Cgroups限制进程使用系统资源
以文件形式设置限制，文件组织在/sys/fs/cgroup ，其中包含cpu、blkio、cpuset、memory等子目录，每个子目录中的文件限制对应的资源
在这些子目录中创建文件夹，新建的文件夹里会有系统自动创建的一些文件，通过修改这些文件进行资源控制，称为控制组

## 总结
docker容器实际上就是一个由docker设置了namespace和Cgroup的单进程模型
单进程模型是指容器没有管理多个进程的能力，而不是只能运行一个进程



# Mount namespace
容器进程开启了Mount namespace，需要进行挂载才能生效
未进行挂载前，容器直接继承宿主机的挂载点
Mount namespace生效后，宿主机对容器的挂载不可见
挂载在容器上的完整操作系统的文件系统，即所谓的容器镜像，被称为rootfs（根文件系统）
rootfs只包含文件系统，不包含系统内核
容器共用宿主机的系统内核

# chroot命令
一个可以将进程的根目录修改为指定目录的linux命令
Mount namespace是基于对chroot的改良而发明出来

# docker创建容器的过程
1.启用linux namespace配置
2.设置指定的Cgroups参数
3.切换进程的根目录（pivot_root（优先）或chroot）

# 容器镜像rootfs
用户制作镜像的每一个操作生成一个层，即一个增量的rootfs
最后将所有的层合并成一个rootfs，即最终的镜像
合并使用的是联合文件系统（Union File System），可以将多个不同位置的目录联合挂载到同一个目录
|-A
  |-a
  |-x
|-B
  |-b
  |-x
可将如上2个目录A和B合并成一个目录C，如下
|-C
  |-a
  |-b
  |-x
对C中的文件进行修改，A和B中也会出现对应修改


一个docker创建的完整的rootfs包含3大层：可读写层（rw）、init层（ro+wh）、只读层（ro+wh）
## 只读层
用户制作镜像过程中的每个操作生成的层（增量rootfs），全部属于只读层
该层内容不会被修改，因此下载的镜像都可以完全复现制作者最初的完整环境，即容器技术的“强一致性”
## 可读写层
可读写层用于记录对容器镜像的修改
    如需要删除只读层中的文件foo，就会在可读写层中创建一个.wh.foo文件，当只读层和可读写层联合挂载后，只读层中的foo文件就会被遮蔽，但实际上不会被删除，只是不可见
    所有对只读层的修改都保存在可读写层中，并且不会实际对只读层的内容进行修改
## init层
用于存放启动容器时写入的指定值
而这些写入的修改一般只对当前容器有效，因此docker将这些修改单独以init层保存
docker commit时，不会上传init层


注意：docker on Mac 和windows docker实际上都是基于虚拟化技术实现，只有linux中的docker是使用容器技术

镜像的各个层保存在宿主机的/var/lib/docker/aufs/diff中
容器启动后，各个层联合挂载在/var/lib/docker/aufs/mnt/中

# DockerFile

原语：
FROM                 指定使用的镜像
RUN                  在容器中执行shell命令
WORKDIR              指定容器中的某个目录为当前目录
ADD                  将宿主机中指定目录（.表示宿主机中dockerfile所在目录）的文件复制到容器指定目录中
ENV                  设置容器的环境变量
EXPOSE               将容器的xx端口暴露给外界访问
CMD、ENTERYPOINT     指定容器启动参数。其完整执行格式为ENTERYPOINT CMD，CMD是ENTERYPOINT的参数。ENTERYPOINT可选，docker默认为/bin/sh -c。因此将docker容器的启动进程称为ENTERYPOINT

每个原语执行后都会生成一个对应的镜像层

# docker commit

可以使用docker commit命令将正在运行的容器直接提交为一个新镜像。此时，执行命令前在容器里进行的所有操作都会被保存到新镜像的可读写层中（如增删等）。commit不会上传init层

# docker exec

可以在宿主机的/proc/进程号/ns中查看某个进程的所有namespace对应的文件
docker exec进入容器进行操作的原理就是通过将一个进程加入到容器的namespace中，从而达到进入容器的目的

# docker volume

volume（数据卷）允许将宿主机上指定的目录或文件挂载到容器中进行读取和修改操作
volume的挂载是在执行chroot之前进行，将指定的宿主机目录或文件挂载到指定的容器目录在宿主机上对应的目录（宿主机存放镜像可读写层的目录）上
volume使用挂载技术是linux的绑定挂载（bind mount）技术，将某个目录或文件挂载到指定目录上，对挂载点进行的操作只在被挂载的目录或文件上生效，挂载点原来的内容被隐藏起来不受影响，当执行unmount后，挂载点恢复原样。因此，将宿主机的某个目录或文件挂载到容器中进行操作会对宿主机上的该目录或文件生效，但不会对容器镜像的内容产生影响，即不会被记录在可读写层中。一般容器中用作挂载点的文件是空的。


# dockerinit

dockerinit（容器初始化进程）会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作。最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1 的进程。

完整容器镜像层级结构
|-联合挂载的rootfs
|-可读写层
|-init层
|-只读层

|-Kubernetes
    |-master
        |-kube-apiserver
        |-kube-scheduler
        |-kube-controller-manager
    |-node
        |-kubelet

# kubelet
1.kubelet负责通过CRI（Container Runtime Interface）同容器运行时(如docker)打交道
容器运行时通过OCI与Linux操作系统进行交互，即把CRI请求翻译成对linux系统的调用

2.kubelet通过gRPC协议与Device Plugin交互。Device Plugin这个插件是kubernetes用来管理GPU等宿主机物理设备的组件

3.kubelet调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与 kubelet 进行交互的接口，分别是 CNI（Container Networking Interface）和CSI（Container Storage Interface）

# kubernetes使用方法
1.通过一个编排对象，如pod、job、cronjob等，描述需要管理的应用
2.为这个编排对象定义服务对象，如service、secret、horizontal pod autoscaler，负责具体平台级功能（管理功能）
这就是“声明式api”，这种api对应的编排对象和服务对象都是api对象（api object）


编排：按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。
调度：把一个容器按照某种规则，放置在某个最佳结点上运行起来。

# staic pod
允许把要部署的pod的yaml文件放在一个指定的目录中，当kubelet启动时就会检查该目录，加载其中的yaml文件启动对应的pod。即可在kubernetes集群未启动前，启动指定容器。
kubelet在kubernetes中是一个完全独立的组件。

# pod
pod是kubernetes的原子调度单位，最小的api对象
pod只是一个逻辑概念，实际上并不存在所谓的pod的边界或隔离环境
pod其实是一组共享了同一个network namespace的容器，也可以共享同一个volume

pod的实现需要一个叫infra的中间容器，在pod中永远先创建infra容器，其他用户定义的容器通过join network namespace的方式与infra容器关联起来。

infra容器解压后只有100-200kb，且永远处于暂停状态，主要用于占用network namespace，被其他属于同一个pod的容器加入到该network namespace

pod的生命周期和infra容器的生命周期一致，与pod中的其他容器无关

如果需要开发网络插件，应重点考虑如何配置pod的network namespace，而不是由用户容器去适应pod的网络配置

在pod中，init contianer比其他用户定义的容器先启动，init contianer逐一启动并退出后，用户定义的容器再启动

pod可以看作是一个虚拟机，pod里的每个容器看作是一个进程
可以在pod中运行一个辅助容器来对pod的volume进行数据读取存储工作，以辅助主容器的工作，这种辅助容器称为sidecar

# pod字段
## nodeselector
用法：
nodeselector:
label:label
nodeselector：将pod与node进行绑定，绑定后，pod只能运行有"label:label"标签的节点（node）上

## nodename
nodename一般由调度器负责设置，当该pod经过调度后，调度结果（节点名字）将被赋值给nodename。当用户手动设置时，将骗过调度器，一般在测试或调试时使用。

## hostaliases
用于定义pod的hosts文件的内容。如需设置hosts文件内容，一定要通过该字段设置；如果直接修改pod中的hosts文件，当pod重建后修改会失效，因为pod创建是根据yaml文件中的hostaliases进行设置。

## shareProcessNamespace
用法：
shareProcessNamespace：true（false）
设置为true后pod中容器共享PID namespace，即该pod中的每个容器中的进程对该pod中的所有容器可见

## hostxxx
如：hostNetwork、hostIPC、hostPID
设置为true后，该pod中的所有容器直接使用宿主机的网络、直接与宿主机进行IPC通信、可见所有宿主机进程

# container字段
## ImagePullPolicy
镜像拉取策略，container中的字段。可选值：Always、Never、IfNotPresent
Always：每次创建pod重新拉取镜像
Never：永远不会主动拉取镜像
IfNotPresent：宿主机上不存在该镜像时拉取

## Lifecycle
container lifecycle hooks，设置在容器状态发生变化时触发钩子，container中的字段。
用法：
lifecycle：
    postStart：
        exec：
            command：[]
    preStop：
        exec：
            command：[]       
postStart：在容器启动之后，即在ENTRYPOINT执行后，执行command中的指令，但不严格保证顺序，即postStart启动时，ENTRYPOINT不一定结束。
如果postStart执行出错或超时，kubernetes会在pod的event中报告错误信息，pod处于失败状态

preStop：在容器被杀死之前执行，收到杀死容器的信号时，会阻塞容器杀死流程，执行preStop，完成preStop后再杀死容器

## pod.status.phase
该字段表示pod的当前状态
1. Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
2. Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
3. Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
4. Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kubeapiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

在status字段中，还有一组Conditions字段，包括PodScheduled、Ready、Initialized，以及 Unschedulable，用于描述造成当前Status的具体原因。Ready表示pod已正常启动并且可以提供服务。

# Projected Volume
kubernetes1.11新特性
4种Project Volume：
    1. Secret；
    2. ConfigMap；
    3. Downward API；
    4. ServiceAccountToken

## Secret
作用是把pod想要访问的加密数据存放到Etcd中。pod中的容器就可以通过挂载Volume来访问Secret保存的信息。

pod示例yaml：
volumes:
- name: mysql-cred
    projected:
        sources:
        - secret:
            name: user
        - secret:
            name: pass

添加Secret的方法：
1.将文件内容添加为Secret
kubectl create secret generic user --from-file=./username.txt
文件内容可以明文表示

2.通过yaml文件创建Secret对象
apiVersion: v1
kind: Secret
metadata:
    name: mysecret
type: Opaque
data:
    user: YWRtaW4=
    pass: MWYyZDFlMmU2N2Rm
数据以键值对存储，在yaml文件中存储的数据必须经过Base64转码（如YWRtaW4=）
转码操作： echo -n 'admin' | base64

创建pod后，对应的Secret对象会在容器里的/projected-volume/以文件形式出现

注意：Secret对象实际存储在Etcd中，由kubelet组件维护，当Etcd中对应的数据更新后，这些Volume中的文件内容也会被更新，但更新可能会存在延时。编写程序时应加入延时应对处理


## ConfigMap
ConfigMap与Secret相似，用法也基本相同，但ConfigMap保存的是不需要加密的、应用所需的配置信息。

创建方式：
1.从文件创建ConfigMap
kubectl create configmap ui-config --from-file=example/ui.properti

2.直接编写ConfigMap的yaml文件

## Downward API
作用是让pod里的容器能直接获取该pod API对象的信息
会将pod的API对象的信息挂载成容器中/etc/podinfo/xx的文件（xx为path的值）
注意：Downward API只能获取pod中容器进程启动前就能确定的信息。
示例：
volumes:
- name: podinfo
projected:
sources:
- downwardAPI:
items:
- path: "labels"
fieldRef:
fieldPath: metadata.labels
示例中将pod的metadata.labels的信息挂载到了容器的/etc/podinfo/labels中

Downward API支持的字段：（使用前查k8s的官方文档）
1. 使用 fieldRef 可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机 IP
metadata.name - Pod 的名字
metadata.namespace - Pod 的 Namespace
status.podIP - Pod 的 IP
spec.serviceAccountName - Pod 的 Service Account 的名字
metadata.uid - Pod 的 UID
metadata.labels['<KEY>'] - 指定 <KEY> 的 Label 值
metadata.annotations['<KEY>'] - 指定 <KEY> 的 Annotation 值
metadata.labels - Pod 的所有 Label
metadata.annotations - Pod 的所有 Annotation
2. 使用 resourceFieldRef 可以声明使用:
容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request

## Service Account
Service Account用于解决在容器内部使用Kubernetes的Client访问操作Kubernetes API的授权问题

Service Account是一种特殊的Secret对象，叫做ServiceAccountToken。所有运行在kubernetes集群上的应用必须使用这个Token才能合法地访问API Server

Kubernetes会提供一个默认的服务账户（default Service Account），所有运行在kubernetes中的pod都可以直接使用，kubernetes会在创建pod时自动添加默认ServiceAccountToken的定义。
Token会被挂载到容器的/var/run/secrets/kubernetes.io/serviceaccount文件夹中
应用程序只要加载这个文件夹中的授权文件就可以访问操作kubernetes api，如果使用的是kubernetes官方的client包会自动加载这些文件。

这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”

可以设置kubernetes不自动挂载默认ServiceAccountToken

# Probe
Probe探针，用于检查容器健康状态，kubelet根据Probe的返回值决定容器状态，而不是以容器是否运行作为依据。

用法示例：
spec:
    livenessProbe:
    exec:
    command:
        - cat
        - /tmp/healthy
    initialDelaySeconds: 5
    periodSeconds: 5

initialDelaySeconds:容器启动后x秒执行健康检查
periodSeconds：每隔x秒执行一次健康检查

在示例中，通过cat命令读取/tmp/healthy文件，如果文件存在cat返回0，表示该容器健康。

此外，Probe可以定义为发起http或tcp请求的方式
livenessProbe:
httpGet:
path: /healthz
port: 8080
httpHeaders:
- name: X-Custom-Header
value: Awesome
initialDelaySeconds: 3
periodSeconds: 3

...
livenessProbe:
tcpSocket:
port: 8080
initialDelaySeconds: 15
periodSeconds: 20

# restartPolicy
当Probe返回非0时，认为容器不健康，kubernetes将根据pod.spec.restartPolicy字段执行操作（即恢复策略）。

pod.spec.restartPolicy可选值有3种：
1.Always：在任何情况下，只要容器不在运行状态，就自动重启容器（包括容器进程成功运行完毕正常退出）；
2.OnFailure: 只在容器异常时才自动重启容器；
3.Never: 从来不重启容器（需要容器退出后的日志、文件和目录时应设置为Never，否则重启将丢失这些信息）

当restartPolicy选择为Always或OnFailure时，容器出现异常就会被重启，因此pod会一致保持running状态。
restartPolicy选择为Never时，只有pod中的所有容器都异常退出，pod才会变为failed状态。

# PodPreset
PodPreset用于预定义pod的追加字段，kubernetes创建用户定义的pod时会根据PodPreset对象添加预定义的字段。

示例：
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
    name: allow-database
spec:
    selector:
        matchLabels:
            role: frontend
env:
    - name: DB_PORT
        value: "6379"
volumeMounts:
    - mountPath: /cache
        name: cache-volume
volumes:
    - name: cache-volume
        emptyDir: {}

PodPreset应设置selector，决定适用的pod，防止误修改其他pod。
当一个pod被kubernetes根据PodPreset修改后，会自动加上一个annotation，表示其被PodPreset修改过。

如果定义同时作用于一个pod对象的多个PodPreset，kubernetes会合并这多个PodPreset（都生效），如果修改有冲突，会放弃修改。

注意：PodPreset不会影响pod的控制器（如deployment），会被影响的是该deployment创建的pod。

# Deployment
Deployment实现了kubernetes中pod的水平扩展/收缩（horizontal scaling out/in）功能
Deployment升级现有容器需要遵循“滚动更新”（rolling update）方式。
滚动更新的实现依赖于ReplicaSet。
从ReplicaSet的yaml文件可以看出它是Deployment的一个子集。Deployment实际操控的是ReplicaSet对象，再由ReplicaSet操控Pod。

Deployment只允许容器的restartPolicy=Always。因为只有容器能保证自己始终是在running状态下，ReplicaSet调整Pod个数才有意义。

Deployment创建后的状态信息：
1. DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；
2. CURRENT：当前处于 Running 状态的 Pod 的个数；
3. UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分
与 Deployment 里 Pod 模板里定义的完全一致；
4. AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并
且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

kubectl rollout status xxdeployment  这条命令可以实时查看Deployment对象的状态
示例：
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...      （表示已经有2个pod更新为最新版本，即进入up-to-date状态）
deployment.apps/nginx-deployment successfully rolled out

可以通过kubectl edit 指令直接修改Etcd中的api对象
$ kubectl edit deployment/nginx-deployment
...
    spec:
        containers:
        - name: nginx
            image: nginx:1.9.1 # 1.7.9 -> 1.9.1
            ports:
            - containerPort: 80
...
deployment.extensions/nginx-deployment edited
kubectl edit实际上将对应的api对象的内容下载到本地，等用户修改完再提交修改

当Deployment中的pod定义发生修改后，Deployment会先创建一个新的ReplicaSet。
每当新的ReplicaSet创建出一个新的pod（先），旧的ReplicaSet就会删除一个旧的pod（后），直到旧的pod没有了，pod的版本更新过程就完成了，旧的ReplicaSet会被保留（方便版本回滚）。
这一更新过程即“滚动更新”

滚动更新的优势在于当第一个新pod创建出了问题时，仍有旧的pod在提供服务，停止滚动更新，由开发、运维人员介入解决问题。

为了保证服务的连续性，Deployment Controller会确保在任意时间窗口内，只有指定比例的Pod处于离线状态，只有指定比例的Pod被新建。该比例值可配置，默认为desired的25%
通过Deployment的RollingUpdateStrategy字段设置
示例：
apiVersion: apps/v1
kind: Deployment
metadata:
    name: nginx-deployment
    labels:
        app: nginx
spec:
...
    strategy:
        type: RollingUpdate
        rollingUpdate:
            maxSurge: 1
            maxUnavailable: 1

maxSurge表示一次滚动中可创建的新pod的数量
maxUnavailable表示一次滚动中可删除的旧pod的数量
这2个配置可以用百分比形式配置，如maxSurge=25%


每更新一个版本，Deployment都创建一个ReplicaSet，旧的ReplicaSet不会被删除，用于版本回归时使用。
可以通过Deployment的spec.revisionHistoryLimit字段限制保留的版本个数（即ReplicaSet个数）

## 版本回滚
可以通过kubectl rollout history命令查看Deployment每次更新的版本。如果创建Deployment时使用了“-record”参数，还会显示每个版本对应的修改命令
示例：
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION CHANGE-CAUSE
1         kubectl create -f nginx-deployment.yaml --record
2         kubectl edit deployment/nginx-deployment
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.91

还可以查看版本的具体api对象细节
示例：
$ kubectl rollout history deployment/nginx-deployment --revision=2

通过kubectl rollout undo命令回滚到指定版本
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2

如果希望执行一连串的修改，但只进行一次滚动更新（只生成一个ReplicaSet），可以先用kubectl rollout pause命令使Deployment进入暂停状态，执行了一连串修改后再执行kubectl rollout resume，使Deployment恢复，这样在pause和resume命令之间的所有操作都只会进行一次滚动更新
示例：
$ kubectl rollout pause deployment/nginx-deployment
$ kubectl rollout resume deploy/nginx-deployment

注：kubectl create xxxx  xxxx.yaml --record    其中"-record"参数的作用是记录每次操作所执行的命令

# 控制器
kube-controller-manager组件是一系列控制器的集合
可以在kubernetes的pkg/controller目录下查看各种控制器

所有控制器都遵循kubernetes项目的一个通用编排模式，控制循环（control loop）。
调谐（reconcile）、reconcile loop（调谐循环）、sync loop（同步循环）都是指控制循环

go语言风格的控制循环伪代码：
for {
    实际状态 := 获取集群中对象 X 的实际状态（Actual State）
    期望状态 := 获取集群中对象 X 的期望状态（Desired State）
    if 实际状态 == 期望状态{
        什么都不做
    } else {
        执行编排动作，将实际状态调整为期望状态
    }
}

被控制器管理的pod都是根据控制器中的template字段的内容创建出来的

# StatefulSet
deployment中管理的pod是无状态的
StatefulSet支持有状态应用

StatefulSet将应用状态抽象为2种：
1. 拓扑状态。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
2. 存储状态。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

## Headless Service
访问Service的方法：
1.直接以Service的VIP（virtual ip，由kubernetes分配的虚拟ip）访问Service，再由Service将请求转发给它代理的pod
2.以Service的DNS，通过访问某条DNS记录，访问到Service所代理的某个pod

其中Service的DNS方式分2种：
1.Normal Service。通过访问DNS记录解析到Service的VIP，再以VIP的方式访问pod
2.Headless Service。通过访问DNS记录直接解析到Service代理的pod的IP进行访问。即Headless Service不需要分配VIP。

Headless Service所代理的所有pod的DNS记录格式为：
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
这个DNS是Kubernetes为pod分配的唯一的可解析身份（Resolvable Identity）。

StatefulSet会为其管理的每个pod进行编号，以“-x”（x为编号，从0开始）格式编号，并且创建或者重建pod的顺序也会严格按照编号顺序进行创建或重建，只有上一个pod创建成功进入ready状态，才会创建下一个pod。
而且重建时，会为pod保留同样的网络标识（DNS），虽然创建的pod的DNS不会变，但其分配的IP地址会出现变化。因此，在访问有状态应用时，应通过hostname或DNS进行访问，不能用IP访问

## PV/PVC
### PVC
Persistent Volume Claim
当开发人员想要定义一个Volume，但一般情况下开发人员并不熟悉Volume的所有设置，且一般不会将所有设置都向开发人员暴露（如重要数据库具体存储位置、授权文件等）时，可以使用PVC。
只需要在PVC中进行一些基础定义，再在pod中声明使用该PVC。Kubernetes就会为这个PVC绑定一个合适的PV。

示例：
PVC.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
    name: pv-claim
spec:
    accessModes:
    - ReadWriteOnce
    resources:
        requests:
        storage: 1Gi
该PVC中只定义了accessModes:ReadWriteOnce（可读写且只能被一个节点挂载）和storage: 1Gi（1GB空间）

pod.yaml
apiVersion: v1
kind: Pod
metadata:
    name: pv-pod
spec:
...
    volumes:
        - name: pv-storage
            persistentVolumeClaim:
                claimName: pv-claim

### PV
Persistent Volume
PV的yaml文件就是一个Volume的完整定义，一般由运维人员负责

PV和PVC的关系可以看作是实现和接口的关系，开发人员只需要在PVC中定义相关基础设置，kubernetes就会为PVC绑定适用的PV，结合PVC和PV中的设置实现一个Volume
从避免了向开发暴露过多信息的隐患

当StatefulSet管理的pod重建之后，会自动绑定原来对应的PVC，进而找到该PVC对应的PV,从而获得之前存储的数据。