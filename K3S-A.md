# 火種雲


---
## 火種雲介紹與架構圖
:wave: **Kubernetes 火種雲是為了幫助企業能夠快速部屬各式各樣的應用程式與服務，且透過 Jenkins 做到持續整合和部屬、Gitea 進行系統程式的版本控管、quay 為私人儲存庫用來存取 container 所需的 image 檔。**  

:arrow_down_small: 火種雲架構圖  
![](https://i.imgur.com/aCv1Ex9.png)


---

## 火種雲運作示意圖
:arrow_right: 火種雲能夠快速部屬客戶所需的 k3s cluster，以便企業能在 k3s 叢集裡面運作企業的應用系統與服務。   
![](https://i.imgur.com/nl1bYLz.png)

---
## 火種雲部屬 k3s Pattern A cluster 操作範例  
:arrow_right: 在火種雲的 master 執行  
\$ ./git-push.sh  
  
:arrow_down_small: 系統運作流程圖   
![](https://i.imgur.com/jVjqj4Y.png)

:arrow_down_small: Jenkins 網站上看到的畫面  
![](https://i.imgur.com/1j0Z38R.png)

:arrow_down_small: k3s patter A 架構圖  
![](https://i.imgur.com/cJsi086.png)

---
## 火種雲運行時所需的程式  
### 1. git-push.sh
```
#!/bin/bash
cd wk/K3S-A
git add www
git add atoken.sh
git add check.sh
git add k3s-a.sh
git add k3s-ap.sh
git add k3s-ssh.sh
git add nginx.yaml
git add Jenkinsfile
git commit -m "create k3s pattern A"
git push
```
:arrow_right: 目的 : 將火種雲運作時所需的程式 push 至 gitea 上面，之後 Jenkins 建立的 pipeline 專案偵測到變動後就會開始進行部屬。  


---

### 2. Jenkinsfile
```
pipeline {
    agent { label 'adm' }
    environment {
        am1 = '192.168.25.51'
        aw1 = '192.168.25.52'
        aw2 = '192.168.25.53'
    }
    stages {
        stage('check node') {
            steps {
                sh "scp check.sh ${am1}:~/"
                sh "scp check.sh ${aw1}:~/"
                sh "scp check.sh ${aw2}:~/"
                sh "ssh ${am1} source check.sh"
            }
        }
        stage('k3s-master') {
            steps {
                sh "scp k3s-ap.sh ${am1}:~/"
                sh "scp k3s-a.sh ${am1}:~/"
                sh "scp k3s-ssh.sh ${am1}:~/"
                sh "ssh ${am1} source k3s-ssh.sh"
                sh "ssh ${am1} source k3s-ap.sh"
                sh "sleep 30"
                sh "ssh ${am1} source k3s-a.sh"
            }
        }
        stage('k3s-worker') {
            steps {
                sh "scp atoken.sh ${aw1}:~/"
                sh "scp atoken.sh ${aw2}:~/"
                sh "ssh ${aw1} source check.sh"
                sh "ssh ${aw1} source atoken.sh"
                sh "ssh ${aw2} source check.sh"
                sh "ssh ${aw2} source atoken.sh"
            }
        }
        stage('quay') {
            steps {
                sh "scp usequay.sh ${am1}:~/"
                sh "scp usequay.sh ${aw1}:~/"
                sh "scp usequay.sh ${aw2}:~/"
                sh "ssh ${am1} source usequay.sh"
                sh "ssh ${aw1} source usequay.sh"
                sh "ssh ${aw2} source usequay.sh"
                sh "sleep 30"
            }
        }
        stage('clear') {
            steps {
                sh "ssh ${am1} rm check.sh k3s-ap.sh k3s-a.sh k3s-ssh.sh usequay.sh"
                sh "ssh ${aw1} rm check.sh atoken.sh usequay.sh"
                sh "ssh ${aw2} rm check.sh atoken.sh usequay.sh"
            }
        }
        stage('test k3s') {
            steps {
                sh "scp nginx.yaml ${am1}:~/"
                sh "scp -r www ${am1}:~/"
                sh "ssh ${am1} kubectl get nodes"
                sh "ssh ${am1} kubectl apply -f nginx.yaml"
            }
        }
    }
}

```
:arrow_right: 目的 : 宣告 Jenkins 要執行的程式內容，因為是透過 SSH 方式至 k3s pattern A 的機器進行部屬，所以在 adm (Jenkins agent) 要先傳送公鑰至要部屬的機器上，做到自由進行即可。  

---

### 3. check.sh
```
#!/bin/bash
declare -i ram1 a b c sum
ap=$(cat /etc/issue | grep 'Alpine' | cut -d ' ' -f 3)
[ $ap = "Alpine" ] && a=1 || a=0
ram=$(free -mh | grep "Mem" | tr -s " " | cut -d ' ' -f 4)
ram1=${ram//[!0-9]/} 
[ $ram1 -ge 512 ] && b=1 || b=0
ping -c 3 8.8.8.8 &>/dev/null
[ $? = 0 ] && c=1 || c=0
echo $a
echo $b
echo $c
sum=$((a+b+c))
[ $sum != 3 ] && sudo reboot || echo "it's ok"
```
:arrow_right: 目的 : 在部屬 k3s-a 之前，確認 node (m1,w1,w2) 的狀態，檢查 3 個點，作業系統是否為 alpine、記憶體可用容量是否大於 512 mb 以及在部屬時的網路狀態。  


---
### 4. k3s-ap.sh  
```
#!/bin/bash
which kubectl &>/dev/null
if [ $? != 0 ]
then
sudo sed -i 's|pax_nouderef quiet rootfstype=ext4|pax_nouderef quiet rootfstype=ext4 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory|g' /etc/update-extlinux.conf
sudo update-extlinux
sudo reboot
else
echo 'k3s is exist'
fi
```
:arrow_right: 目的 : 建置 k3s 時，設定使用存取 metadata 的資料庫為 etcd。

---
### 5. k3s-a.sh  
```
#!/bin/bash
which kubectl &>/dev/null
if [ $? != 0 ]
then
curl -sfL https://get.k3s.io | K3S_TOKEN="mysecret" K3S_KUBECONFIG_MODE="644" sh -s server --cluster-init --cluster-domain=sre
else
echo 'k3s is exist'
fi
```
:arrow_right: 目的 : 部屬 k3s pattern A 的 master。 

---

### 6. k3s-ssh.sh  
```
#!/bin/bash
sudo sed -i 's|#PermitUserEnvironment no|PermitUserEnvironment yes|g' /etc/ssh/sshd_config
echo -e "HOME=/home/bigred\nPATH=/bin:/usr/bin:/sbin:/usr/sbin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" > ~/.ssh/environment
```
:arrow_right: 目的 : 設定 k3s pattern A 的 master 能夠利用 ssh 連線方式進行對於此叢集進行 kubectl 的指令。  

---

### 7. atoken.sh  
```
#!/bin/bash
curl -sfL https://get.k3s.io | K3S_TOKEN="mysecret" K3S_URL="https://192.168.25.51:6443" K3S_KUBECONFIG_MODE="644" sh -s -
```
:arrow_right: 目的 : 部屬 k3s pattern A 的 workers ，192.168.25.51 為此範例 k3s pattern A master 的 IP。  

---
### 8. usequay.sh  
```
#!/bin/bash
# change registries
cat << "EOF" | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "192.168.25.41:8088":
    endpoint:
      - "http://192.168.25.41:8088"
EOF

sudo reboot
```
:arrow_right: 目的 : 設定告訴 k3s pattern A 的 nodes 我們私人儲存庫(quay)的位置。  

---
### 9. nginx.yaml
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx
  labels :
    run : nginx 
spec:
  volumes:
    - name: pv-www
      hostPath:
        path: /home/bigred/www
  containers:
    - name: nginx
      image: 192.168.25.41:8088/quay/nginx
      imagePullPolicy: Always
      ports:
      - containerPort: 80
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-www
  nodeSelector:  
    kubernetes.io/hostname : am1
---
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
spec:
  externalIPs:
  - 192.168.25.51
  selector:
    run: nginx
  ports:
  - port: 8080
    targetPort: 80
```
:arrow_right: 目的 : 為了測試 k3s pattern A 是否能正常運作，於是 k3s pattern A cluster 裡起了一個 pod 架設網站，使用的是火種雲私人儲存庫(quay)的 image ，以及 創造一格svc 開啟 external IP 對外連接。  