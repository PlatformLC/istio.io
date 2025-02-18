---
title: 使用 Telemetry API 自定义 Istio 监控指标
description: 这个任务向您展示如何使用 Telemetry API 自定义 Istio 监控指标。
weight: 10
keywords: [telemetry,metrics,customize]
owner: istio/wg-policies-and-telemetry-maintainers
test: no
---

Telemetry API 现在已经成为 Istio 中的主流API。
之前，用户必须在 Istio 监控配置部分 `telemetry` 指标。
本任务将向您展示如何使用 Telemetry API 自定义生成 Istio 的遥测指标。

## 开始之前{#before-you-begin}

在集群中[安装 Istio](/zh/docs/setup/) 并部署一个应用。
需要注意的是，Telemetry API 无法与 `EnvoyFilter` 一起使用。
有关更多详细信息，请查看此问题 [issue](https://github.com/istio/istio/issues/39772)。
从 Istio 版本 `1.18` 开始，默认情况下不会安装 `EnvoyFilter`。

对于 Istio `1.18` 之前的版本，您应该使用以下的 `IstioOperator` 配置进行安装：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    telemetry:
      enabled: true
      v2:
        enabled: false
{{< /text >}}

## 覆盖指标{#override-metrics}

`metrics` 部分提供了指标维度的表达式值，并允许您移除或覆盖现有的指标维度。
您可以使用 `tags_to_remove` 或重新定义维度来修改标准指标的定义。

1. 从 `REQUEST_COUNT` 指标中删除 `grpc_response_status` 标签

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: remove-tags
      namespace: istio-system
    spec:
      metrics:
        - providers:
            - name: prometheus
          overrides:
            - match:
                mode: CLIENT_AND_SERVER
                metric: REQUEST_COUNT
              tagOverrides:
                grpc_response_status:
                  operation: REMOVE
    {{< /text >}}

1. 为 `REQUEST_COUNT` 指标添加自定义标签

    Telemetry API 无法更新 `MeshConfig` 中的 `extraStatTags`，
    您需要手动更新 `extraStatTags` 并滚动重新部署。

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: custom-tags
      namespace: istio-system
    spec:
      metrics:
        - overrides:
            - match:
                metric: ALL_METRICS
                mode: CLIENT
              tagOverrides:
                destination_x:
                  value: upstream_peer.labels['app'].value
            - match:
                metric: ALL_METRICS
                mode: SERVER
              tagOverrides:
                source_x:
                  value: downstream_peer.labels['app'].value
          providers:
            - name: prometheus
    {{< /text >}}

## 禁用指标{#disable-metrics}

1. 通过以下配置禁用所有指标：

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: remove-all-metrics
      namespace: istio-system
    spec:
      metrics:
        - providers:
            - name: prometheus
          overrides:
            - disabled: true
              match:
                mode: CLIENT_AND_SERVER
                metric: ALL_METRICS
    {{< /text >}}

1. 通过以下配置禁用 `REQUEST_COUNT` 指标：

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: remove-request-count
      namespace: istio-system
    spec:
      metrics:
        - providers:
            - name: prometheus
          overrides:
            - disabled: true
              match:
                mode: CLIENT_AND_SERVER
                metric: REQUEST_COUNT
    {{< /text >}}

1. 通过以下配置禁用客户端的 `REQUEST_COUNT` 指标：

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: remove-client
      namespace: istio-system
    spec:
      metrics:
        - providers:
            - name: prometheus
          overrides:
            - disabled: true
              match:
                mode: CLIENT
                metric: REQUEST_COUNT
    {{< /text >}}

1. 通过以下配置禁用服务端的 `REQUEST_COUNT` 指标：

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1alpha1
    kind: Telemetry
    metadata:
      name: remove-server
      namespace: istio-system
    spec:
      metrics:
        - providers:
            - name: prometheus
          overrides:
            - disabled: true
              match:
                mode: SERVER
                metric: REQUEST_COUNT
    {{< /text >}}
