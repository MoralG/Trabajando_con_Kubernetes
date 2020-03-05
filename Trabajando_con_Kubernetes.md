
### Escenario

![Escenario](image/Escenario.png)

### Puertos necesarios para acceder al cluster de Kubernetes

Tenemos que abrir los puertos necesarios para la configuración de Kubernetes. En nuestro caso, al trabajar en el cloud de OpenStack tenemos que abrir los siguientes puertos:

* 80: Para acceder a los servicios con el controlador Ingress.
* 443: Para acceder a los servicios con el controlador Ingress y HTTPS.
* 6443: Para acceder a la API de Kubernetes.
* 30000-40000: Para acceder a las aplicaciones con el servicio NodePort.

Ya tenemos el escenario listo para realizar la tarea, mostraremos donde hay que realizar los puntos siguiente indicandolo entre paréntesis a las máquinas que se le aplican.

### Instalación de Docker (Master, Nodo1, Nodo2)

Kubernetes trabaja por encima de docker, por lo que tenemos que instalar docker en la máquina **master** y en los **nodos**.

Instalamos los paquetes que nos permiten usar repositorios apt con https:

~~~
sudo apt update

sudo apt install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
~~~

Añadimos las claves GPG oficiales de Docker:

~~~
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
~~~

Añadimos el repositorio para nuestra versión de Debian:

~~~
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/debian \
   $(lsb_release -cs) \
   stable"
~~~

Y ahora ya podemos instalar docker:

~~~
sudo apt update
sudo apt install docker-ce
~~~

Comprobamos la versión de docker instalada:

~~~
docker --version
    Docker version 19.03.7, build 7141c199a2
~~~

### Instalación de kubeadm, kubelet and kubectl (Master, Nodo1, Nodo2)

Vamos a instalar los paquetes necesarios para una instalación y configuración funcional de Kubernetes:

* **kubeadm**: Herramienta que nos permite crear el cluster.
* **kubelet**: Es el componente que se ejecuta en el master y en todos los nodos, el cual se encarga de ejecutar los pods y los contenedores.
* **kubectl**: Herramienta que nos permite controlar el cluster desde la linea de comandos.

La instalación de estos paquetes se realiza de la siguiente manera:

~~~
sudo su
apt update && apt install -y apt-transport-https curl
~~~

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
~~~

~~~
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
~~~
~~~
apt update
apt install -y kubelet kubeadm kubectl
~~~

### Configuración de Kubeadm (Master)

Para inicializar el cluster vamos a ejecutar el siguiente comando:

> **NOTA**: Es posible que tengamos que desastivar el swap `swapoff -a`, esto nos lo indica en la documentación.

~~~
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-cert-extra-sans=172.22.200.129
~~~

>`--pod-network-cidr`: La red por donde se comunican los nodos del cluster.
>
>`--apiserver-cert-extra-sans`: La IP flotante del master para que el certificado que se genera sea válido para esta ip, y se pueda controlar el cluster desde el exterior.

Al terminar nos muestra lo siguiente:

~~~
.
.
.
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.12:6443 --token ijp180.u4s50rzes1ymbipn \
    --discovery-token-ca-cert-hash sha256:6c59402543d65e4bf68865ded7460e8f4af334bc4f397a5736a02b4a7f705834 
~~~

Lo que nos devuelve nos indica 3 pasos a realizar:

1. Las instrucciones que tenemos que ejecutar en la máquina **master** para usar el cliente kubectl y manejar el cluster. Este paso lo realizaremos a continuación:

~~~
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

2. La instalación de un pod para la gestión de la red, que se explicará en el siguiente [punto](https://github.com/MoralG/Trabajando_con_Kubernetes/blob/master/Trabajando_con_Kubernetes.md#configuraci%C3%B3n-del-pod-para-gestionar-la-red-master).

3. Y la instrucción que tenemos que ejecutar en los nodos para añadirlos al cluster. Utilizaremos un token para ello, que se explicará también en los siguientes [puntos](https://github.com/MoralG/Trabajando_con_Kubernetes/blob/master/Trabajando_con_Kubernetes.md#uniendo-los-nodos-al-cluster-nodo1-nodo2).


### Configuración del pod para gestionar la red (Master)

Antes de ello, en el master con un usuario con privilegios podemos usar el cliente kubectl ejecutando:

~~~
export KUBECONFIG=/etc/kubernetes/admin.conf
~~~

Tenemos que instalar un pod para permitir la comunicación de los distintos pods que vamos a iniciar en el cluster, esto lo realizaremos con el comando que nos proporciono la ejecución del cluster en la anterior tarea. Vamos a utilizar la herramienta calico, esta la instalaremos de la siguiente manera:

~~~
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
~~~

Y comprobamos que los pods estan funcionando correctamente:

~~~
kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-77c4b7448-gpqvt   1/1     Running   0          3m31s
calico-node-bz5dw                         1/1     Running   0          3m32s
coredns-6955765f44-7k8xt                  1/1     Running   0          7m46s
coredns-6955765f44-rwzkz                  1/1     Running   0          7m46s
etcd-master                               1/1     Running   0          8m13s
kube-apiserver-master                     1/1     Running   0          8m13s
kube-controller-manager-master            1/1     Running   1          8m13s
kube-proxy-l2z42                          1/1     Running   0          7m47s
kube-scheduler-master                     1/1     Running   1          8m13s
~~~

> Con la opción `-n kube-system` le indicamos el espacio de nombre.

### Uniendo los nodos al cluster (Nodo1, Nodo2)

Para unir los nodos al cluster vamos a ejecutar el comando que nos indicó anteriormente al iniciar el cluster con `kubeadm`. 

~~~
kubeadm join 10.0.0.12:6443 \
--token ijp180.u4s50rzes1ymbipn \
--discovery-token-ca-cert-hash sha256:6c59402543d65e4bf68865ded7460e8f4af334bc4f397a5736a02b4a7f705834 
~~~

Nos devuelve lo siguiente:
~~~
  W0304 19:23:25.055961   13343 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
  [preflight] Running pre-flight checks
	  [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
  [preflight] Reading configuration from the cluster...
  [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
  [kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
  [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
  [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
  [kubelet-start] Starting the kubelet
  [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
~~~

### Comprobación de la unión de los nodos (Master)

Y finalmente desde el master podemos obtener los nodos que forman el cluster:

~~~
kubectl get nodes
    NAME      STATUS    ROLES     AGE       VERSION
    k8s-1     Ready     master    1d        v1.10.2
    k8s-2     Ready     <none>    1d        v1.10.2
    k8s-3     Ready     <none>    1d        v1.10.2
~~~

Y si volvemos a realizar el comando `kubectl get pods -n kube-system`, podemos ver que se han añadidos nuevos pods referente a los nodos que hemos unido.

~~~
kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-77c4b7448-gpqvt   1/1     Running   0          20m
calico-node-44qkz                         1/1     Running   0          3m9s
calico-node-bz5dw                         1/1     Running   0          20m
calico-node-gkl75                         1/1     Running   0          3m19s
coredns-6955765f44-7k8xt                  1/1     Running   0          24m
coredns-6955765f44-rwzkz                  1/1     Running   0          24m
etcd-master                               1/1     Running   0          25m
kube-apiserver-master                     1/1     Running   0          25m
kube-controller-manager-master            1/1     Running   1          25m
kube-proxy-jz4td                          1/1     Running   0          3m9s
kube-proxy-l2z42                          1/1     Running   0          24m
kube-proxy-mnqq8                          1/1     Running   0          3m19s
kube-scheduler-master                     1/1     Running   1          25m
~~~

Ya tenemos Kubernetes instalado y funcioanl, ahora podemos crear, como se muestra en el siguiente punto, un cliente y configurarlo para controlar al master, si no qeremos realizar dicho punto podemos saltarlo y realizar el despliegue de una aplicación con Kubernetes.

### Acceso desde un cliente externo (Cliente1)

Normalmente vamos a interactuar con el cluster desde un cliente externo donde tengamos instalado kubectl. Para instalar kubectl, siguiendo las instrucciones oficiales, ejecutamos:

~~~
apt update && apt install -y apt-transport-https
~~~

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
~~~

~~~
apt update
apt install -y kubectl
~~~

Y para configurar el acceso al cluster:

Desde el nodo master damos permisos de lectura al fichero /etc/kubernetes/admin.conf:

~~~
chmod 644 /etc/kubernetes/admin.conf
~~~

Desde el cliente:

~~~
export IP_MASTER=172.22.200.129
sftp debian@${IP_MASTER}
sftp> get /etc/kubernetes/admin.conf
sftp> exit
    
mv admin.conf ~/.kube/mycluster.conf
sed -i -e "s#server: https://.*:6443#server: https://${IP_MASTER}:6443#g" ~/.kube/mycluster.conf
export KUBECONFIG=~/.kube/mycluster.conf
~~~

Y comprobamos que tenemos acceso al cluster:

~~~
kubectl cluster-info
    Kubernetes master is running at https://172.22.200.129:6443
~~~

### Desplique de una aplicación ()