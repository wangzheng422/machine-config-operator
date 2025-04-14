以下是在一个openshift 4.12 节点上查看 kubelet 的 systemd 文件内容和相关的脚本内容，可以看到kubens是没有被调用的

```bash

[root@ip-10-0-181-94 /]# systemctl cat kubelet
# /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Wants=rpc-statd.service network-online.target
Requires=crio.service kubelet-auto-node-size.service
After=network-online.target crio.service kubelet-auto-node-size.service
After=ostree-finalize-staged.service

[Service]
Type=notify
ExecStartPre=/bin/mkdir --parents /etc/kubernetes/manifests
ExecStartPre=/bin/rm -f /var/lib/kubelet/cpu_manager_state
ExecStartPre=/bin/rm -f /var/lib/kubelet/memory_manager_state
EnvironmentFile=/etc/os-release
EnvironmentFile=-/etc/kubernetes/kubelet-workaround
EnvironmentFile=-/etc/kubernetes/kubelet-env
EnvironmentFile=/etc/node-sizing.env

ExecStart=/usr/local/bin/kubenswrapper \
    /usr/bin/kubelet \
      --config=/etc/kubernetes/kubelet.conf \
      --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig \
      --kubeconfig=/var/lib/kubelet/kubeconfig \
      --container-runtime=remote \
      --container-runtime-endpoint=/var/run/crio/crio.sock \
      --runtime-cgroups=/system.slice/crio.service \
      --node-labels=node-role.kubernetes.io/control-plane,node-role.kubernetes.io/master,node.openshift.io/os_id=${ID} \
      --node-ip=${KUBELET_NODE_IP} \
      --minimum-container-ttl-duration=6m0s \
      --cloud-provider=aws \
      --volume-plugin-dir=/etc/kubernetes/kubelet-plugins/volume/exec \
       \
      --hostname-override=${KUBELET_NODE_NAME} \
      --provider-id=${KUBELET_PROVIDERID} \
      --register-with-taints=node-role.kubernetes.io/master=:NoSchedule \
      --pod-infra-container-image=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:7a8000d1184ad703b24f91fa0e1623e17ce14917e711bdc2b8c172aad3bba7c4 \
      --system-reserved=cpu=${SYSTEM_RESERVED_CPU},memory=${SYSTEM_RESERVED_MEMORY},ephemeral-storage=${SYSTEM_RESERVED_ES} \
      --v=${KUBELET_LOG_LEVEL}

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/kubelet.service.d/01-kubens.conf
# vim:set ft=systemd :
#
# This drop-in will enable any service built with this
# github.com/containers/kubemntns library to properly join the mount namespace
# managed by kubens.service
#

[Unit]
After=kubens.service

[Service]
EnvironmentFile=-/run/kubens/env

# /etc/systemd/system/kubelet.service.d/10-mco-default-env.conf

# /etc/systemd/system/kubelet.service.d/10-mco-default-madv.conf
[Service]
Environment="GODEBUG=x509ignoreCN=0,madvdontneed=1"

# /etc/systemd/system/kubelet.service.d/20-aws-node-name.conf
[Service]
Environment="KUBELET_NODE_NAME=ip-10-0-181-94.us-east-2.compute.internal"

# /etc/systemd/system/kubelet.service.d/20-aws-providerid.conf
[Service]
Environment="KUBELET_PROVIDERID=aws:///us-east-2b/i-0f6844e38fcb7e8cb"

# /etc/systemd/system/kubelet.service.d/20-logging.conf
[Service]
Environment="KUBELET_LOG_LEVEL=2"





[root@ip-10-0-181-94 /]# cat /usr/local/bin/kubenswrapper
#!/bin/sh
if [ -x /usr/bin/kubensenter ]; then
  exec /usr/bin/kubensenter "$@"
else
  exec "$@"
fi





[root@ip-10-0-181-94 /]# systemctl cat kubens
# /etc/systemd/system/kubens.service
[Unit]
Description=Manages a mount namespace for kubernetes-specific mounts

[Service]
Type=oneshot
RemainAfterExit=yes
RuntimeDirectory=kubens
Environment=RUNTIME_DIRECTORY=%t/kubens
Environment=BIND_POINT=%t/kubens/mnt
Environment=ENVFILE=%t/kubens/env

# Set up the runtime directory as an unbindable mountpoint
ExecStartPre=bash -c "findmnt ${RUNTIME_DIRECTORY} || mount --make-unbindable --bind ${RUNTIME_DIRECTORY} ${RUNTIME_DIRECTORY}"
# Ensure the bind point exists
ExecStartPre=touch ${BIND_POINT}
# Use 'unshare' to create the new mountpoint, then 'mount --make-rshared' so it cascades internally
ExecStart=unshare --mount=${BIND_POINT} --propagation slave mount --make-rshared /
# Finally, set an env pointer for ease-of-use
ExecStartPost=bash -c 'echo "KUBENSMNT=${BIND_POINT}" > "${ENVFILE}"'

# On stop, a recursive unmount cleans up the namespace and bind-mounted unbindable parent directory
ExecStop=umount -R ${RUNTIME_DIRECTORY}

[Install]
WantedBy=multi-user.target





[root@ip-10-0-181-94 /]# systemctl status kubelet
● kubelet.service - Kubernetes Kubelet
   Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─01-kubens.conf, 10-mco-default-env.conf, 10-mco-default-madv.conf, 20-aws-node-name.conf, 20-aws-providerid.conf, 20-logging.conf
   Active: active (running) since Mon 2025-04-14 01:13:15 UTC; 24min ago
 Main PID: 2535 (kubelet)
    Tasks: 74 (limit: 402128)
   Memory: 663.4M
      CPU: 6min 1.823s
   CGroup: /system.slice/kubelet.service
           └─2535 /usr/bin/kubelet --config=/etc/kubernetes/kubelet.conf --bootstrap-kubeconfig=/etc/kubernetes/kubeconfig --kubeconfig=/var/lib/kubelet/kubeconf>

Apr 14 01:37:13 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:13.179224    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>
Apr 14 01:37:13 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:13.179258    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>
Apr 14 01:37:15 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:15.084185    2535 kubelet_getters.go:182] "Pod status updated" pod="openshift-kube-scheduler/open>
Apr 14 01:37:15 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:15.084218    2535 kubelet_getters.go:182] "Pod status updated" pod="openshift-kube-apiserver/kube>
Apr 14 01:37:15 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:15.084245    2535 kubelet_getters.go:182] "Pod status updated" pod="openshift-kube-controller-man>
Apr 14 01:37:15 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:15.084256    2535 kubelet_getters.go:182] "Pod status updated" pod="openshift-etcd/etcd-ip-10-0-1>
Apr 14 01:37:23 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:23.178867    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>
Apr 14 01:37:23 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:23.178897    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>
Apr 14 01:37:29 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:29.179423    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>
Apr 14 01:37:29 ip-10-0-181-94 kubenswrapper[2535]: I0414 01:37:29.179471    2535 kubelet_pods.go:897] "Unable to retrieve pull secret, the image pull may not su>




[root@ip-10-0-181-94 /]# systemctl status kubens
● kubens.service - Manages a mount namespace for kubernetes-specific mounts
   Loaded: loaded (/etc/systemd/system/kubens.service; disabled; vendor preset: disabled)
   Active: inactive (dead)


   
```