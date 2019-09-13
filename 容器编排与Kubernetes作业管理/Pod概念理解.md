## Pod概念理解

通过前面的学习我们知道，其实容器本质上是一个进程，是一个特殊的进程。  
其与其他容器通过Linux Namespace进行隔离，通过Cgroup对进程的资源进行限制  
同时将基础镜像的rootfs挂载到容器进程对应的根目录下，由于mount Namespace（修改进程的文件视图），所以在对应的进程里看到的就是挂载后的文件视图。    
	
**pod其实就是容器组，为什么会有容器组的概念？**   
	
我们考虑一个场景，当我们的业务需求需要我们将两个容器调度在同一个节点上时，   
比如容器1产生内容写入某个文件，容器2从对应文件读取内容进行显示。再比如，如果两个容器需要使用localhost通信或者socket通信。    
	
从上面可以看出这种关系比较密切的容器其实是需要共享一些namespace的，比如网络、mount等，同时我们能需要一种机制保证这些容器可以调度到同一个节点。其实这就是pod的本质，pod是一个逻辑概念，同一个pod中的容器共享了mount、network、uts namespace，并且pod内容器可以保证调度到同一个节点。    
	

> 如何保证两个容器调度到同一个节点？很直观的一个想法就是在同一个节点上运行命令  
> docker run --net=B --volumes-from=B --name=A --image =A  
> 	
> 但是这样同样会有问题，那就是本质上AB两个容器是对等的，现在就导致AB启动会有先后顺序，这样就成了一个拓扑结构  
> 	
> Kubenetes采用了如下的处理方式，每次调度一个pod到对应的node时，会先启动一个infra容器，infra容器是一个很简单的容器，其基础镜像是由汇编语言写的，大小也不过只有几十K，目的就是为pod提供linux namespace的环境，这样pod中其他容器就可以通过如上的方式共享infra容器的namespace，pod生命周期只与infra容器有关，并且我们可以理解为pod的流量进出都是通过infra容器。

​	

考虑两个场景实例：

- 一个java web的jar包，想要放在tomcat的/webapps目录下运行，一个解决思路是我们把jar包放在tomcat对应路径下，出一个新的镜像，但是这种其实是比较繁琐的，需要重新打包出镜像。另外一个解决思路是在一个pod中运行两个容器，java web容器启动的时候执行一个command，将jar包拷贝到某个路径下；tomcat容器可以从/webapp路径下读取对应的jar包。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: javaweb-2
  spec:
    initContainers: 
    - image: geektime/sample:v2
      name: war
      command: ["cp", "/sample.war", "/app"] 
      volumeMounts:
      - mountPath: /app
        name: app-volume
    containers:
    - image: geektime/tomcat:7.0
      name: tomcat
      command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
      volumeMounts:
      - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
        name: app-volume
      ports:
      - containerPort: 8080
        hostPort: 8001 
    volumes:
    - name: app-volume
      emptyDir: {}
  
  ```
  
  
  
  有几个注意的地方：
  
  1. initContainers 定义的是init容器，其按照顺序执行，且执行在container容器前，在其容器启动并且退出后，才会执行container容器
  2. 我们定义了一个emptyDir的卷，分别将这个卷挂载到两个容器的不同路径下，首先执行initContainer将jar包拷贝到对应路径的卷中；由于同一pod中容器共享volumes 的mount namespace，这样在两个容器中对应路径下看到的卷的内容是相同的

- 容器日志收集

  之前听阿里云分享的时候讲到阿里容器目前的日志采集方案其实是两种方案都存在的，就是Daemonset和sidecar。我们这里主要说sidecar，sidecar日志采集其实就是对每个需要采集日志的应用pod中都同时部署一个日志采集的容器，二者共享mount卷中数据，应用容器将日志写入/var/log，volume挂载到对应容器的/var/log路径，日志采集容器挂载卷到某个路径，同时从该路径定时获取日志写入es。



​		我们前面说过同一个pod中两个容器共享network命名空间，那么istio其实就是这个方面的应用，todo...

​		

​		