# 证书更新触发 Systemd 重载及 Kubenswrapper 报错分析

## 1. 引言

本文档旨在分析在 OpenShift Container Platform (OCP) 或类似环境中，当 kube-apiserver 相关证书或其他由 Machine Config Operator (MCO) 管理的配置发生更新时，为何会触发节点上 systemd 服务的重载（reload），以及这个过程为何可能导致 `kubenswrapper` 相关报错。

分析基于 MCO 源代码、节点日志 (`node.log`)、MCD 日志 (`mco.log`) 以及 `kubenswrapper` 脚本本身。

## 2. MCO/MCD 更新流程

当集群中的 MachineConfig 发生变化时（例如，由证书轮换触发），MCO 会生成新的 `rendered-<pool>-<hash>` MachineConfig 资源。节点上的 Machine Config Daemon (MCD) 会监视其所属 MachineConfigPool 的 `rendered` 配置。

检测到变更后，MCD 会执行以下主要步骤（简化流程）：

1.  **对比配置差异**: 计算新旧 MachineConfig 之间的差异 (`reconcilable` 函数)。
2.  **应用更改**: 调用 `update` 函数。
    *   **`updateFiles`**:
        *   **`writeFiles`**: 将新的配置文件写入磁盘。
        *   **`writeUnits`**: 将新的 systemd 单元文件和 drop-in 文件写入 `/etc/systemd/system/` 及其子目录 (`.d/`)。
        *   **`enableUnits`/`disableUnits`**: 根据需要调用 `systemctl enable` 或 `systemctl disable`。
        *   **`deleteStaleData`**: 删除旧配置中存在但新配置中不存在的文件或单元。
    *   **`applyOSChanges` (如果是 CoreOS)**: 处理 OS 更新、内核参数、扩展等（可能涉及 `rpm-ostree`）。

**关键代码片段 (`pkg/daemon/update.go`):**

```go
// updateFiles writes files specified by the nodeconfig to disk. it also writes
// systemd units.
// ...
func (dn *Daemon) updateFiles(oldIgnConfig, newIgnConfig ign3types.Config) error {
	glog.Info("Updating files")
	if err := dn.writeFiles(newIgnConfig.Storage.Files); err != nil { // 写入普通文件
		return err
	}
	if err := dn.writeUnits(newIgnConfig.Systemd.Units); err != nil { // 写入 Systemd Units 和 Drop-ins
		return err
	}
	if err := dn.deleteStaleData(oldIgnConfig, newIgnConfig); err != nil { // 删除旧文件
		return err
	}
	return nil
}

// writeUnits writes the systemd units to disk
func (dn *Daemon) writeUnits(units []ign3types.Unit) error {
	var enabledUnits []string
	var disabledUnits []string
	for _, u := range units {
		if err := dn.writeDropins(u); err != nil { // 写入 Drop-ins
			return err
		}

		fpath := filepath.Join(pathSystemd, u.Name)
		// ... (处理 mask) ...

		if u.Contents != nil && *u.Contents != "" {
			glog.Infof("Writing systemd unit %q", u.Name)
            // *** 写入 Unit 文件到 /etc/systemd/system/ ***
			if err := writeFileAtomicallyWithDefaults(fpath, []byte(*u.Contents)); err != nil {
				return fmt.Errorf("failed to write systemd unit %q: %w", u.Name, err)
			}
		}
        // ... (处理 unmask) ...

		if u.Enabled != nil {
			if *u.Enabled {
				enabledUnits = append(enabledUnits, u.Name)
			} else {
				disabledUnits = append(disabledUnits, u.Name)
			}
		}
        // ... (处理 preset) ...
	}

	if len(enabledUnits) > 0 {
        // *** 调用 systemctl enable ***
		if err := dn.enableUnits(enabledUnits); err != nil {
			return err
		}
	}
	if len(disabledUnits) > 0 {
        // *** 调用 systemctl disable ***
		if err := dn.disableUnits(disabledUnits); err != nil {
			return err
		}
	}
	return nil
}

// write dropins to disk
func (dn *Daemon) writeDropins(u ign3types.Unit) error {
	for i := range u.Dropins {
		dpath := filepath.Join(pathSystemd, u.Name+".d", u.Dropins[i].Name)
		// ...
		glog.Infof("Writing systemd unit dropin %q", u.Dropins[i].Name)
        // *** 写入 Drop-in 文件到 /etc/systemd/system/<unit>.d/ ***
		if err := writeFileAtomicallyWithDefaults(dpath, []byte(*u.Dropins[i].Contents)); err != nil {
			return fmt.Errorf("failed to write systemd unit dropin %q: %w", u.Dropins[i].Name, err)
		}
		// ...
	}
	return nil
}
```

从 `wzh.evid/mco.log` 可以看到 MCD 写入了大量文件，包括 systemd 单元和 drop-in：

```log
I0410 14:23:23.175331    4014 update.go:1650] Writing file "/etc/systemd/system.conf.d/kubelet-cgroups.conf"
I0410 14:23:23.177168    4014 update.go:1650] Writing file "/etc/systemd/system/kubelet.service.d/20-logging.conf"
...
I0410 14:23:23.212378    4014 update.go:1582] Writing systemd unit "NetworkManager-clean-initrd-state.service"
...
I0410 14:23:23.219474    4014 update.go:1537] Writing systemd unit dropin "01-kubens.conf"
...
I0410 14:23:24.365373    4014 update.go:1537] Writing systemd unit dropin "01-kubens.conf"
...
I0410 14:23:24.369046    4014 update.go:1582] Writing systemd unit "kubelet.service"
I0410 14:23:24.370899    4014 update.go:1582] Writing systemd unit "kubens.service"
...
I0410 14:23:27.646794    4014 update.go:1492] Enabled systemd units: [...]
I0410 14:23:28.584428    4014 update.go:1503] Disabled systemd units [kubens.service nodeip-configuration.service]
```

## 3. Systemd 重载机制

Systemd 会监视其配置文件目录（主要是 `/etc/systemd/system/` 和 `/run/systemd/system/`）。当这些目录中的单元文件、drop-in 文件或符号链接发生更改时，systemd 管理器配置被视为“过时”。为了让 systemd 加载这些新的或修改过的配置，需要执行 `systemctl daemon-reload` 命令。

MCD 在 `writeUnits` 函数中写入或删除单元/drop-in 文件，并通过 `enableUnits`/`disableUnits` 函数修改单元的启用状态（这会创建或删除符号链接），这些操作都会导致 systemd 认为其配置已更改。**Systemd 会自动触发或被 MCD 间接触发执行 `daemon-reload`**。

节点日志 (`wzh.evid/node.log`) 证实了这一点，在 MCD 更新期间多次出现 `Reloading.` 日志：

```log
Apr 10 14:23:23 ip-10-0-131-77 systemd[1]: Reloading.
Apr 10 14:23:24 ip-10-0-131-77 systemd[1]: Reloading.
Apr 10 14:23:25 ip-10-0-131-77 systemd[1]: Reloading.
Apr 10 14:23:26 ip-10-0-131-77 systemd[1]: Reloading.
Apr 10 14:23:27 ip-10-0-131-77 systemd[1]: Reloading.
```

`daemon-reload` 会重新加载所有单元文件并重新计算依赖关系树，但**不会**导致正在运行的服务重启。然而，如果 `daemon-reload` 过程中需要停止某些服务（例如，旧单元文件被删除），或者如果其他管理操作（如 `systemctl restart <service>`）恰好在此时发生，服务状态就可能改变。

## 4. kubens.service 分析

`kubens.service` 是一个 `Type=oneshot` 且 `RemainAfterExit=yes` 的 systemd 服务。它的主要目的是创建一个独立的 mount namespace，并将该 namespace "钉" 在一个已知的位置 (`/run/kubens/mnt`)，供其他进程（主要是 Kubelet 和 CRI-O）进入。

**`templates/common/_base/units/kubens.service.yaml`:**

```yaml
name: kubens.service
enabled: false # 通常由依赖它的服务（如 kubelet）触发启动
contents: |
  [Unit]
  Description=Manages a mount namespace for kubernetes-specific mounts

  [Service]
  Type=oneshot
  RemainAfterExit=yes # 进程退出后，namespace 仍然保持
  RuntimeDirectory=kubens # 在 /run/ 下创建 kubens 目录
  Environment=RUNTIME_DIRECTORY=%t/kubens
  Environment=BIND_POINT=%t/kubens/mnt # 定义 mount namespace 的挂载点路径
  Environment=ENVFILE=%t/kubens/env

  # ... (创建目录和挂载点) ...
  # 使用 unshare 创建新的 mount namespace，并将其绑定到 BIND_POINT
  ExecStart=unshare --mount=${BIND_POINT} --propagation slave mount --make-rshared /
  # 将挂载点路径写入环境变量文件，方便 kubenswrapper 读取
  ExecStartPost=bash -c 'echo "KUBENSMNT=${BIND_POINT}" > "${ENVFILE}"'

  # 停止时卸载挂载点，清理 namespace
  ExecStop=umount -R ${RUNTIME_DIRECTORY}

  [Install]
  WantedBy=multi-user.target
```

这个服务通过 `unshare --mount=${BIND_POINT}` 创建一个新的 mount namespace，并通过将 `/` 重新挂载到其中（`mount --make-rshared /`）来初始化它。重要的是，这个 namespace 在服务进程退出后仍然存在，并通过 `/run/kubens/mnt` 这个文件描述符暴露给其他进程。

## 5. kubenswrapper 分析

`kubenswrapper` (`/usr/local/bin/kubenswrapper`) 是一个 shell 脚本，被用作 Kubelet 和 CRI-O 等服务的 `ExecStartPre` 或 `ExecStart` 的包装器。它的目的是确保这些服务的主进程在 `kubens.service` 创建的特定 mount namespace 中运行。

**关键逻辑 (`wzh.evid/kubensenter.txt`):**

1.  **`autodetect` 函数**:
    *   检查环境变量 `KUBENSMNT` 是否已设置。
    *   如果未设置，检查默认路径 `/run/kubens/mnt` 是否存在。
    *   **关键验证**: 使用 `findmnt -o SOURCE -n -t nsfs "$1"` 和 `[[ $nsfs =~ ^nsfs\[mnt:\[ ]]` 检查该路径是否确实是一个类型为 `nsfs` 的 *mount* namespace 文件描述符。如果文件存在但类型不对（例如，`kubens.service` 异常退出或被清理），验证会失败。
    *   如果验证成功，设置 `KUBENSMNT` 变量。

    ```bash
    # wzh.evid/kubensenter.txt
    DEFAULT_KUBENSMNT=${DEFAULT_KUBENSMNT:-"/run/kubens/mnt"}
    autodetect() {
        local default=$DEFAULT_KUBENSMNT
        if [[ -n $KUBENSMNT ]]; then
            debug "Autodetect: \$KUBENSMNT already set"
            return 0
        fi
        if [[ ! -e $default ]]; then
            debug "Autodetect: No mount namespace found at $default"
            return 1
        fi
        # 检查是否为有效的 mount namespace
        if ! ismnt "$default"; then
            info "Autodetect: Stale or mismatched namespace at $default"
            return 1
        fi
        KUBENSMNT=$default
        info "Autodetect: kubens.service namespace found at $KUBENSMNT"
        return 0
    }
    # Returns 0 if the argument given is a mount namespace
    ismnt() {
        local nsfs
        nsfs=$(findmnt -o SOURCE -n -t nsfs "$1")
        [[ $nsfs =~ ^nsfs\[mnt:\[ ]]
    }
    ```

2.  **`kubensenter` 函数**:
    *   如果 `KUBENSMNT` 变量被成功设置（意味着检测到有效的 namespace），则构造 `nsenter --mount=$KUBENSMNT` 参数。
    *   使用 `exec nsenter $nsarg "$@"` 执行 `nsenter`。如果 `$nsarg` 存在，则进入指定的 mount namespace；如果不存在（因为 `autodetect` 失败），则不带 `--mount` 参数执行 `nsenter`，这实际上等同于直接执行目标命令 (`"$@"`)。

    ```bash
    # wzh.evid/kubensenter.txt
    kubensenter() {
        local nsarg
        if [[ -n $KUBENSMNT ]]; then
            debug "Joining mount namespace in $KUBENSMNT"
            # 检查 KUBENSMNT 是否确实挂载 (防止 KUBENSMNT 被设置但服务已停止)
            if [[ "$(findmnt --mountpoint="$KUBENSMNT")" ]]; then
                nsarg=$(printf -- "--mount=%q" "$KUBENSMNT")
            else
               info "WARNING: $KUBENSMNT is not mounted; running normally"
            fi
        else
            debug "KUBENSMNT not set; running normally"
        fi
        # 使用 exec nsenter 执行，如果 nsarg 为空，则不进入特定 namespace
        exec nsenter $nsarg "$@"
    }
    ```

Kubelet 和 CRI-O 通过 systemd drop-in 文件来使用 `kubenswrapper`：

**`templates/common/_base/units/kubelet.service-kubens.yaml` (片段):**

```yaml
contents: |
  [Service]
  # Wrap kubelet startup in entering the mount namespace
  # managed by kubens.service
  #
  ExecStartPre=/usr/local/bin/kubenswrapper -- /usr/bin/mkdir -p /var/lib/kubelet/pods
  ExecStart=
  ExecStart=/usr/local/bin/kubenswrapper -- /usr/bin/hyperkube kubelet ...

  [Unit]
  # ...
  After=kubens.service
```

## 6. 关联分析：Systemd 重载如何影响 Kubenswrapper

核心问题在于 `systemd daemon-reload` 和 `kubens.service` 的生命周期以及 `kubenswrapper` 的检测逻辑之间的潜在冲突：

1.  **MCD 更新**: MCD 写入 systemd 单元文件或 drop-in（例如更新 Kubelet 或 CRI-O 的配置，或者 `kubens.service` 本身）。
2.  **Systemd 重载**: systemd 检测到配置变化，执行 `daemon-reload`。
3.  **`kubens.service` 状态变化 (可能)**: 在 `daemon-reload` 期间，systemd 内部状态会更新。虽然 `daemon-reload` 不直接重启服务，但如果 `kubens.service` 的单元文件被修改或其依赖关系发生变化，或者 systemd 决定需要短暂停止再启动它（虽然 `Type=oneshot` 和 `RemainAfterExit=yes` 设计上是为了避免这种情况，但重载过程本身可能存在短暂的窗口期），那么 `/run/kubens/mnt` 这个由 `kubens.service` 维护的 mount namespace 文件描述符可能会短暂失效或变为非 mount namespace 类型。
4.  **依赖服务启动/重启**: 同时，其他服务（如 Kubelet 或 CRI-O）可能因为配置更新或其他原因需要启动或重启。
5.  **`kubenswrapper` 检测**: 这些服务的 `ExecStartPre` 或 `ExecStart` 调用 `kubenswrapper`。
6.  **检测失败**: 如果此时 `kubens.service` 尚未完全就绪或其 namespace 文件描述符 `/run/kubens/mnt` 无效，`kubenswrapper` 的 `autodetect` -> `ismnt` 检查会失败。它会记录 `Stale or mismatched namespace at /run/kubens/mnt`。
7.  **错误 Namespace 执行**: `kubenswrapper` 在检测失败后，会不带 `--mount` 参数执行 `nsenter`，导致 Kubelet 或 CRI-O 的主进程在 MCD 所在的原始 mount namespace 中启动，而不是预期的 `kubens.service` 的 namespace。
8.  **后续错误**: 如果 Kubelet 或 CRI-O 依赖于只有在 `kubens.service` namespace 内才存在的特定挂载点或配置（例如 `/var/lib/kubelet` 下的某些特殊挂载），那么在错误的 namespace 中运行将导致它们无法找到所需资源，从而产生各种运行时错误。这些错误会被记录在 Kubelet 或 CRI-O 的日志中，表面上看起来可能是 `kubenswrapper` 调用失败（因为它未能将进程置于正确的环境中）。

虽然提供的 `node.log` 没有直接显示 Kubelet 或 CRI-O 因 namespace 问题而失败的日志，但 `systemd daemon-reload` 的发生与 `kubenswrapper` 依赖 `kubens.service` 稳定性的机制相结合，解释了为什么在 MCO 更新（尤其是涉及 systemd 单元的更新）期间，存在发生此类错误的风险窗口。

## 7. 时序图 (Mermaid Sequence Diagram)

```mermaid
sequenceDiagram
    participant MCD as Machine Config Daemon
    participant Systemd
    participant KubensService as kubens.service
    participant KubeletService as kubelet.service (or crio.service)
    participant Kubenswrapper as kubenswrapper
    participant KubeletProcess as Kubelet Process (or CRI-O)

    MCD ->>+ Systemd: Write/Modify Unit/Drop-in Files (/etc/systemd/system/...)
    Systemd ->> Systemd: Detect config change, Trigger Daemon Reload
    Note over Systemd: Reloading configuration...
    Systemd ->> KubensService: Potentially stops/restarts or makes ns temporarily invalid
    MCD ->>+ Systemd: systemctl enable/disable (if needed)
    Systemd ->> Systemd: Daemon Reload (if not already triggered)
    Systemd ->>- KubensService: Potentially stops/restarts or makes ns temporarily invalid

    Systemd ->>+ KubeletService: Start/Restart Service (due to config change or other reason)
    KubeletService ->>+ Kubenswrapper: ExecStartPre/ExecStart calls kubenswrapper
    Kubenswrapper ->>+ KubensService: Check validity of /run/kubens/mnt (ismnt)
    
    alt Namespace Invalid/Unavailable (during reload window)
        KubensService ->>- Kubenswrapper: Check Fails (namespace stale or missing)
        Kubenswrapper ->> Kubenswrapper: Log "Stale or mismatched namespace"
        Kubenswrapper ->>+ KubeletProcess: exec nsenter (without --mount) /usr/bin/hyperkube kubelet ...
        KubeletProcess ->> KubeletProcess: Runs in WRONG namespace
        KubeletProcess ->>- KubeletService: Potential runtime errors (missing mounts, etc.)
        KubeletService ->>- Systemd: Service fails or logs errors
    else Namespace Valid
        KubensService ->>- Kubenswrapper: Check Succeeds
        Kubenswrapper ->>+ KubeletProcess: exec nsenter --mount=/run/kubens/mnt /usr/bin/hyperkube kubelet ...
        KubeletProcess ->> KubeletProcess: Runs in CORRECT namespace
        KubeletProcess ->>- KubeletService: Service runs normally
        KubeletService ->>- Systemd: Service runs successfully
    end
```

## 8. 结论

kube-apiserver 证书更新或其他由 MCO 管理的配置更改，通过 MCD 应用到节点时，如果涉及到修改 systemd 单元或 drop-in 文件，或者改变了单元的启用状态，就会触发 `systemd daemon-reload`。这个重载过程可能会短暂地影响 `kubens.service` 的状态或其暴露的 mount namespace 文件描述符 (`/run/kubens/mnt`) 的有效性。依赖 `kubens.service` 来进入特定 mount namespace 的服务（如 Kubelet, CRI-O），通过 `kubenswrapper` 脚本进行检查。如果在检查时 `kubens.service` 的 namespace 无效，`kubenswrapper` 会跳过进入 namespace 的步骤，导致后续进程在错误的上下文中运行，进而可能引发运行时错误。

因此，虽然报错可能最终出现在 Kubelet 或 CRI-O 等服务中，其根本原因之一是在 MCO 更新触发 `systemd daemon-reload` 时，`kubenswrapper` 无法保证稳定地进入由 `kubens.service` 维护的 mount namespace。
