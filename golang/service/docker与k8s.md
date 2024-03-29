## docker
docker是一个Client-Server结构的系统，docker守护进程运行在宿主机上，守护进程从客户端接受命令并管理运行在主机上的容器，容器是一个运行时环境。

### 组成
- docker client，客户端，为用户提供一系列可执行命令，用户用这些命令实现跟 docker daemon 交互；
- docker daemon，守护进程，一般在宿主主机后台运行，等待接收来自客户端的请求消息；
- docker image，镜像，镜像run之后就生成为docker容器；
- docker container，容器，一个系统级别的服务，拥有自己的ip和系统目录结构；运行容器前需要本地存在对应的镜像，如果本地不存在该镜像则就去镜像仓库下载。

docker 使用客户端-服务器 (C/S) 架构模式，使用远程api来管理和创建docker容器。docker 容器通过 docker 镜像来创建。容器与镜像的关系类似于面向对象编程中的对象与类。

### 核心概念
- 镜像：镜像是一种轻量级、可执行的独立软件包，它包含运行某个软件所需的所有内容，我们把应用程序和配置依赖打包好形成一个可交付的运行环境(包括代码、运行时需要的库、环境变量和配置文件等)，这个打包好的运行环境就是image镜像文件。
- 容器：容器是基于镜像创建的，是镜像运行起来之后的一个实例，容器才是真正运行业务程序的地方。如果把镜像比作程序里面的类，那么容器就是对象。
- 镜像仓库：存放镜像的地方，研发工程师打包好镜像之后需要把镜像上传到镜像仓库中去，然后就可以运行有仓库权限的人拉取镜像来运行容器了。

### docker与虚拟机区别
1. 传统虚拟机是需要安装整个操作系统的，然后再在上面安装业务应用，启动应用，通常需要几分钟去启动应用，而docker是直接使用镜像来运行业务容器的，其容器启动属于秒级别；
2. Docker需要的资源更少，Docker在操作系统级别进行虚拟化，Docker容器和内核交互，几乎没有性能损耗，而虚拟机运行着整个操作系统，占用物理机的资源就比较多;
3. Docker更轻量，Docker的架构可以共用一个内核与共享应用程序库，所占内存极小;同样的硬件环境，Docker运行的镜像数远多于虚拟机数量，对系统的利用率非常高;
4. 与虚拟机相比，Docker隔离性更弱，Docker属于进程之间的隔离，虚拟机可实现系统级别隔离;
5. Docker的安全性也更弱，Docker的租户root和宿主机root相同，一旦容器内的用户从普通用户权限提升为root权限，它就直接具备了宿主机的root权限，进而可进行无限制的操作。虚拟机租户root权限和宿主机的root虚拟机权限是分离的，并且虚拟机利用如Intel的VT-d和VT-x的ring-1硬件隔离技术，这种技术可以防止虚拟机突破和彼此交互，而容器至今还没有任何形式的硬件隔离;
6. Docker的集中化管理工具还不算成熟，各种虚拟化技术都有成熟的管理工具，比如：VMware vCenter提供完备的虚拟机管理能力;
7. Docker对业务的高可用支持是通过快速重新部署实现的，虚拟化具备负载均衡，高可用、容错、迁移和数据保护等经过生产实践检验的成熟保障机制，Vmware可承诺虚拟机99.999%高可用，保证业务连续性;
8. 虚拟化创建是分钟级别的，Docker容器创建是秒级别的，Docker的快速迭代性，决定了无论是开发、测试、部署都可以节省大量时间;
9. 虚拟机可以通过镜像实现环境交付的一致性，但镜像分发无法体系化，Docker在Dockerfile中记录了容器构建过程，可在集群中实现快速分发和快速部署。

### 构建镜像过程
1. 创建一个目录用于存放应用程序以及构建过程中使用到的各个文件等；
2. 在这个目录下创建一个Dockerfile文件，一般建议Dockerfile的文件名就是Dockerfile；
3. 编写Dockerfile文件，编写指令，如，使用FORM指令指定基础镜像，COPY指令复制文件，RUN指令指定要运行的命令，ENV设置环境变量，EXPOSE指定容器要暴露的端口，WORKDIR设置当前工作目录，CMD容器启动时运行命令，等等指令构建镜像；
4. Dockerfile编写完成就可以构建镜像了，使用docker build -t 镜像名:tag . 命令来构建镜像，最后一个点是表示当前目录，docker会默认寻找当前目录下的Dockerfile文件来构建镜像，如果不使用默认，可以使用-f参数来指定dockerfile文件，如：docker build -t 镜像名:tag -f /xx/xxx/Dockerfile ；
5. 使用docker build命令构建之后，docker就会将当前目录下所有的文件发送给docker daemon，顺序执行Dockerfile文件里的指令，在这过程中会生成临时容器，在临时容器里面安装RUN指定的命令，安装成功后，docker底层会使用类似于docker commit命令来将容器保存为镜像，然后删除临时容器，以此类推，一层层的构建镜像，运行临时容器安装软件，直到最后的镜像构建成功。

### 镜像构建异常问题排查
Dockerfile是一层一层的构建镜像，期间会产生一个或多个临时容器，构建过程中其实就是在临时容器里面安装应用，如果因为临时容器安装应用出现异常导致镜像构建失败，这时容器虽然被清理掉了，但是期间构建的中间镜像还在，那么我们可以根据异常时上一层已经构建好的临时镜像，将临时镜像运行为容器，然后在容器里面运行安装命令来定位具体的异常。

### 基本命令
FROM 指定基础镜像（必须为第一个指令，因为需要指定使用哪个基础镜像来构建镜像）；
MAINTAINER 设置镜像作者相关信息，如作者名字，日期，邮件，联系方式等；
COPY 复制文件到镜像；
ADD 复制文件到镜像（ADD与COPY的区别在于，ADD会自动解压tar、zip、tgz、xz等归档文件，而COPY不会，同时ADD指令还可以接一个url下载文件地址，一般建议使用COPY复制文件即可，文件在宿主机上是什么样子复制到镜像里面就是什么样子这样比较好）；
ENV 设置环境变量；
EXPOSE 暴露容器进程的端口，仅仅是提示别人容器使用的哪个端口，没有过多作用；
VOLUME 数据卷持久化，挂载一个目录；
WORKDIR 设置工作目录，如果目录不在，则会自动创建目录；
RUN 在容器中运行命令，RUN指令会创建新的镜像层，RUN指令经常被用于安装软件包；
CMD 指定容器启动时默认运行哪些命令，如果有多个CMD，则只有最后一个生效，另外，CMD指令可以被docker run之后的参数替换；
ENTRYOINT 指定容器启动时运行哪些命令，如果有多个ENTRYOINT，则只有最后一个生效，另外，如果Dockerfile中同时存在CMD和ENTRYOINT，那么CMD或docker run之后的参数将被当做参数传递给ENTRYOINT；

进入容器：docker attach、docker exec；
- docker attach命令是attach到容器启动命令的终端，docker exec 是另外在容器里面启动一个新的终端。
- attach是直接进入容器启动命令的终端，不会启动新的进程；exec则是在容器里面打开新的终端，会启动新的进程；一般建议已经exec进入容器。

## k8s
k8s是kubernetes的简称，其本质是一个开源的容器编排系统，主要用于管理容器化的应用，其目标是让部署容器化的应用简单并且高效,Kubernetes提供了应用部署，规划，更新，维护的一种机制。

k8s就是一个编排容器的系统，一个可以管理容器应用全生命周期的工具，从创建应用，应用的部署，应用提供服务，扩容缩容应用，应用更新，都非常的方便，而且还可以做到故障自愈，所以，k8s是一个非常强大的容器编排系统。

### 组件
k8s主要由master节点和node节点构成。master节点负责管理集群，node节点是容器应用真正运行的地方。
- master节点包含的组件有：kube-api-server、kube-controller-manager、kube-scheduler、etcd。
- node节点包含的组件有：kubelet、kube-proxy、container-runtime。

**kube-api-server**：以下简称api-server，api-server是k8s最重要的核心组件之一，它是k8s集群管理的统一访问入口，提供了RESTful API接口, 实现了认证、授权和准入控制等安全功能；api-server还是其他组件之间的数据交互和通信的枢纽，其他组件彼此之间并不会直接通信，其他组件对资源对象的增、删、改、查和监听操作都是交由api-server处理后，api-server再提交给etcd数据库做持久化存储，只有api-server才能直接操作etcd数据库，其他组件都不能直接操作etcd数据库，其他组件都是通过api-server间接的读取，写入数据到etcd。

**kube-controller-manager**：以下简称controller-manager，controller-manager是k8s中各种控制器的的管理者，是k8s集群内部的管理控制中心，也是k8s自动化功能的核心；controller-manager内部包含replication controller、node controller、deployment controller、endpoint controller等各种资源对象的控制器，每种控制器都负责一种特定资源的控制流程，而controller-manager正是这些controller的核心管理者。

**kube-scheduler**：以下简称scheduler，scheduler负责集群资源调度，其作用是将待调度的pod通过一系列复杂的调度算法计算出最合适的node节点，然后将pod绑定到目标节点上。

**etcd**：etcd是一个分布式的键值对存储数据库，主要是用于保存k8s集群状态数据，比如，pod，service等资源对象的信息；etcd可以是单个也可以有多个，多个就是etcd数据库集群，etcd通常部署奇数个实例，在大规模集群中，etcd有5个或7个节点就足够了；另外说明一点，etcd本质上可以不与master节点部署在一起，只要master节点能通过网络连接etcd数据库即可。

**kubelet**：每个node节点上都有一个kubelet服务进程，kubelet作为连接master和各node之间的桥梁，负责维护pod和容器的生命周期，当监听到master下发到本节点的任务时，比如创建、更新、终止pod等任务，kubelet 即通过控制docker来创建、更新、销毁容器；
每个kubelet进程都会在api-server上注册本节点自身的信息，用于定期向master汇报本节点资源的使用情况。

**kube-proxy**：kube-proxy运行在node节点上，在Node节点上实现Pod网络代理，维护网络规则和四层负载均衡工作，kube-proxy会监听api-server中从而获取service和endpoint的变化情况，创建并维护路由规则以提供服务IP和负载均衡功能。简单理解此进程是Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上。

**container-runtime**：容器运行时环境，即运行容器所需要的一系列程序，目前k8s支持的容器运行时有很多，如docker、rkt或其他，比较受欢迎的是docker，但是新版的k8s已经宣布弃用docker。

### 命名空间
namespace是kubernetes系统中的一种非常重要的资源，namespace的主要作用是用来实现多套环境的资源隔离，或者说是多租户的资源隔离。

k8s通过将集群内部的资源分配到不同的namespace中，可以形成逻辑上的隔离，以方便不同的资源进行隔离使用和管理。不同的命名空间可以存在同名的资源，命名空间为资源提供了一个作用域。

可以通过k8s的授权机制，将不同的namespace交给不同的租户进行管理，这样就实现了多租户的资源隔离，还可以结合k8s的资源配额机制，限定不同的租户能占用的资源，例如CPU使用量、内存使用量等等来实现租户可用资源的管理。

### pod
在kubernetes的世界中，k8s并不直接处理容器，而是使用多个容器共存的理念，这组容器就叫做pod。pod是k8s中可以创建和管理的最小单元，是资源对象模型中由用户创建或部署的最小资源对象模型，其他的资源对象都是用来支撑pod对象功能的，比如，pod控制器就是用来管理pod对象的，service或者imgress资源对象是用来暴露pod引用对象的，persistentvolume资源是用来为pod提供存储等等，简而言之，k8s不会直接处理容器，而是pod，pod才是k8s中可以创建和管理的最小单元，也是基本单元。

#### 原理
在微服务的概念里，一般的，一个容器会被设计为运行一个进程，除非进程本身产生子进程，这样，由于不能将多个进程聚集在同一个单独的容器中，所以需要一种更高级的结构将容器绑定在一起，并将它们作为一个单元进行管理，这就是k8s中pod的背后原理。

#### 特点
- 每个pod就像一个独立的逻辑机器，k8s会为每个pod分配一个集群内部唯一的IP地址，所以每个pod都拥有自己的IP地址、主机名、进程等；
- 一个pod可以包含1个或多个容器，1个容器一般被设计成只运行1个进程，1个pod只可能运行在单个节点上，即不可能1个pod跨节点运行，pod的生命周期是短暂，也就是说pod可能随时被消亡（如节点异常，pod异常等情况）；
- 每一个pod都有一个特殊的被称为"根容器"的pause容器，也称info容器，pause容器对应的镜像属于k8s平台的一部分，除了pause容器，每个pod还包含一个或多个跑业务相关组件的应用容器；
- 一个pod中的容器共享network命名空间；
- 一个pod里的多个容器共享pod IP，这就意味着1个pod里面的多个容器的进程所占用的端口不能相同，否则在这个pod里面就会产生端口冲突；既然每个pod都有自己的IP和端口空间，那么对不同的两个pod来说就不可能存在端口冲突；
- 应该将应用程序组织到多个pod中，而每个pod只包含紧密相关的组件或进程；
- pod是k8s中扩容、缩容的基本单位，也就是说k8s中扩容缩容是针对pod而言而非容器。

#### pod创建过程
1. 用户通过kubectl或其他api客户端工具提交需要创建的pod信息给apiserver；
2. apiserver开始生成pod对象信息，并将信息存入etcd，然后返回确认信息给客户端；
3. apiserver开始反馈etcd中pod对象的变化，其他组件使用watch机制跟踪apiserver上的变动；
4. scheduler发现有新的pod对象要创建，开始调用内部算法机制为pod分配主机，并将结果信息更新至apiserver；
5. node节点上的kubectl发现有pod调度过来，尝试调用docker启动容器，并将结果反馈apiserver；
6. apiserver将收到的pod状态信息存入etcd中。

#### pod终止过程
1. 用户向apiserver发送删除pod对象的命令；
2. apiserver中的pod对象信息会随着时间的推移而更新，在宽限期内（默认30s），pod被视为dead；
3. 将pod标记为terminating状态；
4. kubectl在监控到pod对象为terminating状态了就会启动pod关闭过程；
5. endpoint控制器监控到pod对象的关闭行为时将其从所有匹配到此endpoint的server资源endpoint列表中删除；
6. 如果当前pod对象定义了preStop钩子处理器，则在其被标记为terminating后会意同步的方式启动执行；
7. pod对象中的容器进程收到停止信息；
8. 宽限期结束后，若pod中还存在运行的进程，那么pod对象会收到立即终止的信息；
9. kubelet请求apiserver将此pod资源的宽限期设置为0从而完成删除操作，此时pod对用户已不可见。

#### pod生命周期
- Pending（挂起）：API server已经创建pod，但是该pod还有一个或多个容器的镜像没有创建，包括正在下载镜像的过程；
- Running（运行中）：Pod内所有的容器已经创建，且至少有一个容器处于运行状态、正在启动括正在重启状态；
- Succeed（成功）：Pod内所有容器均已退出，且不会再重启；
- Failed（失败）：Pod内所有容器均已退出，且至少有一个容器为退出失败状态
- Unknown（未知）：某于某种原因apiserver无法获取该pod的状态，可能由于网络通行问题导致；

#### pod钩子函数
钩子函数能够感知自身生命周期中的时间，并在相应的时刻到来时运行指定的程序代码。

- PostStart：这个钩子在容器创建后立即执行，如果失败了会重启容器。但是，并不能保证钩子将在容器ENTRYPOINT之前运行，因为没有参数传递给处理程序。主要用于资源部署、环境准备等。不过需要注意的是如果钩子花费太长时间以至于不能运行或者挂起， 容器将不能达到running状态。
- PreStop：这个钩子在容器终止之前立即被调用，执行完成后容器将会成功终止，该钩子函数会在命令执行完成之前会阻塞删除容器的操作。主要用于优雅关闭应用程序、通知其他系统等。如果钩子在执行期间挂起， Pod阶段将停留在running状态并且永不会达到failed状态。

#### pod初始化容器
init container，初始化容器用于在启动应用容器之前完成应用容器所需要的前置条件，初始化容器本质上和应用容器是一样的，但是初始化容器是仅允许一次就结束的任务，初始化容器具有两大特征：
- 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么kubernetes需要重启它直到成功完成；
- 初始化容器必须按照定义的顺序执行，当且仅当前一个初始化容器成功之后，后面的一个初始化容器才能运行；

#### pause容器
每个pod里运行着一个特殊的被称之为pause的容器，也称根容器，而其他容器则称为业务容器；创建pause容器主要是为了为业务容器提供 Linux命名空间，共享基础：包括 pid、icp、net 等，以及启动 init 进程，并收割僵尸进程；这些业务容器共享pause容器的网络命名空间和volume挂载卷，当pod被创建时，pod首先会创建pause容器，从而把其他业务容器加入pause容器，从而让所有业务容器都在同一个命名空间中，这样可以就可以实现网络共享。pod还可以共享存储，在pod级别引入数据卷volume，业务容器都可以挂载这个数据卷从而实现持久化存储。

#### 重启策略
pod重启容器策略是指针对pod内所有容器的重启策略，不是重启pod，其可以通过restartPolicy字段配置pod重启容器的策略

- Always: 当容器终止退出后，总是重启容器，默认策略就是Always。
- OnFailure: 当容器异常退出，退出状态码非0时，才重启容器。
- Never: 当容器终止退出，不管退出状态码是什么，从不重启容器。

#### 镜像拉取策略
pod镜像拉取策略可以通过imagePullPolicy字段配置镜像拉取策略

- IfNotPresent: 默认值，镜像在node节点宿主机上不存在时才拉取。
- Always: 总是重新拉取，即每次创建pod都会重新从镜像仓库拉取一次镜像。
- Never: 永远不会主动拉取镜像，仅使用本地镜像，需要你手动拉取镜像到node节点，如果node节点不存在镜像则pod启动失败。

#### 存活探针类型
kubernetes可以通过存活探针检查容器是否还在运行，可以为pod中的每个容器单独定义存活探针，kubernetes将定期执行探针，如果探测失败，将杀死容器，并根据restartPolicy策略来决定是否重启容器

- httpGet：根据容器的IP、端口、路径发送http请求，返回200-400范围内的状态码表示成功。
- exec：在容器内执行shell命令，根据命令退出状态码是否为0判定成功失败。
- tcpSocket：与容器的端口建立TCP Socket链接。

#### 存活探针属性
- initialDelaySeconds：表示在容器启动后延时多久秒才开始探测；
periodSeconds：表示执行探测的频率，即间隔多少秒探测一次，默认间隔周期是10秒，最小1秒；
- timeoutSeconds：表示探测超时时间，默认1秒，最小1秒，表示容器必须在超时时间范围内做出响应，否则视为本次探测失败；
- successThreshold：表示最少连续探测成功多少次才被认定为成功，默认是1，对于liveness必须是1，最小值是1；
- failureThreshold：表示连续探测失败多少次才被认定为失败，默认是3，连续3次失败，k8s 将根据pod重启策略对容器做出决定；

注意：定义存活探针时，一定要设置initialDelaySeconds属性，该属性为初始延时，如果不设置，默认容器启动时探针就开始探测了，这样可能会存在应用程序还未启动就绪，就会导致探针检测失败，k8s就会根据pod重启策略杀掉容器然后再重新创建容器的莫名其妙的问题。
**在生产环境中，一定要定义一个存活探针。**

#### 就绪探针类型
当一个pod启动后，就会立即加入service的endpoint ip列表中，并开始接收到客户端的链接请求，假若此时pod中的容器的业务进程还没有初始化完毕，那么这些客户端链接请求就会失败，为了解决这个问题，kubernetes提供了就绪探针来解决这个问题的。

在pod中的容器定义一个就绪探针，就绪探针周期性检查容器，如果就绪探针检查失败了，说明该pod还未准备就绪，不能接受客户端链接，则该pod将从endpoint列表中移除，被剔除了service就不会把请求分发给该pod，然后就绪探针继续检查，如果随后容器就绪，则再重新把pod加回endpoint列表

- exec：执行容器中的命令并检查命令退出的状态码，如果状态码为0则说明容器已经准备就绪；
- httpGet：向容器发送http get请求，通过响应的http状态码判断容器是否准备就绪；
- tcpSocket：打开一个tcp连接到容器的指定端口，如果连接已建立，则认为容器已经准备就绪。

#### 就绪探针属性
- initialDelaySeconds：延时秒数，即容器启动多少秒后才开始探测，不写默认容器启动就探测；
- periodSeconds ：执行探测的频率（秒），默认为10秒，最低值为1；
- timeoutSeconds ：超时时间，表示探测时在超时时间内必须得到响应，负责视为本次探测失败，默认为1秒，最小值为1；
- failureThreshold ：连续探测失败的次数，视为本次探测失败，默认为3次，最小值为1次；
- successThreshold ：连续探测成功的次数，视为本次探测成功，默认为1次，最小值为1次；

#### 存活探针与就绪探针区别
- 存活探针是将检查失败的容器杀死，创建新的启动容器来保持pod正常工作；
- 就绪探针是，当就绪探针检查失败，并不重启容器，而是将pod移出endpoint，就绪探针确保了service中的pod都是可用的，确保客户端只与正常的pod交互并且客户端永远不会知道系统存在问题。

#### 标签及标签选择器
标签是键值对类型，标签可以附加到任何资源对象上，主要用于管理对象，查询和筛选。标签常被用于标签选择器的匹配度检查，从而完成资源筛选；一个资源可以定义一个或多个标签在其上面。

标签选择器，标签要与标签选择器结合在一起，标签选择器允许我们选择标记有特定标签的资源对象子集，如pod，并对这些特定标签的pod进行查询，删除等操作。

**标签和标签选择器最重要的使用之一在于，在deployment中，在pod模板中定义pod的标签，然后在deployment定义标签选择器，这样就通过标签选择器来选择哪些pod是受其控制的，service也是通过标签选择器来关联哪些pod最后其服务后端pod。**
