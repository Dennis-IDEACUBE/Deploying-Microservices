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
    ## ALL: (공통사항)
    
    sudo su
    
    printf "\n10.0.2.4 k8s-control\n10.0.2.5 k8s-1\n10.0.2.6 k8s-1\n\n" >> /etc/hosts
    
    printf "overlay\nbr_netfilter\n" >> /etc/modules-load.d/containerd.conf
    
    modprobe overlay
    modprobe br_netfilter
    
    printf "net.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\n" >> /etc/sysctl.d/99-kubernetes-cri.conf
    
    sysctl --system
    
    wget https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz -P /tmp/
    tar Cxzvf /usr/local /tmp/containerd-1.7.13-linux-amd64.tar.gz
    wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
    systemctl daemon-reload
    systemctl enable --now containerd
    
    wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 -P /tmp/
    install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc
    
    wget https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz -P /tmp/
    mkdir -p /opt/cni/bin
    tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.4.0.tgz
    
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml   <<<<<<<<<<< manually edit and change SystemdCgroup to true (not systemd_cgroup)
    vi /etc/containerd/config.toml
    systemctl restart containerd
    
    swapoff -a  <<<<<<<< just disable it in /etc/fstab instead
    
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gpg
    
    mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    
    apt-get update
    
    reboot
    
    sudo su
    
    apt-get install -y kubelet=1.29.1-1.1 kubeadm=1.29.1-1.1 kubectl=1.29.1-1.1
    apt-mark hold kubelet kubeadm kubectl
    
    # check swap config, ensure swap is 0
    free -m
    
    
    ### CONTROL PLANE 부분만 설정 ###
    ### ONLY ON CONTROL NODE .. control plane install:
    kubeadm init --pod-network-cidr 10.10.0.0/16 --kubernetes-version 1.29.1 --node-name k8s-control
    
    export KUBECONFIG=/etc/kubernetes/admin.conf
    
    # add Calico 3.27.2 CNI 
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/tigera-operator.yaml
    wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/custom-resources.yaml
    vi custom-resources.yaml <<<<<< edit the CIDR for pods if its custom
    kubectl apply -f custom-resources.yaml
    
    # get worker node commands to run to join additional nodes into cluster
    kubeadm token create --print-join-command
    ###
    
    
    ### ONLY ON WORKER nodes
    Run the command from the token create output above

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
