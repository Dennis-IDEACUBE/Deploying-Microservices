### Google Drive Path ###
https://drive.google.com/drive/folders/1drUGDcoWGTehvUSHhJnXkOF9pKRbD3NM?usp=drive_link

### Dockerfile로 배포하기(Dockerfile)
    - myapp
    mvn clean package    
    ### Dockerfile #######################
    FROM openjdk:17-slim
    ARG JAR_FILE=target/*.jar
    COPY ${JAR_FILE} app.jar
    ENTRYPOINT ["java","-jar","./app.jar"]
    ######################################
    docker build -t myapp:0.1 .
    docker run --name myapp -d -p 8080:8080 myapp:0.1
    * 접속 : http://localhost:8080/hello

    - catalog-service & database(mysql)
    # docker run으로 mysql container 생성하기
    docker network create -d bridge mynet
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog_dev --net mynet -d -p 3306:3306 mysql:latest

    => ./gradlew clean bootBuildImage (Buildpack을 사용: Permission denied경우, chmod 755 ./gradlew)
       mvn spring-boot:build-image(Maven 프로젝트인경우)
    
    # docker run으로 배포하기
    docker network create -d bridge mynet(이전에 만든 네트워크가 있으면 실행할 필요가 없음)
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog --net mynet -d -p 3306:3306 mysql:latest
    docker run --name catalog-service -d -p 9001:9001 -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/polardb_catalog  --net mynet catalog-service:0.0.1-SNAPSHOT
    * 접속 : http://localhost:9001/books

### Docker Compose로 배포하기(docker-compose.yml)
    - catalog-service & database(mysql)
    ### docker-compose.yml #########################
    services:
    
      # Applications
      catalog-service:
        depends_on:
          - polar-mysql
        image: "catalog-service:0.0.1-SNAPSHOT"
        container_name: "catalog-service"
        restart: always
        ports:
          - 9001:9001
        environment:
          - SPRING_DATASOURCE_URL=jdbc:mysql://polar-mysql:3306/polardb_catalog
          - SPRING_DATASOURCE_USERNAME=user1
          - SPRING_DATASOURCE_PASSWORD=1234
      
      # Backing Services
      polar-mysql:
        image: "mysql:latest"
        container_name: "polar-mysql"
        ports:
          - 3306:3306
        environment:
          - MYSQL_ROOT_PASSWORD=1234
          - MYSQL_USER=user1
          - MYSQL_PASSWORD=1234
          - MYSQL_DATABASE=polardb_catalog
    ################################################
    docker compose up -d
    docker compose down

### Docker Private Registry 구축하기

    docker pull registry
    docker run -itd --name local-registry -p 5000:5000 registry
    
    /etc/init.d/docker에 DOCKER_OPTS=--insecure-registry localhost:5000 추가
    
    /etc/docker/daemon.json 생성후
    {
        "insecure-registries": ["localhost:5000"]
    }
    sudo systemctl restart docker 서비스 재시작
    
    docker tag catalog-service:0.0.1-SNAPSHOT localhost:5000/catalog-service:0.0.1-SNAPSHOT
    docker push localhost:5000/catalog-service:0.0.1-SNAPSHOT
    docker images localhost:5000/catalog-service 검색하기
    ocker rmi localhost:5000/catalog-service:0.0.1-SNAPSHOT 삭제하기
    
    docker run --name mysql-db -e MYSQL_ROOT_PASSWORD=1234 -e MYSQL_USER=user1 -e MYSQL_PASSWORD=1234 -e MYSQL_DATABASE=polardb_catalog --net mynet -d -p 3306:3306 mysql:latest
    docker run --name catalog-service -d -p 9001:9001 -e SPRING_DATASOURCE_URL=jdbc:mysql://mysql-db:3306/polardb_catalog  --net mynet localhost:5000/catalog-service:0.0.1-SNAPSHOT

### Kubernetes Cluster 구성하기 ###
    ## Master Node & Worker Node
    ##############################################################################################
    sudo nano /etc/cloud/cloud.cfg.d/90-installer-network.cfg # 각각의 ip address 설정
    sudo su
    printf "\n10.0.2.4 myserver01\n10.0.2.5 myserver02\n10.0.2.6 myserver03\n\n" >> /etc/hosts
    exit
    * swap 및 hostname 설정
    sudo apt-get update
    sudo swapoff -a
    sudo nano /etc/fstab
    #Comment out (add a # at the beginning of) the line(s) that reference swap. For example
    # /swapfile none swap sw 0 0
    
    sudo hostnamectl set-hostname "myserver01"
    sudo init 6
    
    
    * modules 설정
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    
    sudo modprobe br_netfilter
    sudo modprobe overlay
    
    * Networking 설정
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    
    sudo sysctl --system
    
    * Containerd 설정
    sudo apt-get update
    sudo apt-get install -y containerd
    
    * Containerd configuration 설정
    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    cat /etc/containerd/config.toml
    
    sudo systemctl restart containerd.service
    
    sudo systemctl status containerd
    
    * Kubernetes Management Tools 설치
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl
    sudo apt-get install -y apt-transport-https ca-certificates curl
    
    
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl


    ## Master Node Only
    ##############################################################################################
    sudo kubeadm init --apiserver-advertise-address=10.0.2.4 --pod-network-cidr=192.168.0.0/16 --cri-socket /run/containerd/containerd.sock --ignore-preflight-errors Swap

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
    
    kubeadm token create --print-join-command
    

    ## Worker Node Only
    ##############################################################################################
    sudo kubeadm join 192.168.10.2:6443 --token 위에 print-join-command 출력 토큰을 입력

    kubectl get pod -A
    kubectl get node -o wide

    kubectl run nginx-deployment --image=nginx --port=80
    kubectl expose deployment nginx-deployment --type=LoadBalancer --name=nginx-service
    kubectl get pods
    kubectl get services
    
### Kubernetes에 배포하기(deployment.yml, service.yml)

    ### deployment.yml #############################
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: catalog-service
      labels:
        app: catalog-service
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: catalog-service
      template:
        metadata:
          labels:
            app: catalog-service
        spec:
          containers:
            - name: catalog-service
              image: catalog-service
              imagePullPolicy: IfNotPresent
              lifecycle:
                preStop:
                  exec:
                    command: [ "sh", "-c", "sleep 5" ]
              ports:
                - containerPort: 9001
              env:
                - name: BPL_JVM_THREAD_COUNT
                  value: "50"
                - name: SPRING_DATASOURCE_URL
                  value: jdbc:mysql://polar-postgres/polardb_catalog
                - name: SPRING_PROFILES_ACTIVE
                  value: testdata
    ################################################

    ### service.yml ################################
    apiVersion: v1
    kind: Service
    metadata:
      name: catalog-service
      labels:
        app: catalog-service
    spec:
      type: ClusterIP
      selector:
        app: catalog-service
      ports:
      - protocol: TCP
        port: 80
        targetPort: 9001  
    ################################################

    kubectl apply -f deployment.yml
    kubectl apply -f service.yml
