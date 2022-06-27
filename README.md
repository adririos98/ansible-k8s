# ansible-k8s
Despliegue de k8s mediante ansible en máquinas Ubuntu 22.04 Server
El laboratorio se encuentra formado por 2 nodos.

    1. Ubuntu 22.04 Server, 3GB Ram, 2CPUs (Master, Esclavo)
    2. Ubuntu 22.04 Server, 3GB Ram, 2CPUs (Esclavo)
 

## Instalar ansible en nodo1
    sudo apt update (Ambos nodos)
    sudo apt install ansible (Solo nodo1)

## Generar clave
    ssh-keygen

## Pasar clave a nodo1 y nodo2
    ssh-copy-id root@15.0.0.10
    ssh-copy-id root@15.0.0.11

**NOTA**: En caso de error de login con el usuario root habilitar

	vi /etc/ssh/sshd_config

	# Authentication:
	#LoginGraceTime 2m
	PermitRootLogin yes
	systemctl restart ssh


## Definir fichero de hosts en nodo1
	vi /etc/ansible/hosts

	[servers]
	15.0.0.10
	15.0.0.11
	#cluster1 ansible_hosts=15.0.0.10
	#cluster2 ansible_hosts=15.0.0.11

	[all:vars]
	ansible_python_interpreter=/usr/bin/python3

	#Probar conectividad
	ansible all -m ping -u root


## 1. Instalación K8S (kubeadmin con ansible)
```bash
mkdir ~/kube-cluster
cd ~/kube-cluster
```

### Crear fichero inventario host k8s
```bash
vi ~/kube-cluster/hosts
```

	[masters]
	15.0.0.10 ansible_user=root

	[workers]
	15.0.0.10 ansible_user=root
	15.0.0.11 ansible_user=root

	[all:vars]
	ansible_python_interpreter=/usr/bin/python3

### Crear usuario no root (ubuntu)
```bash
vi ~/kube-cluster/initial.yml
```
```yml
- hosts: all
  become: yes
  tasks:
    - name: Crear el usuario 'ubuntu'
      user: name=ubuntu append=yes state=present createhome=yes shell=/bin/bash

    - name: Permitir que el usuario de ubuntu ejecute comandos sudo sin un mensaje de contraseña.
      lineinfile:
        dest: /etc/sudoers
        line: 'ubuntu ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: Añade la clave pública de su máquina local
      authorized_key: user=ubuntu key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
```

### Crear el usuario en ambos nodos:
```bash
ansible-playbook -i hosts initial.yml
```


## 2. Preparar dependencias de k8s.

```bash
apt-get install kubernetes-cni=0.7.5-00 (ambos nodos)
vi ~/kube-cluster/kube-dependencies.yml
```
```yml
- hosts: all
  become: yes
  tasks:
   - name: instalacion Docker
     apt:
       name: docker.io
       state: present
       update_cache: true

   - name: instalacion APT Transport HTTPS
     apt:
       name: apt-transport-https
       state: present

   - name: Añadir Kubernetes apt-key
     apt_key:
       url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
       state: present

   - name: Añadir Kubernetes' APT repository
     apt_repository:
      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
      state: present
      filename: 'kubernetes'

   - name: install kubelet
     apt:
       name: kubelet=1.14.0-00
       state: present
       update_cache: true

   - name: install kubeadm
     apt:
       name: kubeadm=1.14.0-00
       state: present

- hosts: masters
  become: yes
  tasks:
   - name: install kubectl
     apt:
       name: kubectl=1.14.0-00
       state: present
       force: yes
```

# Ejecutar el playbook anterior
    ansible-playbook -i hosts kube-dependencies.yml

## 3. Configurar nodo master.
### Generar config master
```bash
vi ~/kube-cluster/master.yml
```

```yml
- hosts: masters
  become: yes
  tasks:
    - name: Inicializa el cluster de k8s
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=15.0.0.10 --ignore-preflight-errors=SystemVerification

    - name: crea el directorio .kube
      become: yes
      become_user: ubuntu
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copia admin.conf al usuario de .kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ubuntu/.kube/config
        remote_src: yes
        owner: ubuntu

    - name: variable root kubeconfig
      shell: export KUBECONFIG=/etc/kubernetes/admin.conf

    - name: instalar red de los Pod
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

## Ejecutar el playbook anterior
    ansible-playbook -i hosts master.yml



## 4. Agregar los workers:
```bash
vi ansible-playbook -i hosts ~/kube-cluster/workers.yml
```
```yml
- hosts: masters
  become: yes
  gather_facts: false
  tasks:
    - name: Obtener comando de token
      shell: kubeadm token create --print-join-command
      register: join_command_raw

    - name: Guardar comando token en variable join_command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[0] }} "


- hosts: workers
  become: yes
  tasks:
    - name: Añadir a cluster
      shell: "{{ hostvars['masters'].join_command }}"
```
## Ejecutar el playbook anterior
    ansible-playbook -i hosts workers.yml


# Tareas opcionales
### Vaciar la swap y habilitarla para la instalación.
sudo swapoff -a && sudo sed -i '/swap/d' /etc/fstab

### Ampliar cpu en nodo master a 2 como mínimo.
### Modificar demonio docker:
``` bash
vi /usr/lib/systemd/system/docker.service
```

(Modificar la siguiente linea y añadir **--exec-opt native.cgroupdriver=systemd**)
    
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Debe quedar así:

	ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd

```bash
systemctl daemon-reload
systemctl restart docker
```