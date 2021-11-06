## 前言

在大致分析过k8s的Scheduler、Controller、APIServer三个控制平面组件后，本篇开始进入数据交互平面的daemon组件kubelet部分，看看kubelet是如何在控制平面和数据平面中以承上启下的模式工作的。



## 启动流程

启动入口照旧，位于项目的cmd路径下，使用cobra做cmd封装：

`cmd/kubelet/kubelet.go:39`

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewKubeletCommand(server.SetupSignalHandler())
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

`cmd/kubelet/app/server.go:112`

`NewKubeletFlags`和`NewKubeletConfiguration`方法会初始化kubelet的很多默认flag和参数，来分别看下：

`cmd/kubelet/app/options/options.go:214`

```go
func NewKubeletFlags() *KubeletFlags {
	remoteRuntimeEndpoint := ""
	if runtime.GOOS == "linux" {
		remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock"
	} else if runtime.GOOS == "windows" {
		remoteRuntimeEndpoint = "npipe:////./pipe/dockershim"
	}

	return &KubeletFlags{
		EnableServer:                        true,
    // 容器运行时这个参数需要留意下
		ContainerRuntimeOptions:             *NewContainerRuntimeOptions(),
		CertDirectory:                       "/var/lib/kubelet/pki",
		RootDirectory:                       defaultRootDir,
		MasterServiceNamespace:              metav1.NamespaceDefault,
		MaxContainerCount:                   -1,
		MaxPerPodContainerCount:             1,
		MinimumGCAge:                        metav1.Duration{Duration: 0},
		NonMasqueradeCIDR:                   "10.0.0.0/8",
		RegisterSchedulable:                 true,
		ExperimentalKernelMemcgNotification: false,
		RemoteRuntimeEndpoint:               remoteRuntimeEndpoint,
		NodeLabels:                          make(map[string]string),
		VolumePluginDir:                     "/usr/libexec/kubernetes/kubelet-plugins/volume/exec/",
		RegisterNode:                        true,
		SeccompProfileRoot:                  filepath.Join(defaultRootDir, "seccomp"),
		HostNetworkSources:                  []string{kubetypes.AllSource},
		HostPIDSources:                      []string{kubetypes.AllSource},
		HostIPCSources:                      []string{kubetypes.AllSource},
		// TODO(#58010:v1.13.0): Remove --allow-privileged, it is deprecated
		AllowPrivileged: true,
		// prior to the introduction of this flag, there was a hardcoded cap of 50 images
		NodeStatusMaxImages: 50,
	}
}
```

ContainerRuntimeOptions有必要看看：

`cmd/kubelet/app/options/container_runtime.go:41`

```go
func NewContainerRuntimeOptions() *config.ContainerRuntimeOptions {
	dockerEndpoint := ""
	if runtime.GOOS != "windows" {
    // 默认的容器驱动是docker
		dockerEndpoint = "unix:///var/run/docker.sock"
	}

	return &config.ContainerRuntimeOptions{
		ContainerRuntime:           kubetypes.DockerContainerRuntime,
		RedirectContainerStreaming: false,
		DockerEndpoint:             dockerEndpoint,
    // dockershim路径，dockershim是容器运行中的实际载体，每个docker容器都会产生一个shim进程
		DockershimRootDirectory:    "/var/lib/dockershim",
    // pause容器
		PodSandboxImage:            defaultPodSandboxImage,
		ImagePullProgressDeadline:  metav1.Duration{Duration: 1 * time.Minute},
		ExperimentalDockershim:     false,

		// 这个目录下都是网络相关功能工具的执行文件
		CNIBinDir:  "/opt/cni/bin",
    // 这里是cni的配置文件，如pod网段、网关、bridge等，一般由cni动态生成
		CNIConfDir: "/etc/cni/net.d",
	}
}
```

`cmd/kubelet/app/options/options.go:293`

--> `cmd/kubelet/app/options/options.go:311`

```go
// `NewKubeletConfiguration`方法则会默认设置一些参数
func applyLegacyDefaults(kc *kubeletconfig.KubeletConfiguration) {
	// --anonymous-auth
	kc.Authentication.Anonymous.Enabled = true
	// --authentication-token-webhook
	kc.Authentication.Webhook.Enabled = false
	// --authorization-mode
  // apiserver认证篇提到的针对node设计的AlwaysAllow认证模式
	kc.Authorization.Mode = kubeletconfig.KubeletAuthorizationModeAlwaysAllow
	// 10255采集信息的接口，如prometheus采集cadvisor的metrics
	kc.ReadOnlyPort = ports.KubeletReadOnlyPort
}
```



再来看看Run方法里面做了哪些操作:

--> `cmd/kubelet/app/server.go:148`



```go
		Run: func(cmd *cobra.Command, args []string) {
      ...
			// 上面百来行代码都是默认的init flag和config相关处理，例如featureGates等，略过
      
      // 加载kubelet配置文件，展开进去看可以看到即是--config参数对应指定的文件，一般kubeadm部署时使用的是/var/lib/kubelet/config.yaml
			if configFile := kubeletFlags.KubeletConfigFile; len(configFile) > 0 {
				kubeletConfig, err = loadConfigFile(configFile)
				if err != nil {
					klog.Fatal(err)
				}
      ... 
			}
      
			// 实例化KubeletServer
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}

			// 构建一些kubelet的依赖插件，例如nsenter，连接dockershim的client端
			kubeletDeps, err := UnsecuredDependencies(kubeletServer)
			if err != nil {
				klog.Fatal(err)
			}

			// add the kubelet config controller to kubeletDeps
			kubeletDeps.KubeletConfigController = kubeletConfigController

			// start the experimental docker shim, if enabled
			if kubeletServer.KubeletFlags.ExperimentalDockershim {
				if err := RunDockershim(&kubeletServer.KubeletFlags, kubeletConfig, stopCh); err != nil {
					klog.Fatal(err)
				}
				return
			}

			// 启动kubelet
			klog.V(5).Infof("KubeletConfiguration: %#v", kubeletServer.KubeletConfiguration)
			if err := Run(kubeletServer, kubeletDeps, stopCh); err != nil {
				klog.Fatal(err)
			}
		},
	}

	// 下面是一些cmd help信息，省略
  ...

	return cmd
```

--> `cmd/kubelet/app/server.go:416`

--> `cmd/kubelet/app/server.go:479` 这个函数代码段很长，两百多行，挑主要片段

```go
func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies, stopCh <-chan struct{}) (err error) {
 ...

  // 独立模式指的是不与外部(如apiserver)交互的模式，一般在调试中使用，所以独立模式不需要起client
	standaloneMode := true
	if len(s.KubeConfig) > 0 {
		standaloneMode = false
	}

	if kubeDeps == nil {
		kubeDeps, err = UnsecuredDependencies(s)
		if err != nil {
			return err
		}
	}

  // 取得注册node名
	hostName, err := nodeutil.GetHostname(s.HostnameOverride)
	if err != nil {
		return err
	}
	nodeName, err := getNodeName(kubeDeps.Cloud, hostName)
	if err != nil {
		return err
	}

	
	switch {
    // 独立模式，则所有client设为nil
	case standaloneMode:
		kubeDeps.KubeClient = nil
		kubeDeps.EventClient = nil
		kubeDeps.HeartbeatClient = nil
		klog.Warningf("standalone mode, no API client")
	
   // 正常模式，则初始化client，包括kubeClient/eventClient/heartBeatClient
	case kubeDeps.KubeClient == nil, kubeDeps.EventClient == nil, kubeDeps.HeartbeatClient == nil:
    // client的配置，主要是连接apiserver的cert相关的配置，cert文件默认放在/var/lib/kubelet/pki下，如果开启了循环续期证书，则相应的异步进程会从cert manager循环检测和更新证书。其他的配置诸如超时时间，长连接时间等。closeAllConns接收的是一个方法，用来断开连接。
		clientConfig, closeAllConns, err := buildKubeletClientConfig(s, nodeName)
		if err != nil {
			return err
		}
		if closeAllConns == nil {
			return errors.New("closeAllConns must be a valid function other than nil")
		}
		kubeDeps.OnHeartbeatFailure = closeAllConns
		// 构建一个client-go里的clientset实例，访问各个GV和GVR对象使用
		kubeDeps.KubeClient, err = clientset.NewForConfig(clientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet client: %v", err)
		}

		// event事件使用独立的client，与上面的访问GVR使用的client区分开
		eventClientConfig := *clientConfig
		eventClientConfig.QPS = float32(s.EventRecordQPS)
		eventClientConfig.Burst = int(s.EventBurst)
		kubeDeps.EventClient, err = v1core.NewForConfig(&eventClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet event client: %v", err)
		}

		// 再开启一个心跳检测的client
		heartbeatClientConfig := *clientConfig
		heartbeatClientConfig.Timeout = s.KubeletConfiguration.NodeStatusUpdateFrequency.Duration
    // 如果开启了NodeLease(node定期向apiserver汇报运行状态)，那么心跳间隔最大不超过NodeLease duration
		if utilfeature.DefaultFeatureGate.Enabled(features.NodeLease) {
			leaseTimeout := time.Duration(s.KubeletConfiguration.NodeLeaseDurationSeconds) * time.Second
			if heartbeatClientConfig.Timeout > leaseTimeout {
				heartbeatClientConfig.Timeout = leaseTimeout
			}
		}
    // 心跳1次/s
		heartbeatClientConfig.QPS = float32(-1)
		kubeDeps.HeartbeatClient, err = clientset.NewForConfig(&heartbeatClientConfig)
		if err != nil {
			return fmt.Errorf("failed to initialize kubelet heartbeat client: %v", err)
		}
	}
  // 向apiserver发起认证建立会话
	if kubeDeps.Auth == nil {
		auth, err := BuildAuth(nodeName, kubeDeps.KubeClient, s.KubeletConfiguration)
		if err != nil {
			return err
		}
		kubeDeps.Auth = auth
	}
  // 填充cadvisor接口
	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.ContainerRuntime, s.RemoteRuntimeEndpoint)
		kubeDeps.CAdvisorInterface, err = cadvisor.New(imageFsInfoProvider, s.RootDirectory, cadvisor.UsingLegacyCadvisorStats(s.ContainerRuntime, s.RemoteRuntimeEndpoint))
		if err != nil {
			return err
		}
	}

	// Setup event recorder if required.
	makeEventRecorder(kubeDeps, nodeName)

	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			klog.Info("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}
    // /var/lib/kubelt/config.yaml里可以指定，为系统和kube组件指定不同的cgroup，为它们预留资源
    // kubeReserved即为kube组件指定cgroup预留的资源
		kubeReserved, err := parseResourceList(s.KubeReserved)
		if err != nil {
			return err
		}
    // kubeReserved即为宿主机系统进程指定cgroup预留的资源
		systemReserved, err := parseResourceList(s.SystemReserved)
		if err != nil {
			return err
		}
    // 硬驱逐容器的资源阈值
		var hardEvictionThresholds []evictionapi.Threshold
		// If the user requested to ignore eviction thresholds, then do not set valid values for hardEvictionThresholds here.
		if !s.ExperimentalNodeAllocatableIgnoreEvictionThreshold {
			hardEvictionThresholds, err = eviction.ParseThresholdConfig([]string{}, s.EvictionHard, nil, nil, nil)
			if err != nil {
				return err
			}
		}
		experimentalQOSReserved, err := cm.ParseQOSReserved(s.QOSReserved)
		if err != nil {
			return err
		}

		devicePluginEnabled := utilfeature.DefaultFeatureGate.Enabled(features.DevicePlugins)
    
    // 上面的参数汇集起来，初始化容器管理器
		kubeDeps.ContainerManager, err = cm.NewContainerManager(
			kubeDeps.Mounter,
			kubeDeps.CAdvisorInterface,
			cm.NodeConfig{
				RuntimeCgroupsName:    s.RuntimeCgroups,
				SystemCgroupsName:     s.SystemCgroups,
				KubeletCgroupsName:    s.KubeletCgroups,
				ContainerRuntime:      s.ContainerRuntime,
				CgroupsPerQOS:         s.CgroupsPerQOS,
				CgroupRoot:            s.CgroupRoot,
				CgroupDriver:          s.CgroupDriver,
				KubeletRootDir:        s.RootDirectory,
				ProtectKernelDefaults: s.ProtectKernelDefaults,
				NodeAllocatableConfig: cm.NodeAllocatableConfig{
					KubeReservedCgroupName:   s.KubeReservedCgroup,
					SystemReservedCgroupName: s.SystemReservedCgroup,
					EnforceNodeAllocatable:   sets.NewString(s.EnforceNodeAllocatable...),
					KubeReserved:             kubeReserved,
					SystemReserved:           systemReserved,
					HardEvictionThresholds:   hardEvictionThresholds,
				},
				QOSReserved:                           *experimentalQOSReserved,
				ExperimentalCPUManagerPolicy:          s.CPUManagerPolicy,
				ExperimentalCPUManagerReconcilePeriod: s.CPUManagerReconcilePeriod.Duration,
				ExperimentalPodPidsLimit:              s.PodPidsLimit,
				EnforceCPULimits:                      s.CPUCFSQuota,
				CPUCFSQuotaPeriod:                     s.CPUCFSQuotaPeriod.Duration,
			},
			s.FailSwapOn,
			devicePluginEnabled,
			kubeDeps.Recorder)

		if err != nil {
			return err
		}
	}
   
	if err := checkPermissions(); err != nil {
		klog.Error(err)
	}

	utilruntime.ReallyCrash = s.ReallyCrashForTesting

	rand.Seed(time.Now().UnixNano())

	// oom判定器给当前进程设置oom分数，容器内存资源管控的手段就是使用的oom，这里待会儿拎出来单独分析
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		klog.Warning(err)
	}
  // RunKubelet接往下文
	if err := RunKubelet(s, kubeDeps, s.RunOnce); err != nil {
		return err
	}
  // 起一个健康检查的http服务
	if s.HealthzPort > 0 {
		healthz.DefaultHealthz()
		go wait.Until(func() {
			err := http.ListenAndServe(net.JoinHostPort(s.HealthzBindAddress, strconv.Itoa(int(s.HealthzPort))), nil)
			if err != nil {
				klog.Errorf("Starting health server failed: %v", err)
			}
		}, 5*time.Second, wait.NeverStop)
	}

	if s.RunOnce {
		return nil
	}

	// If systemd is used, notify it that we have started
	go daemon.SdNotify(false, "READY=1")

	select {
	case <-done:
		break
	case <-stopCh:
		break
	}

	return nil
}

```

### OOMAdjuster

--> `pkg/util/oom/oom.go:22`上面提到的oom判定器，这里分析一下，这个结构体有三个方法:

```go
// 这里目前用的还是结构体，看todo描述是后面要改成interface
// TODO: make this an interface, and inject a mock ioutil struct for testing.
type OOMAdjuster struct {
	pidLister                 func(cgroupName string) ([]int, error)
	ApplyOOMScoreAdj          func(pid int, oomScoreAdj int) error
	ApplyOOMScoreAdjContainer func(cgroupName string, oomScoreAdj, maxTries int) error
}

```

--> `pkg/util/oom/oom_linux.go:35`实现方法

```go
func NewOOMAdjuster() *OOMAdjuster {
	oomAdjuster := &OOMAdjuster{
		pidLister:        getPids,
		ApplyOOMScoreAdj: applyOOMScoreAdj,
	}
	oomAdjuster.ApplyOOMScoreAdjContainer = oomAdjuster.applyOOMScoreAdjContainer
	return oomAdjuster
}
// 获取cgroup下所有进程的pid
func getPids(cgroupName string) ([]int, error) {
	return cmutil.GetPids(filepath.Join("/", cgroupName))
}

// 修改oom分数，在linux下即是修改/proc/<pid>/oom_score_adj对应的值，当内存紧张时由linux系统的oom机制去杀掉oom score最高的进程，默认情况下是使用内存越多的进程oom score越高越容易被kill，applyOOMScoreAdj函数就是用来修改oom score的。

// Writes 'value' to /proc/<pid>/oom_score_adj. PID = 0 means self
// Returns os.ErrNotExist if the `pid` does not exist.
func applyOOMScoreAdj(pid int, oomScoreAdj int) error {
	if pid < 0 {
		return fmt.Errorf("invalid PID %d specified for oom_score_adj", pid)
	}

	var pidStr string
	if pid == 0 {
		pidStr = "self"
	} else {
		pidStr = strconv.Itoa(pid)
	}

	maxTries := 2
	oomScoreAdjPath := path.Join("/proc", pidStr, "oom_score_adj")
	value := strconv.Itoa(oomScoreAdj)
	klog.V(4).Infof("attempting to set %q to %q", oomScoreAdjPath, value)
	var err error
	for i := 0; i < maxTries; i++ {
		err = ioutil.WriteFile(oomScoreAdjPath, []byte(value), 0700)
		if err != nil {
			if os.IsNotExist(err) {
				klog.V(2).Infof("%q does not exist", oomScoreAdjPath)
				return os.ErrNotExist
			}

			klog.V(3).Info(err)
			time.Sleep(100 * time.Millisecond)
			continue
		}
		return nil
	}
	if err != nil {
		klog.V(2).Infof("failed to set %q to %q: %v", oomScoreAdjPath, value, err)
	}
	return err
}

// 修改整个容器的oom评分，即修改某个cgroup下所有进程的评分，getPids取得所有pid遍历执行applyOOMScoreAdj
// Writes 'value' to /proc/<pid>/oom_score_adj for all processes in cgroup cgroupName.
// Keeps trying to write until the process list of the cgroup stabilizes, or until maxTries tries.
func (oomAdjuster *OOMAdjuster) applyOOMScoreAdjContainer(cgroupName string, oomScoreAdj, maxTries int) error {
	adjustedProcessSet := make(map[int]bool)
	for i := 0; i < maxTries; i++ {
		continueAdjusting := false
		pidList, err := oomAdjuster.pidLister(cgroupName)
		if err != nil {
			if os.IsNotExist(err) {
				// Nothing to do since the container doesn't exist anymore.
				return os.ErrNotExist
			}
			continueAdjusting = true
			klog.V(10).Infof("Error getting process list for cgroup %s: %+v", cgroupName, err)
		} else if len(pidList) == 0 {
			klog.V(10).Infof("Pid list is empty")
			continueAdjusting = true
		} else {
			for _, pid := range pidList {
				if !adjustedProcessSet[pid] {
					klog.V(10).Infof("pid %d needs to be set", pid)
					if err = oomAdjuster.ApplyOOMScoreAdj(pid, oomScoreAdj); err == nil {
						adjustedProcessSet[pid] = true
					} else if err == os.ErrNotExist {
						continue
					} else {
						klog.V(10).Infof("cannot adjust oom score for pid %d - %v", pid, err)
						continueAdjusting = true
					}
					// Processes can come and go while we try to apply oom score adjust value. So ignore errors here.
				}
			}
		}
		if !continueAdjusting {
			return nil
		}
		// There's a slight race. A process might have forked just before we write its OOM score adjust.
		// The fork might copy the parent process's old OOM score, then this function might execute and
		// update the parent's OOM score, but the forked process id might not be reflected in cgroup.procs
		// for a short amount of time. So this function might return without changing the forked process's
		// OOM score. Very unlikely race, so ignoring this for now.
	}
	return fmt.Errorf("exceeded maxTries, some processes might not have desired OOM score")
}
```



### RunKubelet

--> `cmd/kubelet/app/server.go:955` 回到主线

```go
func RunKubelet(kubeServer *options.KubeletServer, kubeDeps *kubelet.Dependencies, runOnce bool) error {
	...
  
  // 这里的几个source都是"*"，意为接收api/file/http来源的pod更新
	hostNetworkSources, err := kubetypes.GetValidatedSources(kubeServer.HostNetworkSources)
	if err != nil {
		return err
	}

	hostPIDSources, err := kubetypes.GetValidatedSources(kubeServer.HostPIDSources)
	if err != nil {
		return err
	}

	hostIPCSources, err := kubetypes.GetValidatedSources(kubeServer.HostIPCSources)
	if err != nil {
		return err
	}

	privilegedSources := capabilities.PrivilegedSources{
		HostNetworkSources: hostNetworkSources,
		HostPIDSources:     hostPIDSources,
		HostIPCSources:     hostIPCSources,
	}
	capabilities.Setup(kubeServer.AllowPrivileged, privilegedSources, 0)

	credentialprovider.SetPreferredDockercfgPath(kubeServer.RootDirectory)
	klog.V(2).Infof("Using root directory: %v", kubeServer.RootDirectory)

	if kubeDeps.OSInterface == nil {
		kubeDeps.OSInterface = kubecontainer.RealOS{}
	}
  
  // kubelet初始化，这个函数比较复杂，下面拎出来分析
	k, err := createAndInitKubelet(&kubeServer.KubeletConfiguration,
		kubeDeps,
		&kubeServer.ContainerRuntimeOptions,
		kubeServer.ContainerRuntime,
		kubeServer.RuntimeCgroups,
		kubeServer.HostnameOverride,
		kubeServer.NodeIP,
		kubeServer.ProviderID,
		kubeServer.CloudProvider,
		kubeServer.CertDirectory,
		kubeServer.RootDirectory,
		kubeServer.RegisterNode,
		kubeServer.RegisterWithTaints,
		kubeServer.AllowedUnsafeSysctls,
		kubeServer.RemoteRuntimeEndpoint,
		kubeServer.RemoteImageEndpoint,
		kubeServer.ExperimentalMounterPath,
		kubeServer.ExperimentalKernelMemcgNotification,
		kubeServer.ExperimentalCheckNodeCapabilitiesBeforeMount,
		kubeServer.ExperimentalNodeAllocatableIgnoreEvictionThreshold,
		kubeServer.MinimumGCAge,
		kubeServer.MaxPerPodContainerCount,
		kubeServer.MaxContainerCount,
		kubeServer.MasterServiceNamespace,
		kubeServer.RegisterSchedulable,
		kubeServer.NonMasqueradeCIDR,
		kubeServer.KeepTerminatedPodVolumes,
		kubeServer.NodeLabels,
		kubeServer.SeccompProfileRoot,
		kubeServer.BootstrapCheckpointPath,
		kubeServer.NodeStatusMaxImages)
	if err != nil {
		return fmt.Errorf("failed to create kubelet: %v", err)
	}

	// NewMainKubelet should have set up a pod source config if one didn't exist
	// when the builder was run. This is just a precaution.
	if kubeDeps.PodConfig == nil {
		return fmt.Errorf("failed to create kubelet, pod source config was nil")
	}
	podCfg := kubeDeps.PodConfig

	rlimit.RlimitNumFiles(uint64(kubeServer.MaxOpenFiles))

	// 只运行一次处理完pod就退出
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		klog.Info("Started kubelet as runonce")
	} else {
    // 正常运行
		startKubelet(k, podCfg, &kubeServer.KubeletConfiguration, kubeDeps, kubeServer.EnableServer)
		klog.Info("Started kubelet")
	}
	return nil
}
```

### createAndInitKubelet

--> `cmd/kubelet/app/server.go:1078`，这里主要走到NewMainKubelet函数:

--> `pkg/kubelet/kubelet.go:326` God！这个函数简直了，500+行代码...挑重要的说一下吧

```go
func NewMainKubelet(...) (...) {
  ...
  // 加载service informer
  serviceIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})
	if kubeDeps.KubeClient != nil {
		serviceLW := cache.NewListWatchFromClient(kubeDeps.KubeClient.CoreV1().RESTClient(), "services", metav1.NamespaceAll, fields.Everything())
		r := cache.NewReflector(serviceLW, &v1.Service{}, serviceIndexer, 0)
		go r.Run(wait.NeverStop)
	}
	serviceLister := corelisters.NewServiceLister(serviceIndexer)
  
  // 加载node informer
	nodeIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{})
	if kubeDeps.KubeClient != nil {
		fieldSelector := fields.Set{api.ObjectNameField: string(nodeName)}.AsSelector()
		nodeLW := cache.NewListWatchFromClient(kubeDeps.KubeClient.CoreV1().RESTClient(), "nodes", metav1.NamespaceAll, fieldSelector)
		r := cache.NewReflector(nodeLW, &v1.Node{}, nodeIndexer, 0)
		go r.Run(wait.NeverStop)
	}
	nodeInfo := &predicates.CachedNodeInfo{NodeLister: corelisters.NewNodeLister(nodeIndexer)}

  ...
  
  // secretManager和configMapManager初始化，因为这两者被使用都是需要往容器内挂载目录的，需要kubelet来参与
  var secretManager secret.Manager
	var configMapManager configmap.Manager
	switch kubeCfg.ConfigMapAndSecretChangeDetectionStrategy {
	case kubeletconfiginternal.WatchChangeDetectionStrategy:
		secretManager = secret.NewWatchingSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewWatchingConfigMapManager(kubeDeps.KubeClient)
	case kubeletconfiginternal.TTLCacheChangeDetectionStrategy:
		secretManager = secret.NewCachingSecretManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
		configMapManager = configmap.NewCachingConfigMapManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
	case kubeletconfiginternal.GetChangeDetectionStrategy:
		secretManager = secret.NewSimpleSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewSimpleConfigMapManager(kubeDeps.KubeClient)
	default:
		return nil, fmt.Errorf("unknown configmap and secret manager mode: %v", kubeCfg.ConfigMapAndSecretChangeDetectionStrategy)
	}

	klet.secretManager = secretManager
	klet.configMapManager = configMapManager

  // 初始化存活探针管理器
  klet.livenessManager = proberesults.NewManager()
  
  //专为dockershim开辟的网络插件集 
	pluginSettings := dockershim.NetworkPluginSettings{
		HairpinMode:        kubeletconfiginternal.HairpinMode(kubeCfg.HairpinMode),
		NonMasqueradeCIDR:  nonMasqueradeCIDR,
		PluginName:         crOptions.NetworkPluginName,
		PluginConfDir:      crOptions.CNIConfDir,  // 默认在/etc/cni/net.d/下存放cni配置文件
		PluginBinDirString: crOptions.CNIBinDir,   // 默认在/opt/cni/bin/下存放cni二进制文件，如bridge/tuning/vlan/dhcp/macvlan等等
		MTU:                int(crOptions.NetworkPluginMTU), // 网卡mtu
	}  
  
  // kubelet相关运行时初始化 
	runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
		kubecontainer.FilterEventRecorder(kubeDeps.Recorder),
		klet.livenessManager,
		seccompProfileRoot,
		containerRefManager,
		machineInfo,
		klet,
		kubeDeps.OSInterface,
		klet,
		httpClient,
		imageBackOff,
		kubeCfg.SerializeImagePulls,
		float32(kubeCfg.RegistryPullQPS),
		int(kubeCfg.RegistryBurst),
		kubeCfg.CPUCFSQuota,
		kubeCfg.CPUCFSQuotaPeriod,
		runtimeService,
		imageService,
		kubeDeps.ContainerManager.InternalContainerLifecycle(),
		legacyLogProvider,
		klet.runtimeClassManager,
	)
	if err != nil {
		return nil, err
	}
	klet.containerRuntime = runtime
	klet.streamingRuntime = runtime
	klet.runner = runtime

	runtimeCache, err := kubecontainer.NewRuntimeCache(klet.containerRuntime)
	if err != nil {
		return nil, err
	}
	klet.runtimeCache = runtimeCache  
  
  // pleg初始化(Pod Lifecycle Event Generator)
  klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity,      plegRelistPeriod, klet.podCache, clock.RealClock{})
  
  // pod workQueue初始化
  klet.workQueue = queue.NewBasicWorkQueue(klet.clock)
  // pod worker初始化，worker从workQueue中取队首，根据指令对pod进行相应的直接操作，另外还有更新pod cache的操作
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)

	klet.backOff = flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
	klet.podKillingCh = make(chan *kubecontainer.PodPair, podKillingChannelCapacity)

	// 初始化驱逐管理器
	evictionManager, evictionAdmitHandler := eviction.NewManager(klet.resourceAnalyzer, evictionConfig, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.podManager.GetMirrorPodByPod, klet.imageManager, klet.containerGC, kubeDeps.Recorder, nodeRef, klet.clock)

	klet.evictionManager = evictionManager
	klet.admitHandlers.AddPodAdmitHandler(evictionAdmitHandler)
  
  ...
}	

```

再次回到主线，进入最后的k.Run()函数循环逻辑：

--> `cmd/kubelet/app/server.go:1058`

```go
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	// start the kubelet
  // 循环执行kubelet的工作逻辑k.Run()方法
	go wait.Until(func() {
		k.Run(podCfg.Updates())
	}, 0, wait.NeverStop)

	// start the kubelet server
  // 提供诸如/metrics /health等api
	if enableServer {
		go k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)

	}
  // 配置查询api
	if kubeCfg.ReadOnlyPort > 0 {
		go k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.KubeletPodResources) {
		go k.ListenAndServePodResources()
	}
}
```

wait.Until()循环执行函数前面的文章中已经分析过多次了，不再过多赘述，这里传参是period是0，说明是无间隔死循环调用k.Run()方法，体现在实际环境中kubelet运行时的表现就是：无论运行中遇到什么报错，kubelet都会持续工作。

来分析一下k.Run(podCfg.Updates())传的实参是什么:

--> `pkg/kubelet/config/config.go:105`

```
func (c *PodConfig) Updates() <-chan kubetypes.PodUpdate {
   return c.updates
}
```

接着看`c.updates` --> `pkg/kubelet/config/config.go:58`

```go
type PodConfig struct {
   pods *podStorage
   mux  *config.Mux

   // the channel of denormalized changes passed to listeners
   updates chan kubetypes.PodUpdate

   // contains the list of all configured sources
   sourcesLock       sync.Mutex
   sources           sets.String
   checkpointManager checkpointmanager.CheckpointManager
}
```

--> `pkg/kubelet/types/pod_update.go:80`

```go
type PodUpdate struct {
   Pods   []*v1.Pod
   Op     PodOperation
   Source string
}
```

猜测是将pod的写(删查改)请求转换成结构体，放入chan中，然后由k.Run()方法来处理这些写请求，k.Run()的实现在这里``pkg/kubelet/kubelet.go:1382`，留作下回分析，本篇启动流程篇到此结束。



## 小结

kubelet的源码果真是相当的复杂，一个函数动辄数百行，也难怪，毕竟作为daemon端执行数据平面工作的它要承担着很多职责，先到这吧