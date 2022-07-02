# kubebuilder-webhook
```text
测试webhook 功能 当  大于5时 变成5 小于5报错  等于5不做操作
```
而对于我们自定义的需求，可以通过MutatingAdmissionWebhook 和ValidatingAdmissionWebhook 进行处理。
1. 初始化项目
```shell
kubebuilder init --domain demo.io --repo github.com/ylinyang/kubebuilder-webhook
kubebuilder create api --group test --version v1 --kind App
```
2. 初始化webhook
```shell
# 在api功能测试完成基础上
kubebuilder create webhook --group test --version v1 --kind App --defaulting --programmatic-validation
```
同时会生成下列文件，主要有：
- api/v1beta1/app_webhook.go webhook对应的handler，我们添加业务逻辑的地方
- api/v1beta1/webhook_suite_test.go 测试
- config/certmanager 自动生成自签名的证书，用于webhook server提供https服务
- config/webhook 用于注册webhook到k8s中
- config/crd/patches 为conversion自动注入caBoundle
- config/default/manager_webhook_patch.yaml 让manager的deployment支持webhook请求
- config/default/webhookcainjection_patch.yaml 为webhook server注入caBoundle

3. 配置修改

```shell
# 为了支持webhook，我们需要修改config/default/kustomization.yaml将相应的配置打开，具体可参考注释。
# Adds namespace to all resources.
namespace: kubebuilder-demo-system

# Value of this field is prepended to the
# names of all resources, e.g. a deployment named
# "wordpress" becomes "alices-wordpress".
# Note that it should also match with the prefix (text before '-') of the namespace
# field above.
namePrefix: kubebuilder-demo-

# Labels to add to all resources and selectors.
#commonLabels:
#  someName: someValue

bases:
- ../crd
- ../rbac
- ../manager
# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
# crd/kustomization.yaml
- ../webhook
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
- ../certmanager
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
#- ../prometheus

patchesStrategicMerge:
# Protect the /metrics endpoint by putting it behind auth.
# If you want your controller-manager to expose the /metrics
# endpoint w/o any authn/z, please comment the following line.
- manager_auth_proxy_patch.yaml

# Mount the controller dev file for loading manager configurations
# through a ComponentConfig type
#- manager_config_patch.yaml

# [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
# crd/kustomization.yaml
- manager_webhook_patch.yaml

# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'.
# Uncomment 'CERTMANAGER' sections in crd/kustomization.yaml to enable the CA injection in the admission webhooks.
# 'CERTMANAGER' needs to be enabled to use ca injection
- webhookcainjection_patch.yaml

# the following dev is for teaching kustomize how to do var substitution
vars:
# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
- name: CERTIFICATE_NAMESPACE # namespace of the certificate CR
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1
    name: serving-cert # this name should match the one in certificate.yaml
  fieldref:
    fieldpath: metadata.namespace
- name: CERTIFICATE_NAME
  objref:
    kind: Certificate
    group: cert-manager.io
    version: v1
    name: serving-cert # this name should match the one in certificate.yaml
- name: SERVICE_NAMESPACE # namespace of the service
  objref:
    kind: Service
    version: v1
    name: webhook-service
  fieldref:
    fieldpath: metadata.namespace
- name: SERVICE_NAME
  objref:
    kind: Service
    version: v1
    name: webhook-service
```
4. 编写webhook的业务逻辑

# 本地测试
1. 安装cert-manager
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
```
```shell
# 配置config/dev文件
# 添加makefile配置信息
.PHONY: dev
dev: manifests kustomize ## Deploy controller to the K8s cluster specified in ~/.kube/config.
	cd config/manager && $(KUSTOMIZE) edit set image controller=${IMG}
	$(KUSTOMIZE) build config/dev | kubectl apply -f -
.PHONY: undev
undev: manifests kustomize ## Deploy controller to the K8s cluster specified in ~/.kube/config.
	$(KUSTOMIZE) build config/dev | kubectl delete --ignore-not-found=$(ignore-not-found) -f -
```
2. 编译
```shell
make dev
```
3. 获取临时证书
```shell
kubectl get secrets webhook-server-cert -n  kubebuilder-webhook-system -o jsonpath='{..tls\.crt}' |base64 -d > certs/tls.crt
kubectl get secrets webhook-server-cert -n  kubebuilder-webhook-system -o jsonpath='{..tls\.key}' |base64 -d > certs/tls.key
```
4. 运行\
需要配置环境变量 ENVIRONMENT=DEV