Helm ライブラリ チャートを使用した Kubernetes の抽象化

この抽象化のアイデアは、私が大好きなTerraformモジュールから生まれました。また、この記事ではK8sやHelmの長所と短所については深く掘り下げません。

前提条件: Kubernetes と Helm に関する基本的な知識。

> この記事は、開発者の数よりも多くの製品を持つ企業向けです。

## コンテンツ

- Helmライブラリの紹介
- これらがない場合のプロジェクトコード
- 独自のHelmライブラリを設計する方法
- ライブラリを使用するとプロジェクト コードがどのようになるか。

## 問題の説明

多くの中小企業では、チームや製品ごとに専任のDevOpsエンジニアがいません。K8sを使う上で、これは必須事項となります。Schmiede.oneがK8sへの移行を決定した際、クラスタ管理の第一候補は間違いなくHelmでした。しかし、私たちは繰り返し発生する問題に直面しました。それは、各プロジェクトにボイラープレートコードが存在することです。すべてのプロジェクトメンテナーがこれほど多くのHelmコードに精通していることを期待し、管理するのは現実的ではないと感じました。

完全な制御を維持しながら、各製品またはチームに最小のプラグアンドプレイ ソリューションを提供することを目指しました。

> ユニットを構成できるソリューションは純金です

## Helm ライブラリチャートの紹介

Helmのメンテナー自身からの 適切だと確信しています。それが私にとって何を意味したかをお話ししたいと思います。

> ライブラリチャートは、膨大な定型コードを使わずに多数のインフラストラクチャユニットを構成できるK8sテンプレートです。

ChatGPTの言う通り、

> ライブラリ チャートは、Helm の世界では「愛 (とコード) を共有し、チャートを DRY に保ちましょう!」と言う方法です。

## 深掘り

> 記事を書くときに段落構成や直感的な小見出しを考えるのが面倒で、プレゼンテーションの準備をしているような気分です。先延ばし癖がすっかり戻ってきました。どこかのパラレルワールドでは、アイデアが明確な順序もなく散らばっていて、読者はなんとか理解しているようです。念のため言っておきますが、私はハイになっているわけでも、人々が何年もかけて理解しようとして、それでもなお鑑賞に堪えられるような、一見ランダムな傑作を絵画で生み出せるアーティストを羨んでいるわけでもありません。

いくつか例を挙げてみましょう。Kubernetesリソースを使ってデプロイされ、利用可能になるサービスを想像してみてください。

- 構成マップ
- 秘密
- サービス
- 展開
- 仮想サービス
![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*Tx6UNG1O1Xy_f3hVwuALZQ.png)

テンプレートのフォルダ構造が必要

もちろん、Helm ライブラリがない場合、通常、ボイラープレート コードがどれだけ多くなるかを誇張して言うつもりです。

```hs
---
# Source: example/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-secrets
  namespace: schmiede
  annotations:
    checksum/config: 72bf64258899b4e6b1d30f4
type: Opaque
data:
  API_SECRET: VUc2TUJPVFAzQkNSNUZ
  DB_CONNECTION_STRING: c3Fsc2VydmVyOi8vZGNqLsaddW1pY3Jvc2VydmljZ
---
# Source: example/templates/config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-config-map
  namespace: schmiede
  annotations:
    checksum/config: 1b37f1fb4c023342ef05772028368100f93d28065e323e
data:
  CACHING_ENABLED: "1"
  SERVICE_A_API: https://service-a.azurewebsites.net/api
  SERVICE_B_API: https://service-b.azurewebsites.net/api
---
# Source: example/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-service
  namespace: schmiede
spec:
  selector:
    app: sample
  ports:
    - name: http
      port: 80
      targetPort: 8000
---
# Source: example/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment
  namespace: schmiede
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample
  template:
    metadata:
      labels:
        app: sample
      annotations:
        checksum/config: 1b37f1fb4c023d2ef0a4a45f5772028368100f93d28065e323e
        checksum/secrets: 72bf64258899b9e21f5c58e02ad5c85d7ac29d30f4
    spec:
      containers:
        - name: sample
          image: schmiede/sample:test
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: sample-secrets
            - configMapRef:
                name: sample-config-map
          livenessProbe:
            httpGet:
              path: /health/ping
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/metrics
              port: 8000
            initialDelaySeconds: 20
            periodSeconds: 10
          resources:
            limits:
              cpu: 1
              memory: 512Mi
            requests:
              cpu: 0.5
              memory: 256Mi
      imagePullSecrets:
        - name: docker-schmiede-secrets
---
# Source: example/templates/virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: sample-virtual-service
  namespace: schmiede
spec:
  hosts:
    - schmiede.one
  gateways:
    - istio-ingress/generic-gateway
  http:
    - route:
        - destination:
            host: sample-service
            port:
              number: 80
```

Imagine having to maintain these requirements in each project. All developers needed to provide were details about the domain/subdomain for hosting the service, some Docker information, and certain standards that could be set as default values.

**Let’s build the Helm Library**

> I hope I’ve effectively conveyed my excitement about how, in the end, it will be used and how it will seamlessly integrate with any project.

Again, I would suggest you to follow to learn how to begin with libraries then come here.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*SKs3Q22oDBixEzX8_RNvOg.png)

helm library structure

Let’s start from from the main

```hs
---
# Source: helm-lib/Chart.yaml
apiVersion: v2
name: helm-lib
description: A Helm chart for Schmiede APIs
type: library
version: 0.1.15
appVersion: "0.1.0"
```

And then all the templates we might need

```hs
---
# Source: helm-lib/templates/_secrets.yaml
{{- define "helm-lib.secrets" -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Values.app_name }}-secrets
  namespace: {{ default "dcj-service-test" .Values.namespace }}
  annotations:
    checksum/config: {{ toYaml .Values.secrets | sha256sum }}
type: Opaque
data: {{ toYaml .Values.secrets | nindent 2 }}
{{- end -}}
---
# Source: helm-lib/templates/config-map.yaml
{{- define "helm-lib.config-map" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.app_name }}-config-map
  namespace: {{ default "dcj-service-test" .Values.namespace }}
  annotations:
    checksum/config: {{ toYaml .Values.environment | sha256sum }}
data: {{ toYaml .Values.environment | nindent 2 }}
{{- end -}}
---
# Source: helm-lib/templates/service.yaml
{{- define "helm-lib.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app_name }}-service
  namespace: {{ default "dcj-service-tst" .Values.namespace }}
spec:
  selector:
    app: {{ .Values.app_name }}
  ports:
  - name: http
    port: 80
    targetPort: 8000
{{- end -}}
---
# Source: helm-lib/templates/deployment.yaml
{{- define "helm-lib.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app_name }}-deployment
  namespace: {{ default "dcj-service-test" .Values.namespace }}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Values.app_name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app_name }}
      annotations:
        checksum/config: {{ toYaml .Values.environment | sha256sum }}
        checksum/secrets: {{ toYaml .Values.secrets | sha256sum }}
    spec:
      containers:
      - name: {{ .Values.app_name }}
        image: {{ .Values.container.image_name }}
        imagePullPolicy: Always
        ports:
        - containerPort: {{ .Values.container.port }}
        envFrom:
        - secretRef:
            name: {{ .Values.app_name }}-secrets
        - configMapRef:
            name: {{ .Values.app_name }}-config-map
        livenessProbe: {{ toYaml .Values.container.livenessProbe | nindent 10 }}
        readinessProbe: {{ toYaml .Values.container.readinessProbe | nindent 10 }}
        resources:
          limits:
            cpu: 1
            memory: 512Mi
          requests:
            cpu: 0.5
            memory: 256Mi
      imagePullSecrets:
        - name: {{ .Values.container.docker_secret }}
{{- end -}}
---
# Source: helm-lib/templates/virtual-service.yaml
{{- define "helm-lib.virtual-service" -}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: {{ .Values.app_name }}-virtual-service
  namespace: {{ default "dcj-service-test" .Values.namespace }}
spec:
  hosts:
  - {{ default (printf "%s.codekauf.io" .Values.app_name) .Values.networking.domain }}
  gateways:
  - {{ default "dcj-service/generic-gateway" .Values.networking.gateway }}
  http:
  - route:
    - destination:
        host: {{ .Values.app_name }}-service
        port:
          number: 80
{{- end -}}
```

**Usage**

We’re now approaching the best part: understanding how much code applications would need to use it while retaining control over composition. This is essential because each application may have different resource requirements and behaviour like liveness probes. Some applications might not even require secrets and config maps.

In previous step we created a helm library, now we can use them in our simple application chart.

![](https://miro.medium.com/v2/resize:fit:640/format:webp/1*CyfR_iamv3myy_3H8Kt5gw.png)

project chart using helm library

```hs
---
# Source: services-products/Chart.yaml
apiVersion: v2
name: services-products
description: This
type: application
version: 0.1.1
appVersion: "0.1.0"
dependencies:
  - name: helm-lib
    version: 0.1.15
    repository: file:///Users/mukku/work/helm-lib
---
# Source: services-products/templates/app.yaml
# Pick all the templates you need in your application from the library
{{ - include "helm-lib.secrets" . }}
---
{{ - include "helm-lib.config-map" . }}
---
{{ - include "helm-lib.deployment" . }}
---
{{ - include "helm-lib.service" . }}
---
{{ - include "helm-lib.virtual-service" . }}
---
# Source: services-products/values.yaml
replicaCount: 1
app_name: "services-products"
namespace: "schmiede"
networking:
  domain: "schmiede.one"
  gateway: "istio-ingress/generic-gateway"
container:
  image_name: "schmiede/services-products:test"
  port: "8000"
  docker_secret: "docker-schmiede-secrets"
  livenessProbe:
    httpGet:
      path: /health/ping
      port: 8000
    initialDelaySeconds: 10
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /health/metrics
      port: 8000
    initialDelaySeconds: 20
    periodSeconds: 10
secrets:
  DB_CONNECTION_STRING: c3Fsc2VydmVyOi8vZGNqLsaddW1pY3Jvc2VydmljZ
environment:
  CACHING_ENABLED: "1"
  SERVICE_A_API: https://service-a.azurewebsites.net/api
  SERVICE_B_API: https://service-b.azurewebsites.net/api
```

And that’s it.

Most significant part here is the **app.yaml** file which let’s you compose what you need. Project maintainer wouldn’t need to know how the infra is setup.

Sample repo always helps right? Try out [this](https://github.com/johnwatson484/helm-library) one I found on googling “helm library example repo”.
