# Machine Config 渲染与分发机制详解

本文档旨在详细解释 OpenShift Machine Config Operator (MCO) 中 MachineConfig 的渲染、分发机制，ControllerConfig 在其中的作用，并探讨为何 CA 证书的更新似乎不直接触发新的 Rendered MachineConfig 的生成。

## 1. Rendered MachineConfig 的创建过程

新的 Rendered MachineConfig（通常命名为 `rendered-<pool_name>-<hash>`）是由 `machine-config-controller` 中的 `render-controller` 负责生成的。其核心流程如下：

**a. 触发渲染:**

`render-controller` 主要监听两类资源的变化：

*   `MachineConfigPool` (MCP)：当 MCP 的 `.spec`（特别是 `machineConfigSelector`）或 `.metadata.labels` 发生变化时，会触发对其关联的 MachineConfig 进行重新渲染。
*   `MachineConfig` (MC)：当一个 MC 被创建、更新或删除，并且其标签（`.metadata.labels`）匹配某个 MCP 的 `machineConfigSelector` 时，会触发该 MCP 的重新渲染。

```go
// pkg/controller/render/render_controller.go - New()
// ...
mcpInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    ctrl.addMachineConfigPool,
    UpdateFunc: ctrl.updateMachineConfigPool,
    DeleteFunc: ctrl.deleteMachineConfigPool,
})
mcInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    ctrl.addMachineConfig,
    UpdateFunc: ctrl.updateMachineConfig,
    DeleteFunc: ctrl.deleteMachineConfig,
})
// ...
```

**b. 选择 MachineConfigs:**

当一个 MCP 需要渲染时，`render-controller` 会根据该 MCP 的 `.spec.machineConfigSelector` 查找所有匹配的 `MachineConfig` 资源。

```go
// pkg/controller/render/render_controller.go - syncMachineConfigPool()
// ...
selector, err := metav1.LabelSelectorAsSelector(pool.Spec.MachineConfigSelector)
if err != nil {
    return err
}
// ...
mcs, err := ctrl.mcLister.List(selector)
if err != nil {
    return err
}
// ...
```

**c. 合并 MachineConfigs:**

选出的 `MachineConfig` 列表会按照特定规则进行合并，生成一个临时的、聚合的 `MachineConfig`。合并逻辑位于 `pkg/controller/common/helpers.go` 的 `MergeMachineConfigs` 函数中：

*   **排序:** MCs 会先按字母顺序排序，但有一个特殊规则：非 worker 角色的 MC 会排在同名的 worker 角色 MC 之后，确保自定义池的配置能覆盖通用的 worker 配置。
*   **Ignition 合并:** 以排序后的第一个 MC 的 Ignition 配置为基础，后续 MC 的 Ignition 配置通过 `ign3.Merge` 逐层合并。对于同路径的文件，后合并的 MC 会覆盖先合并的（除非设置 `overwrite: false`）。Systemd Unit 类似，但 drop-in 会被合并。
*   **内核参数 (`KernelArguments`):** 所有 MC 的内核参数会被简单地连接在一起。
*   **FIPS:** 只要有一个 MC 启用了 FIPS (`.spec.FIPS = true`)，合并后的配置就启用 FIPS。
*   **内核类型 (`KernelType`):** 如果有 MC 指定了非默认的 `kernelType`（如 `realtime`），该类型优先。
*   **OSImageURL:** 默认使用 `ControllerConfig` 中定义的 `OSImageURL`。但如果任何一个参与合并的 MC 指定了 `OSImageURL`，则该 MC 的值会覆盖 `ControllerConfig` 的默认值。
*   **扩展 (`Extensions`):** 所有 MC 的扩展列表会被连接在一起。

```go
// pkg/controller/common/helpers.go - MergeMachineConfigs()
func MergeMachineConfigs(configs []*mcfgv1.MachineConfig, cconfig *mcfgv1.ControllerConfig) (*mcfgv1.MachineConfig, error) {
	// ... sorting logic ...

	// ... Ignition merging logic using ign3.Merge ...

	// ... FIPS, KernelType logic ...

	kargs := []string{}
	for _, cfg := range configs {
		kargs = append(kargs, cfg.Spec.KernelArguments...)
	}

	extensions := []string{}
	for _, cfg := range configs {
		extensions = append(extensions, cfg.Spec.Extensions...)
	}

	// ... OSImageURL override logic ...
	osImageURL := GetDefaultBaseImageContainer(&cconfig.Spec) // Default from ControllerConfig
	for _, cfg := range configs {
		if cfg.Spec.OSImageURL != "" {
			osImageURL = cfg.Spec.OSImageURL // Override if specified in an MC
		}
	}

	// ... BaseOSExtensionsContainerImage override logic ...

	return &mcfgv1.MachineConfig{
		Spec: mcfgv1.MachineConfigSpec{
			OSImageURL:                     osImageURL,
			BaseOSExtensionsContainerImage: baseOSExtensionsContainerImage,
			KernelArguments:                kargs,
			Config: runtime.RawExtension{
				Raw: rawOutIgn,
			},
			FIPS:       fips,
			KernelType: kernelType,
			Extensions: extensions,
		},
	}, nil
}
```

**d. 生成 Rendered MachineConfig:**

合并后的配置会经过最终处理（例如添加 OwnerReference 指向对应的 MCP，添加版本注解等），并根据其内容计算出一个哈希值，最终生成名为 `rendered-<pool_name>-<hash>` 的 `MachineConfig` 资源。

```go
// pkg/controller/render/render_controller.go - generateRenderedMachineConfig()
// ...
merged, err := ctrlcommon.MergeMachineConfigs(configs, cconfig)
if err != nil {
	return nil, err
}
hashedName, err := getMachineConfigHashedName(pool, merged) // Generates rendered-<pool>-<hash> name
if err != nil {
	return nil, err
}
oref := metav1.NewControllerRef(pool, controllerKind)

merged.SetName(hashedName)
merged.SetOwnerReferences([]metav1.OwnerReference{*oref})
// ... add annotations ...
return merged, nil
```

**e. 更新 MachineConfigPool:**

最后，`render-controller` 会更新触发渲染的 `MachineConfigPool`，将其 `.spec.configuration.name` 指向新生成的 Rendered MachineConfig 的名称。

```go
// pkg/controller/render/render_controller.go - syncGeneratedMachineConfig()
// ...
_, err = ctrl.client.MachineconfigurationV1().MachineConfigs().Create(context.TODO(), generated, metav1.CreateOptions{})
// ...
newPool := pool.DeepCopy()
newPool.Spec.Configuration.Source = source
newPool.Spec.Configuration.Name = generated.Name // Point MCP to the new rendered MC
pool, err = ctrl.client.MachineconfigurationV1().MachineConfigPools().Update(context.TODO(), newPool, metav1.UpdateOptions{})
// ...
```

## 2. 配置分发与应用

当一个新的 Rendered MachineConfig 被生成并关联到某个 MCP 后，配置的分发和应用由运行在每个节点上的 `machine-config-daemon` (MCD) 负责。

**a. 检测变更:**

MCD 会 watch 其所在节点的 Node 对象。它主要关注以下 annotations：

*   `machineconfiguration.openshift.io/currentConfig`: 当前节点应用的配置名称。
*   `machineconfiguration.openshift.io/desiredConfig`: 控制器期望节点应用的配置名称。
*   `machineconfiguration.openshift.io/currentImage`: 当前节点应用的 OS 镜像 URL (如果使用了 OS 更新)。
*   `machineconfiguration.openshift.io/desiredImage`: 控制器期望节点应用的 OS 镜像 URL。
*   `machineconfiguration.openshift.io/state`: MCD 的状态 (如 `Done`, `Working`, `Degraded`)。

当 `desiredConfig` (和 `desiredImage`) 与 `currentConfig` (和 `currentImage`) 不一致时，MCD 会认为需要进行更新。

```go
// pkg/daemon/daemon.go - syncNode() / getStateAndConfigs()
// ... reads annotations like constants.DesiredMachineConfigAnnotationKey ...

// pkg/daemon/daemon.go - prepUpdateFromCluster()
// ... compares desiredConfigName with currentConfigName and desiredImage with currentImage ...
if desiredImage == "" && odc.currentImage == "" {
    if desiredConfigName == currentConfigName {
        // ... no update needed if state is Done ...
    }
} else {
    if desiredImage == odc.currentImage && desiredConfigName == currentConfigName {
       // ... no update needed if state is Done ...
    }
}
// ... otherwise, an update is needed ...
```

**b. 执行更新 (`update`):**

如果检测到需要更新，MCD 会调用 `triggerUpdate` -> `update` (或特定更新类型的函数)。`update` 函数执行以下操作：

*   **获取配置:** 从 API Server 获取 `desiredConfig` 的 `MachineConfig` 对象。
*   **解析 Ignition:** 解析 `desiredConfig.spec.config.raw` 中的 Ignition 配置。
*   **应用变更:**
    *   写入文件 (`writeFiles`)。
    *   管理 Systemd 单元 (`writeUnits`, `reloadSystemd`)。
    *   如果 `OSImageURL` 发生变化，执行 `rpm-ostree` 更新 (`updateOS`)。
    *   更新 SSH 密钥 (`updateSSHKeys`)。
    *   应用内核参数 (`applyKernelArguments`)。
    *   更新 FIPS 设置 (`applyFIPS`)。
*   **记录状态:** 将成功应用的 `desiredConfig` 内容写入本地文件 `/etc/machine-config-daemon/currentconfig` (镜像 URL 写入 `/etc/machine-config-daemon/currentimage`)，作为节点当前状态的持久化记录。
*   **判断后续操作:** 根据 Ignition 配置的变化（文件、systemd unit 等）判断是否需要重启节点 (`rebootRequired`) 或仅重载某些服务（如 CRI-O）。
*   **执行重启/重载:** 如果需要重启，调用 `rebootCommand` 执行 `systemd-run systemctl reboot`。如果需要重载服务，执行 `systemctl reload <service>`。

```go
// pkg/daemon/daemon.go - update()
func (dn *Daemon) update(currentConfig, desiredConfig *mcfgv1.MachineConfig, skipCertificateWrite bool) error {
	// ... lock updateActiveLock ...

	// ... calculate diff between current and desired ...

	// ... parse desired Ignition config ...
	newIgnConfig, err := ctrlcommon.ParseAndConvertConfig(desiredConfig.Spec.Config.Raw)
	// ...

	// ... write files, units, ssh keys, fips, kargs ...
	if err := dn.writeFiles(newIgnConfig.Storage.Files, isLayered); err != nil {
		// ...
	}
	if err := dn.writeUnits(newIgnConfig.Systemd.Units); err != nil {
		// ...
	}
	// ...

	// ... OS update via rpm-ostree if OSImageURL changed ...
	osUpdateRequired := dn.checkOS(desiredConfig.Spec.OSImageURL)
	if !osUpdateRequired {
		if err := dn.updateOS(desiredConfig); err != nil {
			// ...
		}
	}

	// ... store current config/image to disk ...
	odc := &onDiskConfig{
		currentConfig: desiredConfig,
		currentImage:  desiredConfig.Spec.OSImageURL,
	}
	if err := dn.storeCurrentConfigOnDisk(odc); err != nil {
		// ...
	}

	// ... determine if reboot or reload is needed ...
	actions, err := calculatePostConfigChangeAction(mcDiff, diffFileSet, oldIgnConfig, newIgnConfig, osUpdateRequired)
	// ...

	// ... perform reboot or reload ...
	if ctrlcommon.InSlice(postConfigChangeActionReboot, actions) {
		return dn.reboot(fmt.Sprintf("Node will reboot into config %s", desiredConfig.Name))
	}
	// ... reload services like crio if needed ...

	// ... unlock updateActiveLock ...
	return nil
}
```

**c. 完成更新:**

节点重启（如果需要）后，MCD 再次启动，`checkStateOnFirstRun` 会验证当前应用的配置（通过 `/etc/machine-config-daemon/currentconfig` 和 `rpm-ostree status` 等）是否与 Node 对象上的 `desiredConfig` 一致。如果一致，MCD 会调用 `completeUpdate`：

*   更新 Node 对象的 annotation，将 `currentConfig` 设置为 `desiredConfig`，并将状态设置为 `Done`。
*   请求 `machine-config-controller` 解除对节点的 Cordon (通过设置 `DesiredDrainerAnnotationKey` 为 `uncordon-<config_hash>`)。

```go
// pkg/daemon/daemon.go - checkStateOnFirstRun() -> updateConfigAndState()
// ... validates on-disk state against desiredConfig annotation ...
if inDesiredConfig {
	// ...
	klog.Infof("Completing update to target %s", state.getCurrentName())
	if err := dn.completeUpdate(state.currentConfig.GetName()); err != nil {
		// ...
	}
	// ...
	if err := dn.nodeWriter.SetDone(state); err != nil { // Updates node annotations (current=desired, state=Done)
		// ...
	}
	// ...
}

// pkg/daemon/daemon.go - completeUpdate()
func (dn *Daemon) completeUpdate(desiredConfigName string) error {
	if err := dn.nodeWriter.SetDesiredDrainer(fmt.Sprintf("%s-%s", "uncordon", desiredConfigName)); err != nil { // Request uncordon
		return fmt.Errorf("could not set drain annotation: %w", err)
	}
	// ... waits for controller to confirm uncordon ...
	return nil
}
```

## 3. ControllerConfig 的作用

`ControllerConfig` (通常名为 `machine-config`) 是一个集群范围的单例资源，它扮演着 MCO 全局配置和状态中心的角色。其主要作用包括：

*   **提供基础信息:** 存储集群的基础 OS 镜像 URL (`.spec.baseOSContainerImage`)、扩展镜像 URL (`.spec.baseOSExtensionsContainerImage`) 等，这些信息被 `render-controller` 用作生成 Rendered MachineConfig 的默认值（如上文 `MergeMachineConfigs` 所示）。
*   **存储证书和 Pull Secret:** 包含集群范围的 CA 证书（如 Kube APIServer Serving CA, Root CA, Cloud Provider CA, Additional Trust Bundle）和镜像仓库的 Pull Secret (`.spec.internalRegistryPullSecret`, `.spec.imageRegistryBundleData`, `.spec.imageRegistryBundleUserData`)。
    *   `template-controller` 使用 Pull Secret 来渲染需要访问镜像仓库的模板（虽然 CA 证书不直接用于模板渲染）。
    *   `machine-config-daemon` (通过 `certificate_writer.go` 逻辑) 会 watch `ControllerConfig` 的变化，并将这些证书同步到节点上的指定路径（如 `/etc/kubernetes/kubelet-ca.crt`, `/etc/docker/certs.d/` 等）。
*   **版本控制:** `ControllerConfig` 的 annotations 中包含了 MCO 的版本信息 (`machineconfiguration.openshift.io/generated-by-version`)。`render-controller` 会检查这个版本，以确保它只使用由兼容版本的 MCO 生成的 `MachineConfig` 来进行渲染，防止在 MCO 升级过程中出现不兼容问题。
*   **驱动模板渲染:** `template-controller` watch `ControllerConfig` 的变化。当 `ControllerConfig` 更新时（例如 `OSImageURL` 改变），`template-controller` 会使用 `ControllerConfig` 中的信息和预定义的模板（位于 `/etc/mcc/templates`）来生成或更新特定的 `MachineConfig` 资源（例如 `99-<pool>-generated-kubelet`）。这些生成的 MC 随后会被 `render-controller` 纳入渲染过程。

```go
// pkg/controller/template/template_controller.go - syncControllerConfig()
// ... watches ControllerConfig ...
mcs, err := getMachineConfigsForControllerConfig(ctrl.templatesDir, cfg, clusterPullSecretRaw)
if err != nil {
    return ctrl.syncFailingStatus(cfg, err)
}

for _, mc := range mcs {
    _, updated, err := mcoResourceApply.ApplyMachineConfig(ctrl.client.MachineconfigurationV1(), mc) // Apply generated MC
    // ...
}
```

## 4. CA 证书更新与 Rendered MachineConfig

根据上述分析，CA 证书的更新通常**不会**直接触发 `render-controller` 生成一个新的 Rendered MachineConfig。原因如下：

*   **Render Controller 的触发机制:** `render-controller` 主要监听 `MachineConfigPool` 和 `MachineConfig` 的变化。CA 证书通常存储在 `ControllerConfig` 的 `.spec` 中，或者在 Secret/ConfigMap 中（然后被同步到 `ControllerConfig`）。直接修改 `ControllerConfig` 中的证书数据，并不会触发 `render-controller` 的 `syncMachineConfigPool` 逻辑。
*   **Daemon 的证书同步:** 节点上的 `machine-config-daemon` (MCD) 通过 `certificate_writer.go` 中的逻辑直接 watch `ControllerConfig`。当 `ControllerConfig` 中的证书数据更新时（`ResourceVersion` 变化），MCD 会被触发 (`syncControllerConfigHandler`)，并将更新后的证书写入节点本地文件系统（如 `/etc/kubernetes/kubelet-ca.crt`, `/etc/docker/certs.d/` 等）。这个过程绕过了 `render-controller` 和 MachineConfig 的渲染。

```go
// pkg/daemon/certificate_writer.go - syncControllerConfigHandler()
func (dn *Daemon) syncControllerConfigHandler(key string) error {
	// ... get ControllerConfig ...

	currentNodeControllerConfigResource := dn.node.Annotations[constants.ControllerConfigResourceVersionKey]
	// ...
	// Checks if the ControllerConfig resource version has changed or if a service CA rotation is annotated
	if currentNodeControllerConfigResource != controllerConfig.ObjectMeta.ResourceVersion || controllerConfig.Annotations[ctrlcommon.ServiceCARotateAnnotation] == ctrlcommon.ServiceCARotateTrue {
		pathToData := make(map[string][]byte)
		kubeAPIServerServingCABytes := controllerConfig.Spec.KubeAPIServerServingCAData
		cloudCA := controllerConfig.Spec.CloudProviderCAData
		// ... populate pathToData with certs from controllerConfig.Spec ...

		// ... handle image registry certs ...

		// Writes certificates directly to disk paths like /etc/kubernetes/kubelet-ca.crt, /etc/docker/certs.d/*
		if err := writeToDisk(pathToData); err != nil {
			return err
		}
		// ... update node annotation with new resourceVersion ...
	}
	// ...
	return nil
}
```

*   **日志佐证 (脱敏):** `wzh.evid/mco.log` 中的日志显示 `Certificate was synced from controllerconfig resourceVersion XXXXX`。这表明 MCD 确实在响应 `ControllerConfig` 的 `resourceVersion` 变化并同步证书，但这并不等同于生成新的 Rendered MachineConfig。
*   **模板控制器:** `template-controller` 监听 `ControllerConfig`，但主要是为了响应像 `OSImageURL` 或 Pull Secret 这样的变化来重新生成模板化的 `MachineConfig`。它不直接处理 `.spec` 中的 CA 证书数据来触发模板更新。

**结论:** CA 证书的更新是通过 MCD 直接同步到节点文件系统的，而不是通过触发 `render-controller` 重新生成包含新证书的 Rendered MachineConfig 来分发。这种设计可能是为了效率和避免不必要的节点重启，因为仅仅更新 CA 文件通常不需要修改其他系统配置或重启服务（除非特定服务配置了监视这些文件）。如果需要将新的 CA 证书嵌入到某个服务的配置文件中（而不仅仅是放在标准信任路径下），则需要创建一个新的 `MachineConfig` 来包含这个文件，这才会触发 `render-controller`。

## 5. 时序图 (Mermaid)

以下时序图展示了 MachineConfig 的渲染和分发流程，并区分了不同类型的触发事件：

```mermaid
sequenceDiagram
    participant User
    participant MCC as Machine Config Controller
    participant RenderCtrl as Render Controller (in MCC)
    participant TemplateCtrl as Template Controller (in MCC)
    participant APIServer
    participant MCD as Machine Config Daemon (on Node)
    participant Node

    User->>APIServer: Apply MachineConfig (MC) / MachineConfigPool (MCP) / ControllerConfig (CC) Update
    APIServer->>MCC: Inform Watchers

    alt MCP/MC Change (Directly affects rendering)
        APIServer->>RenderCtrl: Inform MCP/MC Change
        RenderCtrl->>APIServer: Get matching MCs for MCP
        APIServer-->>RenderCtrl: Return MCs
        RenderCtrl->>APIServer: Get ControllerConfig (for defaults/version)
        APIServer-->>RenderCtrl: Return ControllerConfig
        RenderCtrl->>RenderCtrl: Merge MCs (using CC defaults)
        RenderCtrl->>RenderCtrl: Calculate Hash & Generate Rendered MC Name
        RenderCtrl->>APIServer: Create Rendered MachineConfig (rendered-pool-hash)
        APIServer-->>RenderCtrl: Confirm Creation
        RenderCtrl->>APIServer: Update MCP .spec.configuration.name = rendered-pool-hash
        APIServer-->>RenderCtrl: Confirm MCP Update
        APIServer->>MCD: Inform Node Annotation Change (desiredConfig = rendered-pool-hash)
    else ControllerConfig Change (e.g., OSImageURL - Indirectly affects rendering via TemplateCtrl)
        APIServer->>TemplateCtrl: Inform CC Change
        TemplateCtrl->>APIServer: Get ControllerConfig
        APIServer-->>TemplateCtrl: Return ControllerConfig
        TemplateCtrl->>TemplateCtrl: Render Templates (e.g., kubelet config) using CC data
        TemplateCtrl->>APIServer: Apply/Update Generated MachineConfig (e.g., 99-pool-generated-kubelet)
        APIServer-->>TemplateCtrl: Confirm Apply/Update
        Note over APIServer, RenderCtrl: Generated MC update triggers RenderCtrl (as MC Change)
        APIServer->>RenderCtrl: Inform MC Change (Generated MC)
        RenderCtrl->>APIServer: Get matching MCs for MCP (incl. Generated MC)
        APIServer-->>RenderCtrl: Return MCs
        RenderCtrl->>APIServer: Get ControllerConfig
        APIServer-->>RenderCtrl: Return ControllerConfig
        RenderCtrl->>RenderCtrl: Merge MCs
        RenderCtrl->>RenderCtrl: Calculate Hash & Generate New Rendered MC Name
        RenderCtrl->>APIServer: Create Rendered MachineConfig
        APIServer-->>RenderCtrl: Confirm Creation
        RenderCtrl->>APIServer: Update MCP .spec.configuration.name
        APIServer-->>RenderCtrl: Confirm MCP Update
        APIServer->>MCD: Inform Node Annotation Change (desiredConfig)
    else ControllerConfig Change (CA Certs Only - Does NOT affect rendering)
        APIServer->>MCD: Inform CC Change (via ControllerConfig Watcher in MCD's certificate_writer)
        MCD->>APIServer: Get ControllerConfig
        APIServer-->>MCD: Return ControllerConfig
        MCD->>Node: Write updated CA certs directly to disk (e.g., /etc/kubernetes/kubelet-ca.crt)
        MCD->>APIServer: Update Node Annotation (controllerconfig resource version)
        Note over MCD, Node: No new Rendered MC generated, no node reboot typically needed.
    end

    MCD->>APIServer: Get Node Object (check annotations)
    APIServer-->>MCD: Return Node Object
    MCD->>MCD: Compare currentConfig vs desiredConfig annotations
    alt Update Needed (Node needs new rendered config)
        MCD->>APIServer: Get Desired MachineConfig (rendered-pool-hash)
        APIServer-->>MCD: Return Desired MachineConfig
        MCD->>MCD: Parse Ignition Config
        MCD->>Node: Apply changes (write files, units, rpm-ostree, etc.)
        MCD->>Node: Write desiredConfig to /etc/machine-config-daemon/currentconfig
        MCD->>MCD: Determine if reboot/reload needed
        opt Reboot Needed
            MCD->>Node: Initiate Reboot (systemd-run systemctl reboot)
            Node-->>MCD: Node Reboots
            MCD->>MCD: Restart MCD after reboot
            MCD->>APIServer: Get Node Object (check annotations after reboot)
            APIServer-->>MCD: Return Node Object
            MCD->>MCD: Validate state (current == desired)
            MCD->>APIServer: Update Node Annotations (current=desired, state=Done)
            MCD->>APIServer: Request Uncordon (update drain annotation)
            APIServer->>MCC: Inform Drain Annotation Change
            MCC->>APIServer: Uncordon Node
        else No Reboot Needed
            MCD->>APIServer: Update Node Annotations (current=desired, state=Done)
            MCD->>APIServer: Request Uncordon
            APIServer->>MCC: Inform Drain Annotation Change
            MCC->>APIServer: Uncordon Node
        end
    else No Update Needed (Node is up-to-date)
        MCD->>MCD: State is Done, No Action
    end
```