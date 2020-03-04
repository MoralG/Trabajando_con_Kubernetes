
### Escenario

![Escenario](image/Escenario.png)

### Puertos necesarios para acceder al cluster de Kubernetes
Si instalamos el cluster en instancias de un servicio cloud de infraestuctura hay que tener en cuanta que los siguientes puertos deben estar accesibles:

* 80: Para acceder a los servicios con el controlador Ingress.
* 443: Para acceder a los servicios con el controlador Ingress y HTTPS.
* 6443: Para acceder a la API de Kubernetes.
* 30000-40000: Para acceder a las aplicaciones con el servicio NodePort.

### Instalación de Docker (Master, Nodo1, Nodo2)

Instalación de DockerPermalink
Lo primero que hacemos, siguiendo las instrucciones de instalación de página oficial, es instalar la última versión de docker, en los tres nodos:
~~~
sudo apt update
~~~

Instalamos los paquetes que nos permiten usar repositorios apt con https:

~~~
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

Y por último instalamos docker:

~~~
sudo apt-get update
sudo apt-get install docker-ce
~~~

Finalmente comprobamos la versión instalada:

~~~
docker --version
    Docker version 18.03.1-ce, build 9ee9f40
~~~

### Instalación de kubeadm, kubelet and kubectl (Master, Nodo1, Nodo2)

Vamos a instalar los siguientes paquetes en nuestras máquinas:

* kubeadm: Instrucción que nos permite crear el cluster.
* kubelet: Es el componente de kubernetes que se ejecuta en todos los nodos y es responsable de ejecutar los pods y los contenedores.
* kubectl: La utilidad de línea de comandos que nos permite controlar el cluster.

Para la instalación, seguimos los pasos indicados en la documentación:

~~~
sudo su
apt-get update && apt-get install -y apt-transport-https curl
~~~

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
~~~
~~~
apt-get update
apt-get install -y kubelet kubeadm kubectl
~~~

### Configuración de Kubeadm (Master)

En el nodo que vamos a usar como master, ejecutamos la siguiente instrucción como superusuario:

~~~
swapoff -a
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-cert-extra-sans=172.22.201.15
~~~

Este comando inicializa el cluster, hemos indicado le CIDR de la red por donde se comunican los nodos del cluster.

> Estoy utilizando como infraestructura tres instancias de OpenStack, es necesario indicar el parámetro `--apiserver-cert-extra-sans` con la IP flotante del master para que el certificado que se genera sea válido para esta ip, y se pueda controlar el cluster desde el exterior.

Cuando termina muestra un mensaje similar a este:

~~~
.
.
.
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
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

kubeadm join 10.0.2.15:6443 --token xdpy6d.i8fwbzhj9opwtxwh \
    --discovery-token-ca-cert-hash sha256:6bf63347b5dc6deed279ae152d979d47b5b07bfa2e1a42e3a6ad7de3d74fead9 

~~~

Nos indica tres cosas:

1. Las instrucciones que tenemos que ejecutar en el master, con un usuario sin privilegios para usar el cliente kubectl y manejar el cluster.
2. La necesidad de instalar un pod para la gestión de la red.
3. Y la instrucción que tenemos que ejecutar en los nodos para añadirlos al cluster. Utilizaremos un token para ello.

### Configuración del pod para gestionar la red (Master)

Antes de ello, en el master con un usuario con privilegios podemos usar el cliente kubectl ejecutando:

~~~
export KUBECONFIG=/etc/kubernetes/admin.conf
~~~

A continuación tenemos que instalar un pod que nos permita la comunicación por red de los distintos pods que vamos a correr en el cluster. `kubeadm` solo soporta plugins de red CNI (Container Network Interface), que es un proyecto que consiste en crear especificaciones y librerías para configure las redes que interconectan los contenedores. De las distintas alternativas vamos a instalar `Calico`, para ello:

~~~
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
~~~

Y comprobamos que todos los pods del espacio de nombres kube-system están funcionando con normalidad:

~~~
kubectl get pods -n kube-system
    NAME                                       READY        STATUS    RESTARTS   AGE
    calico-etcd-kxfp2                          1/1          Running   0          1d
    calico-kube-controllers-5d74847676-fb9nl   1/1          Running   0          1d
    calico-node-c8pjr                          2/2          Running   0          1d
    calico-node-g8xpt                          2/2          Running   0          1d
    calico-node-q5ls4                          2/2          Running   0          1d
    etcd-k8s-1                                 1/1          Running   0          1d
    kube-apiserver-k8s-1                       1/1          Running   0          1d
    kube-controller-manager-k8s-1              1/1          Running   0          1d
    kube-dns-86f4d74b45-828lm                  3/3          Running   0          1d
    kube-proxy-9jmdn                           1/1          Running   0          1d
    kube-proxy-pjr2b                           1/1          Running   0          1d
    kube-proxy-xqsmg                           1/1          Running   0          1d
    kube-scheduler-k8s-1                       1/1          Running   0          1d
~~~

#### Uniendo los nodos al cluster (Nodo1, Nodo2)

En cada nodo que va a formar parte del cluster tenemos que ejecutar, como superusuario, el comando que nos ofreció el comando kubeadm al iniciar el cluster en el master:

~~~
kubeadm join --token <token> <master-ip>:<master-port>
...
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.    

Run 'kubectl get nodes' on the master to see this machine join.
~~~

Y finalmente desde el master podemos obtener los nodos que forman el cluster:

~~~
kubectl get nodes
    NAME      STATUS    ROLES     AGE       VERSION
    k8s-1     Ready     master    1d        v1.10.2
    k8s-2     Ready     <none>    1d        v1.10.2
    k8s-3     Ready     <none>    1d        v1.10.2
~~~

### Acceso desde un cliente externo ()

Normalmente vamos a interactuar con el cluster desde un cliente externo donde tengamos instalado kubectl. Para instalar kubectl, siguiendo las instrucciones oficiales, ejecutamos:

~~~
apt-get update && apt-get install -y apt-transport-https
~~~

~~~
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
~~~

~~~
apt-get update
apt-get install -y kubectl
~~~

Y para configurar el acceso al cluster:

Desde el nodo master damos permisos de lectura al fichero /etc/kubernetes/admin.conf:

~~~
chmod 644 /etc/kubernetes/admin.conf
~~~

Desde el cliente:

~~~
export IP_MASTER=172.22.201.15
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
    Kubernetes master is running at https://172.22.201.15:6443
~~~
