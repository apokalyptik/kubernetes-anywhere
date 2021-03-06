# Running Kubernetes on Docker for Mac

> **Q:** What's good about this approach?
>
> **A:** You can actually extend the cluster pretty easily, i.e. start single node on your laptop, add colleague's laptop as another node or even relocate your cluster to the cloud :)


## Boostrap

First, setup Weave Net:
```
sudo curl --silent --location https://git.io/weave --output /usr/local/bin/weave
sudo chmod +x /usr/local/bin/weave
weave launch
weave expose -h moby.weave.local
```

Next, bootstrap single-node local cluster with one command:
```
docker run --rm \
  --volume="/:/rootfs" \
  --volume="/var/run/weave/weave.sock:/docker.sock" \
    weaveworks/kubernetes-anywhere:toolbox-v1.2 \
      sh -c 'setup-single-node && compose -p kube up -d'
```

## Setup `kubectl`

Install `kubectl` command via Homebrew:
```
brew install kubectl
```

If you have installed it by some other means earlier, you should check if the version you have is at least v1.2.0,
otherwise some features may not work as expected.

> If you have installed Google Cloud SDK (`gcloud`), you will have `kubectl`, just make sure to run `gcloud components update`.

To confirm all is working correctly, run `kubectl cluster-info && kubectl get nodes` and you should see one node called `moby` on the list, i.e.:

```
> kubectl cluster-info && kubectl get nodes
Kubernetes master is running at http://localhost:8080
NAME      STATUS    AGE
moby      Ready     5m
```

If the output you see is completely different, you might be talking to (or trying to talk to) another cluster. To check,
please try to specify explicit address like this:

```
> kubectl -s http://localhost:8080 cluster-info && kubectl -s http://localhost:8080 get nodes
Kubernetes master is running at http://localhost:8080
NAME      STATUS    AGE
moby      Ready     5m
```

If you would like to avoid passing `-s` flag each time, you can overwrite default configuration file like this:
```
docker run --rm --volumes-from=kube-toolbox-pki weaveworks/kubernetes-anywhere:toolbox-v1.2 print-local-config > ~/.kube/config
```

> If you don't wish to install anything, you can use provided toolbox container like this:
> ```
> docker run --rm \
>   --net=weave --dns=172.17.0.1 \
>   --volumes-from=kube-toolbox-pki \
>   weaveworks/kubernetes-anywhere:toolbox-v1.2 \
>     kubectl [flags] [command]
> ```

## Create Cluster Addons

The cluster is empty right now. Most example apps reply on SkyDNS addon being present. You can setup all neccessary
addons (currently only SkyDNS) with one command:
```
kubectl create -f https://git.io/vw5Uc
```

## Deploy the Pixel Monsterz App

> **Q:** What's this app?
>
> **A:** [The Pixel Monsterz App](https://github.com/ThePixelMonsterzApp) is a fairly simple one and uses cool frameworks,
> Python/Flask and Node.js/Restify, as well as Redis. You might have seen an earlier version of this app, if you have read
> Adrian Mouat’s book “Using Docker”. This app was adapted from MonsterID, which generates unique avatars for signed up users
> (as seen on Github).

```
kubectl create -f https://raw.github.com/ThePixelMonsterzApp/infra/master/kubernetes-app.yaml
```

You can run `kubectl get pods --watch` to see pods being created. Once all pods are created, you should see output
similar to this:

```
> kubectl get pods
NAME                 READY     STATUS    RESTARTS   AGE
hello-d53og          1/1       Running   0          1m
hello-wis7n          1/1       Running   0          1m
monsterz-den-hu0af   1/1       Running   0          1m
monsterz-den-q4sp6   1/1       Running   0          1m
monsterz-den-t1wxe   1/1       Running   0          1m
redis-94stx          1/1       Running   0          1m
```

Now, you can load the app in your browser using proxied URL:
```
http://localhost:8001/api/v1/proxy/namespaces/default/services/hello:app/
```

If you reload the page a few times, you will see counter being incremented and different mosnters will show each time. The
monster shown at the very top belongs to a pod that serves it (`hello-*`), the other monster is pseudo-random. You can get a great variety of monsters by creating more
pods, i.e. scaling the replication controller.

```
kubectl scale rc hello --replicas=12
```

## Additional notes and TODOs

- [ ] How to expand the cluster?
- [ ] <strike>Explain how to use [`weave-osx-ctl`](https://github.com/pidster/weave-osx-ctl/)...</strike> _(broken as of beta9, thanks Apple)_

