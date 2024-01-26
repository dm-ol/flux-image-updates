# Налаштування автоматичного оновлення образу за допомогою Flux
1. Встановлюємо kind:
```shell
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```
2. Встановлюємо flux cd:
```shell
curl -s https://fluxcd.io/install.sh | sudo bash
```
3. Створюємо кластер за допомогою kind:
```shell
kind create cluster --config=[config.yaml] --name [name]
```
Приклад `config.yaml` на дві ноди:
```yaml
# two node (one workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
#- role: worker
#- role: worker
```
4. Перевіряємо умови для встановлення flux:
```shell
flux check --pre
```
5. Встановлюємо flux до кластеру:
```shell
export GITHUB_TOKEN=[token]

flux bootstrap github --components-extra=image-reflector-controller,image-automation-controller --token-auth --owner=[owner name] --repository=flux-image-updates --branch=main --path=clusters/[cluster name] --read-write-key --personal
```
6. Перевіряємо створення подів і запуск всих систем Flux:
```shell
k get po -A
```
Всі поди запущені і працюють:
```
NAMESPACE            NAME                                           READY   STATUS    RESTARTS   AGE
flux-system          helm-controller-865448769d-xz228               1/1     Running   0          3m21s
flux-system          image-automation-controller-69b75845c5-4ljnj   1/1     Running   0          3m21s
flux-system          image-reflector-controller-84f896568c-ngpff    1/1     Running   0          3m21s
flux-system          kustomize-controller-5c8878fd86-x4g98          1/1     Running   0          3m21s
flux-system          notification-controller-59696fbb58-v8qxl       1/1     Running   0          3m21s
flux-system          source-controller-fc5555fb-2nw8x               1/1     Running   0          3m21s
kube-system          coredns-5d78c9869d-f2l8r                       1/1     Running   0          14m
kube-system          coredns-5d78c9869d-prgcg                       1/1     Running   0          14m
kube-system          etcd-kind-control-plane                        1/1     Running   0          14m
kube-system          kindnet-kzgfd                                  1/1     Running   0          14m
kube-system          kindnet-qllbt                                  1/1     Running   0          14m
kube-system          kube-apiserver-kind-control-plane              1/1     Running   0          14m
kube-system          kube-controller-manager-kind-control-plane     1/1     Running   0          14m
kube-system          kube-proxy-rtcr9                               1/1     Running   0          14m
kube-system          kube-proxy-vjjg7                               1/1     Running   0          14m
kube-system          kube-scheduler-kind-control-plane              1/1     Running   0          14m
local-path-storage   local-path-provisioner-6bc4bddd6b-j62c5        1/1     Running   0          14m
```
7. Клонуємо Flux репозиторій:
```shell
git clone https://github.com/$GITHUB_USER/flux-image-updates
cd flux-image-updates
```
8. Створимо розгортання для бота в кластері `clusters/kind`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
name: kbot
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kbot
  namespace: kbot
spec:
  selector:
    matchLabels:
      app: kbot
  template:
    metadata:
      labels:
        app: kbot
    spec:
      containers:
        - name: kbot
          image: docker.io/devdp/cicd-test:v1.0.0
          imagePullPolicy: IfNotPresent
```
9. Перевіримо інформацію про деплоймент:
```shell
k get deployment/kbot -n kbot -o yaml | grep 'image:'
```
або 
```shell
k describe deploy -n kbot kbot | grep 'Image:'
```
Повинні отримати відповідь з назвою нашого образу:
`Image:        docker.io/devdp/cicd-test:v1.0.0
10. Створюємо `ImageRepository`, щоб вказати Flux де шукати оновлення образів і з яким інтервалом:
```shell
flux create image repository kbot \
--image=docker.io/devdp/cicd-test \
--interval=3m \
--export > ./clusters/kind/kbot-registry.yaml
```
Буде згенерований наступний маніфест:
```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: kbot
  namespace: flux-system
spec:
  image: docker.io/devdp/cicd-test
  interval: 3m0s
```
11. Для приватних зображень ви можете створити секрет Kubernetes у тому самому просторі імен, що й `ImageRepository`with `kubectl create secret docker-registry`. Тоді ви можете налаштувати Flux на використання облікових даних, посилаючись на секрет Kubernetes у `ImageRepository`:
```yaml
kind: ImageRepository
spec:
  secretRef:
    name: regcred
```
12. Створюємо `ImagePolicy`, щоб вказати Flux, який діапазон версій використовувати під час фільтрації тегів:
```shell
flux create image policy kbot \
--image-ref=kbot \
--select-semver=1.x \
--export > ./clusters/kind/podinfo-policy.yaml
```
Наведена вище команда створює такий маніфест:
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: kbot
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: kbot
  policy:
    semver:
      range: '>=1.0.0'
```
При потребі, можна додати фільтри або правила для тегів: https://fluxcd.io/flux/guides/image-update/#imagepolicy-examples
13. Зафіксуйте та надішліть зміни до головної гілки та зачекайте, поки Flux отримає список тегів зображень із реєстру контейнерів. Потім можна перевірити список доступних тегів:
```shell
flux get image repository kbot
```
```shell
k -n flux-system describe imagerepositories kbot
```
14.  Налаштуємо оновлення образів. Оновлюємо `kbot-deployment.yaml`та додаєм маркер для Flux, яку політику використовувати під час оновлення зображення контейнера:
```yaml
spec:
  containers:
  - name: podinfod
    image: docker.io/devdp/cicd-test:v1.0.0 # {"$imagepolicy": "flux-system:kbot"}
```
15. Створюємо `ImageUpdateAutomation`, щоб вказати Flux, до якого репозиторію Git і з яким інтервалом записувати оновлення зображень:
```shell
flux create image update kbot \                 
--interval=3m \
--git-repo-ref=flux-system \
--git-repo-path="./clusters/kind/kbot" \
--checkout-branch=main \
--push-branch=main \
--author-name=fluxcdbot \
--author-email=fluxcdbot@users.noreply.github.com \
--commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
--export > ./clusters/kind/kbot/flux-system-automation.yaml
```
Наведена вище команда створює такий маніфест:
```yaml
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: kbot
  namespace: flux-system
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@users.noreply.github.com
        name: fluxcdbot
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: main
  interval: 3m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  update:
    path: ./clusters/kind/kbot
    strategy: Setters
```
16. Перевіряємо що все працює правильно:
```shell
flux get image repository kbot
flux get image policy kbot
flux get image update kbot
```
або
```shell
flux get images all --all-namespaces
```
17. Якщо все вірно налаштовано, Flux підтягне оновлену версію образу з репозиторію і оновить її в кластері:
```shell
flux get image repository kbot            
flux get image policy kbot
flux get image update kbot
NAME	LAST SCAN                	SUSPENDED	READY	MESSAGE                       
kbot	2024-01-26T16:43:43+02:00	False    	True 	successful scan: found 2 tags	
NAME	LATEST IMAGE                   	READY	MESSAGE                                                                      
kbot	docker.io/devdp/cicd-test:1.0.1	True 	Latest image tag for 'docker.io/devdp/cicd-test' updated from 1.0.0 to 1.0.1	
NAME	LAST RUN                 	SUSPENDED	READY	MESSAGE                                                      
kbot	2024-01-26T16:45:43+02:00	False    	True 	no updates made; last commit 5bad63d at 2024-01-26T14:34:40Z
```
