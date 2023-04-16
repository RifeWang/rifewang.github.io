+++
draft = false
date = 2023-04-16T14:22:36+08:00
title = "Kubernetes Admission Controller 简介 - 注入 sidacar 示例"
description = "Kubernetes Admission Controller 简介 - 注入 sidacar 示例"
slug = ""
authors = []
tags = ["Kubernetes"]
categories = ["Kubernetes"]
externalLink = ""
series = []
disableComments = true
+++

## Admission Controller

Kubernetes Admission Controller（准入控制器）是什么？

如下图所示：

![Admission Controller Phases](https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)

当我们向 k8s api-server 提交了请求之后，需要经过认证鉴权、mutation admission、validation 校验等一系列过程，最后才会将资源对象持久化到 etcd 中（其它组件诸如 controller 或 scheduler 等操作的也是持久化之后的对象）。而所谓的 Kubernetes Admission Controller 其实就是在这个过程中所提供的 webhook 机制，让用户能够在资源对象被持久化之前任意地修改资源对象并进行自定义的校验。

使用 Kubernetes Admission Controller ，你可以：

- 安全性：强制实施整个命名空间或集群范围内的安全规范。例如，禁止容器以root身份运行或确保容器的根文件系统始终以只读方式挂载；只允许从企业已知的特定注册中心拉取镜像，拒绝未知的镜像源；拒绝不符合安全标准的部署。
- 治理：强制遵守某些实践，例如具有良好的标签、注释、资源限制或其他设置。一些常见的场景包括：在不同的对象上强制执行标签验证，以确保各种对象使用适当的标签，例如将每个对象分配给团队或项目，或指定应用程序标签的每个部署；自动向对象添加注释。
- 配置管理：验证集群中对象的配置，并防止任何明显的错误配置影响到您的集群。准入控制器可以用于检测和修复部署了没有语义标签的镜像，例如：自动添加资源限制或验证资源限制；确保向Pod添加合理的标签；确保在生产部署的镜像不使用 latest tag 或带有 -dev 后缀的 tag。

Admission Controller（准入控制器）提供了两种 webhook：

- `Mutation admission webhook`：修改资源对象
- `Validation admission webhook`：校验资源对象

所谓的 webhook 其实就是你需要部署一个 HTTPS Server ，然后 k8s 会将 admission 的请求发送给你的 server，当然你的 server 需要按照约定格式返回响应。

使用 Kubernetes Admission Controller，你需要：

- 确保 k8s 的 api-server 开启 `admission plugins`。
- 准备好 TLS/SSL 证书，用于 HTTPS，可以是自签的。
- 构建自己的 HTTPS server，实现处理逻辑。
- 配置 `MutatingWebhookConfiguration` 或者 `ValidatingWebhookConfiguration`，你得告诉 k8s 怎么跟你的 server 通信。

## 注入 sidacar 示例

接下来，我们来实现一个最简单的为 pod 注入 sidacar 的示例。

### 1. 确保 k8s 的 api-server 开启 admission plugins

首先需要确认你的 k8s 集群支持 admission controller 。

执行 `kubectl api-resources | grep admission`：

```
mutatingwebhookconfigurations       admissionregistration.k8s.io/v1     false   MutatingWebhookConfiguration
validatingwebhookconfigurations     admissionregistration.k8s.io/v1     false   ValidatingWebhookConfiguration
```

得到以上结果就说明你的 k8s 集群支持 admission controller。

然后需要确认 api-server 开启 admission plugins，根据你的 api-server 的启动方式，确认如下参数：

```
--enable-admission-plugins=MutatingAdmissionWebhook,ValidatingAdmissionWebhook
```

plugins 可以有多个，用逗号分隔，本示例其实只需要 `MutatingAdmissionWebhook`，至于其它的 plugins 用途请参考[官方文档](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-on-an-admission-controller)。

### 2. 准备 TLS/SSL 证书

这里我们使用自签的证书，先创建一个证书目录，比如 `~/certs`，以下操作都在这个目录下进行。

1. 创建我们自己的 root CA
    ```
    openssl genrsa -des3 -out rootCA.key 4096
    ```

    ```
    openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.crt
    ```

2. 创建证书
    ```
    openssl genrsa -out mylocal.com.key 2048
    ```

    ```
    openssl req -new -key mylocal.com.key -out mylocal.com.csr
    ```

3. 使用我们自己的 root CA 去签我们的证书
    注意：由于我们会把 HTTPS server 部署在本地进行测试，所以我们在签名的时候要额外指定自己的内网IP。
    ```
    echo subjectAltName = IP:192.168.100.22 > extfile.cnf

    openssl x509 -req -in mylocal.com.csr \
        -CA rootCA.crt -CAkey rootCA.key \
        -CAcreateserial -out mylocal.com.crt \
        -days 500 -extfile extfile.cnf
    ```

执行完后你会得到以下文件：

- `rootCA.key`：根 CA 私钥
- `rootCA.crt`：根 CA 证书（后面 k8s 需要用到）
- `rootCA.srl`：追踪发放的证书
- `mylocal.com.key`：自签域名的私钥（HTTPS server 需要用到）
- `mylocal.com.csr`：自签域名的证书签名请求文件
- `mylocal.com.crt`：自签域名的证书（HTTPS server 需要用到）

### 3. 构建自己的 HTTPS Server

Webhook 的请求和响应都要是 JSON 格式的 [AdmissionReview](https://github.com/kubernetes/api/blob/e2bb673563f3cf77dedec1dedb5645f2eaf6741b/admission/v1/types.go#L29) 对象。

> 注意：AdmissionReview v1 版本和 v1beta1 版本有区别，我们这里使用 v1 版本。

```go
// AdmissionReview describes an admission review request/response.
type AdmissionReview struct {
	metav1.TypeMeta `json:",inline"`
	// Request describes the attributes for the admission request.
	// +optional
	Request *AdmissionRequest `json:"request,omitempty" protobuf:"bytes,1,opt,name=request"`
	// Response describes the attributes for the admission response.
	// +optional
	Response *AdmissionResponse `json:"response,omitempty" protobuf:"bytes,2,opt,name=response"`
}
```

我们需要处理的逻辑其实就是解析 `AdmissionRequest`，然后构造 `AdmissionResponse` 最后返回响应。

```go
// AdmissionResponse describes an admission response.
type AdmissionResponse struct {
	// UID is an identifier for the individual request/response.
	// This must be copied over from the corresponding AdmissionRequest.
	UID types.UID `json:"uid" protobuf:"bytes,1,opt,name=uid"`

	// Allowed indicates whether or not the admission request was permitted.
	Allowed bool `json:"allowed" protobuf:"varint,2,opt,name=allowed"`

	// The patch body. Currently we only support "JSONPatch" which implements RFC 6902.
	// +optional
	Patch []byte `json:"patch,omitempty" protobuf:"bytes,4,opt,name=patch"`

	// The type of Patch. Currently we only allow "JSONPatch".
	// +optional
	PatchType *PatchType `json:"patchType,omitempty" protobuf:"bytes,5,opt,name=patchType"`

    // ...
}
```

`AdmissionResponse` 中的 `PatchType` 字段必须是 `JSONPatch`，`Patch` 字段必须是 [rfc6902 JSON Patch](https://www.rfc-editor.org/rfc/rfc6902) 格式。

我们使用 go 编写一个最简单的 HTTPS Server 示例如下，该示例会修改 pod 的 spec.containers 数组，向其中追加一个 sidecar 容器：

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"

	v1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
)

// patchOperation is an operation of a JSON patch, see https://tools.ietf.org/html/rfc6902 .
type patchOperation struct {
	Op    string      `json:"op"`
	Path  string      `json:"path"`
	Value interface{} `json:"value,omitempty"`
}

var (
	certFile = "/Users/wy/certs/mylocal.com.crt"
	keyFile  = "/Users/wy/certs/mylocal.com.key"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		defer req.Body.Close()
		var admissionReview v1.AdmissionReview
		err := json.NewDecoder(req.Body).Decode(&admissionReview)
		if err != nil {
			log.Fatal(err)
		}

		var patches []patchOperation
		patches = append(patches, patchOperation{
			Op:   "add",
			Path: "/spec/containers/-",
			Value: &corev1.Container{
				Image: "busybox",
				Name:  "sidecar",
			},
		})
		patchBytes, err := json.Marshal(patches)
		if err != nil {
			log.Fatal(err)
		}

		var PatchTypeJSONPatch v1.PatchType = "JSONPatch"
		admissionReview.Response = &v1.AdmissionResponse{
			UID:       admissionReview.Request.UID,
			Allowed:   true,
			Patch:     patchBytes,
			PatchType: &PatchTypeJSONPatch,
		}
		// Return the AdmissionReview with a response as JSON.
		bytes, err := json.Marshal(&admissionReview)
		if err != nil {
			log.Fatal(err)
		}
		w.Write(bytes)
	})

	log.Printf("About to listen on 8443. Go to https://127.0.0.1:8443/")
	err := http.ListenAndServeTLS(":8443", certFile, keyFile, nil)
	log.Fatal(err)
}
```

### 4. 配置 MutatingWebhookConfiguration

我们需要告诉 k8s 往哪里发送请求以及其它信息，这就需要配置 `MutatingWebhookConfiguration`。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: test-sidecar-injector
webhooks:
  - name: sidecar-injector.mytest.io
    admissionReviewVersions:
      - v1  # 版本一定要与 HTTPS Server 处理的版本一致
    sideEffects: "NoneOnDryRun"
    reinvocationPolicy: "Never"
    timeoutSeconds: 30
    objectSelector: # 选择特定资源触发 webhook
      matchExpressions:
        - key: run
          operator: In
          values:
            - "nginx"
    rules:  # 触发规则
      - apiGroups:
          - ""
        apiVersions:
          - v1
        operations:
          - CREATE
        resources:
          - pods
        scope: "*"
    clientConfig:
      caBundle: ${CA_PEM_B64}
      url: https://192.168.100.22:8443/   # 指向我本地的IP地址
      # service:  # 如果把 server 部署到集群内部则可以通过 service 引用
```

其中的 ${CA_PEM_B64} 需要填入第一步的 rootCA.crt 文件的 base64 编码，我们可以执行以下命令得到：

```
openssl base64 -A -in rootCA.crt
```

在上例中，我们还配置了 webhook 触发的资源要求和规则，比如这里的规则是创建 pods 并且 pod 的 labels 标签必须满足 `matchExpressions` 。

最后测试，我们可以执行 `kubectl run nginx --image=nginx`，成功之后再查看提交的 pod ，你会发现 containers 中包含有我们注入的 sidecar 。

## 结语

通过本文相信你已经了解了 Admission Controller 的基本使用过程，诸多开源框架，比如 Istio 等也广泛地使用了 Admission Controller。