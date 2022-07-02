# kubebuilder-webhook
```text
测试webhook 功能 当  大于5时 变成5 小于5报错  等于5不做操作
```

1. 初始化项目
```shell
kubebuilder init --domain demo.io --repo github.com/ylinyang/kubebuilder-webhook
kubebuilder create api --group test --version v1 --kind App
```