## [Назад: Запуск приложения](Oracle Kubernetes Workshop p5.md)

# Работа с кластером Kubernetes

## Пример быстрого обновления контейнера

Сейчас для генерации облака слов используется стандартная сине-зеленая цветовая схема.

Изменим цветовую схему для иллюстрации концепции быстрого развертывания изменений в микросервисах.

Пометьте имеющийся Docker образ **wc** номером версии с цветом:

```bash
$ docker tag $REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc:latest $REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc:1.0-green
```

Отредактируйте файл, который генерирует облако слов.

```bash
$ nano $HOME/workshop/clouddemo-micro/clouddemo-wc/docker/context/app/wc.py
```

Пролистайте вниз до строки **def genwordcloud (fulltext):**

Раскомментируйте строку:

```
#        wordcloud.recolor (color_func=color_func, random_state=3)
```

Эта команда вызывает функцию, изменяющую цвет облака слов.

Сохраните файл (в nano используйте сочетание клавиш Ctrl+O).

Выполните следующие команды:

```bash
$ docker build -t $REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc:1.0-red clouddemo-wc/docker/
```

```
Sending build context to Docker daemon   5.12kB
Step 1/10 : FROM python:3.7-slim
...
Successfully built 90083ee72432
Successfully tagged eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc:1.0-red
```

Можно посмотреть список образов:

```bash
$ docker images
```

```
REPOSITORY                                       TAG                 IMAGE ID            CREATED             SIZE
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc      1.0-red             b123509b7992        7 seconds ago       307MB
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/api     latest              8507fc1fd1a0        9 minutes ago       352MB
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/db      latest              3f762664991b        27 minutes ago      330MB
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc      1.0-green           a053353c1c0f        40 minutes ago      307MB
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc      latest              a053353c1c0f        40 minutes ago      307MB
eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/front   latest              419dceb70b9e        44 minutes ago      95MB
python                                           3.7-slim            69afd9568c9d        2 weeks ago         179MB
debian                                           buster-slim         2f14a0fb67b9        4 weeks ago         69.2MB
```

Выполните следующие команды:

```bash
$ docker push $REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc
```

```
The push refers to repository [eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro-wc]
9130414ff7f5: Pushed
...
```

```bash
$ kubectl set image deployment/wc wc=$REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc:1.0-red --record
```

```
deployment.extensions/wc image updated
```

Через некоторое время обновленный контейнер загрузится и новая версия pod будет запушена поэтапно.

```bash
$ kubectl get pods
```

```
NAME                     READY   STATUS        RESTARTS   AGE
api-7679c7fb4b-c5z2s     1/1     Running       1          13h
api-7679c7fb4b-kz25v     1/1     Running       0          14h
api-7679c7fb4b-nzdgz     1/1     Running       0          13h
api-7679c7fb4b-tgrtf     0/1     Pending       0          13h
db-579b47b499-88q6h      1/1     Running       0          13h
db-579b47b499-8drqx      1/1     Running       0          13h
front-7545444c66-6q59l   1/1     Running       0          13h
front-7545444c66-ttv2r   1/1     Running       0          13h
wc-59577c8b5f-pwtgw      1/1     Terminating   0          13h
wc-59577c8b5f-wpxbn      1/1     Terminating   0          13h
wc-8449dc49b5-cksfr      1/1     Running       0          16s
wc-8449dc49b5-t2m6n      1/1     Running       0          24s
```

После загрузки и обработки новой картинки в приложении облако слов будет перестроено в новой цветовой гамме (облако слов генерируется один раз при загрузке каждой новой картинки).

![](media/p6/image1.png)

## История развертывания и откат к предыдущей версии

Можно вновь изменить образ, выполнив команду:

```bash
$ kubectl set image deployment/wc wc=$REGION.ocir.io/$NAMESPACE/clouddemo-micro/wc:1.0-green --record
```

```
deployment.extensions/wc image updated
```

Теперь можно посмотреть историю развертывания образов. Выполните команду:

```bash
$ kubectl rollout history deployment wc
```

```
deployment.extensions/wc
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment/wc wc=eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc:1.0-red --record=true
3         kubectl set image deployment/wc wc=eu-frankfurt-1.ocir.io/frxhexdipnsp/clouddemo-micro/wc:1.0-green --record=true
```

В результате выполнения команды показаны 3 ревизии в истории развертывания: первоначальная (1), которая имела метку latest, (2) 1.0-red и (3) 1.0-green.

Можно вернуться к предыдущей версии образа, выполнив команду:

```bash
$ kubectl rollout undo deployment wc
```

```
deployment.extensions/wc
```

В результате будет создана новая ревизия развертывания, в которой будет отменено последнее изменение.

Можно вернуться к любой ревизии в сохраненной истории:

```bash
$ kubectl rollout history deployment wc
```

```bash
$ kubectl rollout undo deployment wc --to-revision=1
```

## Масштабирование кластера

Масштабирование pod выполняется следующим образом:

```bash
$ kubectl scale --replicas=1 deployment api
```

```bash
$ kubectl scale --replicas=1 deployment db
```

```bash
$ kubectl scale --replicas=1 deployment front
```

```bash
$ kubectl scale --replicas=1 deployment wc
```

```bash
$ kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
api-79d94d5cf9-9ztnb     1/1     Running   0          4m33s
db-5787f46fdb-x9dfq      1/1     Running   0          4m33s
front-75f59468cd-klwzd   1/1     Running   0          4m33s
wc-56947b8cf7-kf8fj      1/1     Running   0          4m34s
```

Масштабирование узлов кластера производится следующим образом:

![](media/p6/image2.png)На странице информации о кластере пролистайте ниже до раздела **Node Pools** и нажмите **Actions / Scale**.

В открывшемся окне укажите требуемое количество рабочих узлов (например, 4), пролистайте вниз и нажмите **Scale**.

Будут созданы (или удалены) узлы, чтобы общее их количество равнялось требуемому.

Если установить размер Node Pool в 0, то рабочие узлы этого пула будут удалены, нагрузка с них убрана (а в отсутствии других пулов все pod будут поставлены в режим Pending), однако кластер удален не будет.

Также можно масштабировать кластер командой oci cli:

```bash
$ oci ce node-pool update --node-pool-id <Paste your Node Pool OCID Here> --size 2
```

Вставьте OCID вашего Node Pool. Замените значение **size** на нужный размер пула.

## [Далее: завершение и очистка](Oracle Kubernetes Workshop p7.md)