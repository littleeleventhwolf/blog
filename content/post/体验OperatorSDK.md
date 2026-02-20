---
title: '体验OperatorSDK'
date: '2026-02-20T11:27:01+08:00'
lastmod: '2026-02-20T11:27:01+08:00'
keywords: ['kubernetes']
categories: ['kubernetes']
tags: ['kubernetes']
author: '小十一狼'
---

# 前言

本文的代码样例来自 O'REILLY《Kubernetes 编程》第 6 章 编写 Operator 方案 6.4 小节的内容。由于该书出版时，Operator SDK 还处于发展阶段，所以自己在按步骤操作时，还是发现了很多不一样的地方，因此做此记录。

# 本机环境

Apple M4 Pro，Sequoia 15.7.3

# 安装 colima

Colima 是一个开源项目，提供轻量级的容器运行时环境。

```shell
brew install colima docker docker-compose

# 启动 colima
colima start --cpu 6 --memory 10

# 其他管理命令，试验期间无需执行
# 停止 colima
colima stop
```

# 安装 k3d

参考了[如何在本地快速启动一个K8S集群](https://zhuanlan.zhihu.com/p/357907926)。

```shell
brew install k3d kubectl kubecm

# 创建集群
k3d cluster create first-cluster --port 8080:80@loadbalancer --port 8443:443@loadbalancer --api-port 6443 --servers 1 --agents 2

# 其他管理命令，试验期间无需执行
# 停止集群
k3d cluster stop first-cluster
# 重启集群
k3d cluster start first-cluster
```

# 使用 OperatorSDK

这里很多步骤和书上的内容不一样，是最新的试验过程，记录如下：

```shell
brew install operator-sdk

# 创建项目目录并进入
mkdir cnat-operator && cd cnat-operator

# 初始化 go 模块，生成 go.mod 文件，cnat-operator 是模块名称，通常与项目目录名保持一致
go mod init cnat-operator

# 初始化 operator 项目脚手架，创建标准的 Kubernetes Operator 项目结构
# 生成的目录包括：api/（API 定义）、config/（配置文件）、internal/controller/（控制器代码）等
# 同时会创建必要的配置文件，如 Makefile、Dockerfile、PROJECT 文件等
operator-sdk init

# 创建自定义资源（CRD）和对应的控制器
# --group cnat.programming-kubernetes.info: 指定 API 组名，类似于域名的格式
# --version v1alpha1: 指定 API 版本，v1alpha1 表示初始版本
# --kind At: 指定自定义资源的种类名称（Kind），资源的类型名
# 这个命令会：
#   - 在 api/v1alpha1/ 目录下生成 At 类型的定义文件（At 结构体）
#   - 在 internal/controller/ 目录下生成对应的控制器代码
#   - 生成 CRD 的 YAML 配置文件
#   - 创建 RBAC 权限配置
operator-sdk create api --group cnat.programming-kubernetes.info --version v1alpha1 --kind At

# 生成 CRD 和 RBAC 配置
make manifests
```

以上命令执行完后得到的目录结构如下：

```
 ~/MyOpen/cnat-operator ▓▒░ tree
.
├── api
│   └── v1alpha1
│       ├── at_types.go
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── bin
│   ├── controller-gen -> ~/MyOpen/cnat-operator/bin/controller-gen-v0.18.0
│   └── controller-gen-v0.18.0
├── cmd
│   └── main.go
├── config
│   ├── crd
│   │   ├── bases
│   │   │   └── cnat.programming-kubernetes.info.my.domain_ats.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── cert_metrics_manager_patch.yaml
│   │   ├── kustomization.yaml
│   │   ├── manager_metrics_patch.yaml
│   │   └── metrics_service.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── manifests
│   │   └── kustomization.yaml
│   ├── network-policy
│   │   ├── allow-metrics-traffic.yaml
│   │   └── kustomization.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   ├── monitor_tls_patch.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── at_admin_role.yaml
│   │   ├── at_editor_role.yaml
│   │   ├── at_viewer_role.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── metrics_auth_role_binding.yaml
│   │   ├── metrics_auth_role.yaml
│   │   ├── metrics_reader_role.yaml
│   │   ├── role_binding.yaml
│   │   ├── role.yaml
│   │   └── service_account.yaml
│   ├── samples
│   │   ├── cnat.programming-kubernetes.info_v1alpha1_at.yaml
│   │   └── kustomization.yaml
│   └── scorecard
│       ├── bases
│       │   └── config.yaml
│       ├── kustomization.yaml
│       └── patches
│           ├── basic.config.yaml
│           └── olm.config.yaml
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── internal
│   └── controller
│       ├── at_controller_test.go
│       ├── at_controller.go
│       └── suite_test.go
├── Makefile
├── PROJECT
├── README.md
└── test
    ├── e2e
    │   ├── e2e_suite_test.go
    │   └── e2e_test.go
    └── utils
        └── utils.go

24 directories, 52 files
```

接着编写代码逻辑：

这部分和书上 6.3 中的将 kubebuilder 的内容相同。

首先，在 api/v1alpha1/at_types.go 中修改 AtSpec 和 AtStatus 结构体：

```go
const (
	PhasePending = "PENDING"
	PhaseRunning = "RUNNING"
	PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At.
type AtSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Schedule is the desired time the command is supposed to be executed.
	// Note: the format used here is UTC time https://www.utctime.net
	Schedule string `json:"schedule,omitempty"`
	// Command is the desired command (executed in a Bash shell) to be executed.
	Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At.
type AtStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Phase represents the state of the schedule: until the command is executed
	// it is PENDING, afterwards it is DONE.
	Phase string `json:"phase,omitempty"`
}
```

然后，修改 internal/controller/at_controller.go，完整代码如下：

```go
/*
Copyright 2026.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controller

import (
	"context"
	"fmt"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	cnatv1alpha1 "cnat-operator/api/v1alpha1"
)

// AtReconciler reconciles a At object
type AtReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=cnat.programming-kubernetes.info.my.domain,resources=ats,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cnat.programming-kubernetes.info.my.domain,resources=ats/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cnat.programming-kubernetes.info.my.domain,resources=ats/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the At object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.21.0/pkg/reconcile
func (r *AtReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	reqLogger := logf.FromContext(ctx, "namespace", req.Namespace, "at", req.Name)
	reqLogger.Info("=== Reconciling At")
	// Fetch the At instance
	instance := &cnatv1alpha1.At{}
	err := r.Get(context.TODO(), req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after
			// reconcile request - return and don't requeue:
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request
		return reconcile.Result{}, err
	}

	// If no phase set, default to pending (the initial phase):
	if instance.Status.Phase == "" {
		instance.Status.Phase = cnatv1alpha1.PhasePending
	}

	// Now let's make the main case distinction: implementing
	// the state diagram PENDING -> RUNNING -> DONE
	switch instance.Status.Phase {
	case cnatv1alpha1.PhasePending:
		reqLogger.Info("Phase: PENDING")
		// As long as we haven't executed the command yet, we need to check if
		// it's already time to act:
		reqLogger.Info("Checking schedule", "Target", instance.Spec.Schedule)
		// Check if it's already time to execute the command with a tolerance
		// of seconds:
		d, err := timeUntilSchedule(instance.Spec.Schedule)
		if err != nil {
			reqLogger.Error(err, "Schedule parsing failure")
			// Error reading the schedule. Wait until it is fixed.
			return reconcile.Result{}, err
		}
		reqLogger.Info("Schedule parsing done", "Result", fmt.Sprintf("%v", d))
		if d > 0 {
			// Not yet time to execute the command, wait until the scheduled time
			return reconcile.Result{RequeueAfter: d}, nil
		}
		reqLogger.Info("It's time!", "Ready to execute", instance.Spec.Command)
		instance.Status.Phase = cnatv1alpha1.PhaseRunning
	case cnatv1alpha1.PhaseRunning:
		reqLogger.Info("Phase: RUNNING")
		pod := newPodForCR(instance)
		// Set At instance as the owner and controller
		err := controllerutil.SetControllerReference(instance, pod, r.Scheme)
		if err != nil {
			// requeue with error
			return reconcile.Result{}, err
		}
		found := &corev1.Pod{}
		nsName := types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}
		err = r.Get(context.TODO(), nsName, found)
		// Try to see if the pod already exists and if not
		// (which we expect) then create a one-shot pod as per spec:
		if err != nil && errors.IsNotFound(err) {
			err = r.Create(context.TODO(), pod)
			if err != nil {
				// requeue with error
				return reconcile.Result{}, err
			}
			reqLogger.Info("Pod launched", "name", pod.Name)
		} else if err != nil {
			// requeue with error
			return reconcile.Result{}, err
		} else if found.Status.Phase == corev1.PodFailed ||
			found.Status.Phase == corev1.PodSucceeded {
			reqLogger.Info("Container terminated", "reason",
				found.Status.Reason, "message", found.Status.Message)
			instance.Status.Phase = cnatv1alpha1.PhaseDone
		} else {
			// Don't requeue because it will happen automatically when the
			// pod status changes.
			return reconcile.Result{}, nil
		}
	case cnatv1alpha1.PhaseDone:
		reqLogger.Info("Phase: DONE")
		return reconcile.Result{}, nil
	default:
		reqLogger.Info("NOP")
		return reconcile.Result{}, nil
	}

	// Update the At instance, setting the status to the respective phase:
	err = r.Status().Update(context.TODO(), instance)
	if err != nil {
		return reconcile.Result{}, err
	}

	// Don't requeue. We should be reconcile because either the pod
	// or the CR changes.
	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *AtReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&cnatv1alpha1.At{}).
		Named("at").
		Complete(r)
}

func timeUntilSchedule(schedule string) (time.Duration, error) {
	now := time.Now().UTC()
	layout := time.RFC3339
	s, err := time.Parse(layout, schedule)
	if err != nil {
		return time.Duration(0), err
	}
	return s.Sub(now), nil
}

func newPodForCR(cr *cnatv1alpha1.At) *corev1.Pod {
	labels := map[string]string{
		"app": cr.Name,
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name + "-pod",
			Namespace: cr.Namespace,
			Labels:    labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "busybox",
					Image:   "busybox",
					Command: []string{"/bin/sh", "-c", cr.Spec.Command},
				},
			},
			RestartPolicy: corev1.RestartPolicyOnFailure,
		},
	}
}
```

接着，执行以下命令：

```shell
# 重新生成 CRD 和 RBAC 配置
make manifests

# 本地启动运行自定义控制器
# 这个终端窗口就保留下来观察日志，后续的命令切换到别的终端窗口执行
make run
```

最后，修改 config/samples/cnat.programming-kubernetes.info_v1alpha1_at.yaml：

```yaml
apiVersion: cnat.programming-kubernetes.info.my.domain/v1alpha1
kind: At
metadata:
  labels:
    app.kubernetes.io/name: cnat-operator
    app.kubernetes.io/managed-by: kustomize
  name: sample-at
spec:
  schedule: "2026-02-20T04:08:38Z"
  command: "echo WXL"
```

```shell
kubectl apply -f config/samples/cnat.programming-kubernetes.info_v1alpha1_at.yaml
```

观察 `make run` 下面的日志输出：

```
 ~/MyOpen/cnat-operator ▓▒░ make run
~/MyOpen/cnat-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
~/MyOpen/cnat-operator/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go run ./cmd/main.go
2026-02-20T12:07:32+08:00	INFO	setup	starting manager
2026-02-20T12:07:32+08:00	INFO	starting server	{"name": "health probe", "addr": "[::]:8081"}
2026-02-20T12:07:32+08:00	INFO	Starting EventSource	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "source": "kind source: *v1alpha1.At"}
2026-02-20T12:07:32+08:00	INFO	Starting Controller	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At"}
2026-02-20T12:07:32+08:00	INFO	Starting workers	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "worker count": 1}
2026-02-20T12:07:40+08:00	INFO	=== Reconciling At	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "5416fff7-c054-46a1-a409-dae677592ac0", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:07:40+08:00	INFO	Phase: PENDING	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "5416fff7-c054-46a1-a409-dae677592ac0", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:07:40+08:00	INFO	Checking schedule	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "5416fff7-c054-46a1-a409-dae677592ac0", "namespace": "cnat", "at": "sample-at", "Target": "2026-02-20T04:08:38Z"}
2026-02-20T12:07:40+08:00	INFO	Schedule parsing done	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "5416fff7-c054-46a1-a409-dae677592ac0", "namespace": "cnat", "at": "sample-at", "Result": "57.277798s"}
2026-02-20T12:08:38+08:00	INFO	=== Reconciling At	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "48b464c4-17bd-4112-9306-1aa1f0cff845", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:08:38+08:00	INFO	Phase: PENDING	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "48b464c4-17bd-4112-9306-1aa1f0cff845", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:08:38+08:00	INFO	Checking schedule	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "48b464c4-17bd-4112-9306-1aa1f0cff845", "namespace": "cnat", "at": "sample-at", "Target": "2026-02-20T04:08:38Z"}
2026-02-20T12:08:38+08:00	INFO	Schedule parsing done	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "48b464c4-17bd-4112-9306-1aa1f0cff845", "namespace": "cnat", "at": "sample-at", "Result": "-1.392ms"}
2026-02-20T12:08:38+08:00	INFO	It's time!	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "48b464c4-17bd-4112-9306-1aa1f0cff845", "namespace": "cnat", "at": "sample-at", "Ready to execute": "echo WXL"}
2026-02-20T12:08:38+08:00	INFO	=== Reconciling At	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "cfb4e573-e828-47df-ab91-19ed991a3681", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:08:38+08:00	INFO	Phase: RUNNING	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "cfb4e573-e828-47df-ab91-19ed991a3681", "namespace": "cnat", "at": "sample-at"}
2026-02-20T12:08:38+08:00	INFO	Pod launched	{"controller": "at", "controllerGroup": "cnat.programming-kubernetes.info.my.domain", "controllerKind": "At", "At": {"name":"sample-at","namespace":"cnat"}, "namespace": "cnat", "name": "sample-at", "reconcileID": "cfb4e573-e828-47df-ab91-19ed991a3681", "namespace": "cnat", "at": "sample-at", "name": "sample-at-pod"}
```

至此，完成了自定义控制器的开发工作。建议大家可以参考书里的讲解和本文的试验，尝试简单体验一下 Kubernetes Operator。
