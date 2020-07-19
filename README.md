
# Intro
I have been looking for a long time for a solution I could call it my home cloud. The answer was to buy a Synology 918+ NAS with 4 vCPUs and 8 GBs of RAM. It runs Docker images but, as I discovered later on, it's not able to run Kubernetes. On my first try, I created a VM inside the NAS and then installed K8s on it, it didn't take me long to realise that it was taking 1-2 vCPU only to run the master layer.
This didn't leave much room for other containers to run. Not good. On my second try, I gave MicroK8s a go, still the CPU consumption is too high.

The answer came with K3s, it's a fantastic K8s distribution, very lean, and it runs on 0.5 vCPU of constant usage. So far, I haven't had any compatibility issues.

As my first useful deployment, I decided to build a PiHole service to keep my devices free from ads.

# Requirements
I used Helm to deploy this solution, so you will need:
* Some VM / Device using Ubuntu 20.04 in the same network you want to block ads on.
* Kubectl
* Helm 3

# Installing K3s
K3s is a Kubernetes distribution made for Edge by Rancher. Installation instructions are [here](https://rancher.com/docs/k3s/latest/en/installation/).

It comes with Traefik by default, as I am not really familiar with it and I wanted to try [Rancher's MetalLB](https://metallb.universe.tf/), I disabled Traefik along side with the internal *servicelb* component.

For my "home cloud" I installed K3s over Ubuntu 20.04 by simply typing this:

`curl -sfL https://get.k3s.io | sh -s - server --no-deploy traefik --no-deploy servicelb`

It does everything, even creates all the init files for K3s to come up after machine restart.

To see the kube config for this cluster: 

`sudo k3s kubectl config view --raw`

Copy / paste this output into the `~/.kube/config` file in the computer you are going to use to install the rest of the components.

# Disabling Traefik

***Skip this section if you installed K3s with the command above.***

If you, like me, installed K3s using other methods, you will still need to disable Traefik. 

```
# Remove traefik helm chart resource: 
kubectl -n kube-system delete helmcharts.helm.cattle.io traefik

# Stop the k3s service: 
sudo service k3s stop

# Edit service file 
sudo vi /etc/systemd/system/k3s.service

# To disable traefik and servicelb, add the last two lines, it will look something like this:
# ExecStart=/usr/local/bin/k3s \
#    server \
#    --no-deploy traefik \
#    --no-deploy servicelb \

# Then reload the service file: 
sudo systemctl daemon-reload

# Remove the manifest file from auto-deploy folder:
sudo rm /var/lib/rancher/k3s/server/manifests/traefik.yaml

# Start the K3s service: 
sudo service k3s start
```

# Installing MetalLB
Once the K3s cluster is up and running, we proceed with MetalLB installation. This can be done from the same K3s server or from a remote client.
The installation is quite straightforward:

First, creating the configuration, there is an example of the required ConfigMap on this git repository at *./metallb_config.yml* . There, you need to change the last lines where it tells MetalLB the range of **free** IPs that it will use in your local LAN. Once finished, save changes on the file and apply them:

`kubectl apply -f metallb_config.yml`

Now we can proceed with MetalLB installation. You may want to change `v0.9.3` to the latest version available.

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
```


# Installing The PiHole Helm Chart
Once K3s and MetalLB is ready, we are going to proceed to install PiHole. I normally use Helm everywhere I go, I found this Helm chart I liked at https://github.com/MoJo2600/pihole-kubernetes.

Before installing PiHole, we need to deal with Ubuntu's local DNS resolution on the server. It seems to come by default with Ubuntu latest versions but it can be easily disabled by executing:
```
sudo systemctl disable systemd-resolved
sudo systemctl mask systemd-resolved
```

Finally, have a look into the `./pihole_values.yml` file, you may want to customise some configuration for your installation. 

Once you are happy with your configuration, add its Mojo2600 repo to our local repository list:
```
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

And install the Helm chart:

`helm install pihole mojo2600/pihole -f ./pihole_values.yml`

After few seconds, in the list of K3s available resources you should see:
```
$ kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/pihole-b597ff7d4-lz99c   1/1     Running   0          136m

NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                   AGE
service/kubernetes   ClusterIP      10.43.0.1      <none>          443/TCP                                   19d
service/pihole-udp   LoadBalancer   10.43.205.93   192.168.0.120   53:32652/UDP,67:32764/UDP                 11d
service/pihole-tcp   LoadBalancer   10.43.252.64   192.168.0.120   80:31734/TCP,443:32616/TCP,53:32042/TCP   11d
```
That EXTERNAL-IP column should show the first IP on the range you configured in *./metallb_config.yml*. Once the pod is up and running, everything should be ready to serve the first DNS requests.

To ensure that everything works, you can use `nslookup`:
```
$ nslookup www.google.com 192.168.0.120

Server:		192.168.0.120
Address:	192.168.0.120#53

Non-authoritative answer:
Name:	www.google.com
Address: 216.58.204.228
```

That should be it! Enjoy!

# Using PiHole At Home
I am the only user at home that does not enjoy adverts... I know, right?. What I do to avoid complaints from my partner and friends is to disable DHCP on PiHole and configure DNS on all the devices I use day-to-day to point to PiHole MetalLB IP address.
