# Установка Kubernetes v1.31 с помощью kubeadm 
## Требования для VM (взято с офф документации, не стал переводить)
* A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
* 2 GB or more of RAM per machine (any less will leave little room for your apps).
* 2 CPUs or more for control plane machines.
* Full network connectivity between all machines in the cluster (public or private network is fine).
* Unique hostname, MAC address, and product_uuid for every node. See here for more details.
* Certain ports are open on your machines. See here for more details.

## Для всех node (master & worker)
### Для изменения имя хоста
  ```
  sudo hostnamectl set-hostname <hostname>
  ```
### Вносим записть в /etc/hosts 
  ```
  echo "<IP address> <hostname>" |sudo tee -a /etc/hosts 
  ```
### Выключаем swap
  ```
  sudo swapoff -a 
  ```
###  Выключаем swap c nf,kbws 
  ```
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  ```
## Активируют поддержку фильтрации сетевых мостов и включают маршрутизацию IPv4
### Откройте файл /etc/sysctl.conf в текстовом редакторе с правами суперпользователя:
```
sudo nano /etc/sysctl.conf
```
### Добавьте или раскомментируйте следующую строку:
```
net.ipv4.ip_forward = 1
```
Сохраните изменения и закройте редактор.
### Примените изменения с помощью команды:
```
sudo sysctl -p
```

### Обновляем индекс пакетов и устанавливаем зависимости
  ```
  sudo apt-get update
  # apt-transport-https may be a dummy package; if so, you can skip that package
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg software-properties-common
  ```
### Скачайте публичный ключ подписи для репозиториев пакетов Kubernetes. Один и тот же ключ подписи используется для всех репозиториев, поэтому вы можете игнорировать версию в URL.
  ```
  # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
  # sudo mkdir -p -m 755 /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
  ```
### Добавьте соответствующий репозиторий apt для Kubernetes. Обратите внимание, что этот репозиторий содержит пакеты только для Kubernetes 1.31; для других минорных версий Kubernetes вам нужно изменить минорную версию в URL, чтобы она соответствовала желаемой минорной версии (также убедитесь, что вы читаете документацию для версии Kubernetes, которую планируете установить).

  ```
  # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
  ```
### Обновите индекс пакетов apt, установите kubelet, kubeadm и kubectl, и зафиксируйте их версию.
  ```
  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl
  ```

### (По желанию) Включите службу kubelet перед запуском kubeadm:
  ```
  sudo systemctl enable --now kubelet
  ```
## Установка CRI-O
### Добавляем репозиторию и ключи
  ```
  curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/Release.key |
      sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
  
  echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/deb/ /" |
      sudo tee /etc/apt/sources.list.d/cri-o.list
  ```
### Обновляем индекс пакетов и устанавливаем
```
sudo apt-get update && sudo apt-get install -y cri-o 
```

### Включаем 
```
sudo systemctl start crio.service
sudo systemctl enable crio.service
```

## Теперь только на master 
```
sudo kubeadm init
```
копируем токен

## На worker ноде подставляем команду добавляя sudo
```
sudo kubeadm join 172.16.54.55:6443 --token 6o9dgv.rjx52c0b1h92vctv \
        --discovery-token-ca-cert-hash sha256:9c26062aac49a932d5df7c308c4867774309493aa9457e8f1fc6e3d6b7399c8f 
```

### Теперь только на master 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Устанавливаем сетевой драйвер
```
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.31/net.yaml
```
