# Running a Kubernetes Cluster

## How to Reset the firewall zone

```bash
 firewall-cmd --new-zone=kubernetes --permanent
 firewall-cmd --reload
 firewall-cmd --get-zones
 firewall-cmd --zone=kubernetes --add-source=192.168.0.1/16 --permanent
 firewall-cmd --zone=kubernetes --add-port=6443/tcp --permanent
 firewall-cmd --reload
 firewall-cmd --zone=kubernetes --list-all 
 firewall-cmd --zone=kubernetes --add-source=192.168.0.2/16 --permanent
 firewall-cmd --zone=kubernetes --list-all 
 firewall-cmd --zone=kubernetes --add-source=192.168.0.0/16 --permanent
 firewall-cmd --zone=kubernetes --remove-source=192.168.0.1/16 --permanent
 firewall-cmd --zone=kubernetes --add-source=192.168.0.0/16 --permanent
 firewall-cmd --zone=kubernetes --add-source=192.168.0.2/16 --permanent
 firewall-cmd --zone=kubernetes --list-all 
 firewall-cmd  --list-all 
 firewall-cmd --zone=public --remove-source=192.168.0.2/16 --permanent
 firewall-cmd --zone=public --remove-source=192.168.0.0/16 --permanent
 firewall-cmd --reload
 firewall-cmd --zone=kubernetes --add-source=192.168.0.2/16 --permanent
 firewall-cmd --zone=kubernetes --list-all 
 firewall-cmd --zone=kubernetes --add-source=192.168.0.0/16 --permanent
 firewall-cmd --reload
 firewall-cmd --zone=kubernetes --list-all
 ```

 ## How to run Spark 3 on kubernetes

 ### Methode 1 - Spark 3 Native on kubernetes (spark-submit)

 #### how to build the image
Becareful the build is os arch dependent
This eample is done on RHEL9 aarch64 build
ssh edge31.clemlab.com
 ```bash
 
  cd /usr/odp/1.2.4.0-102/spark3
  ./bin/docker-image-tool.sh -r nexus.clemlab.com:5000 -t v3.5.1.1.2.4.0-102-arm build




  docker login -u jenkins nexus.clemlab.com:5000
  docker push nexus.clemlab.com:5000/spark:v3.5.1.1.2.4.0-102-arm
  ```

  #### how to run the image

  ssh edge31.clemlab.com

  * edit the required configuration
  ```bash
  mkdir -p /etc/spark3/conf
  ln -s /etc/spark3/conf /usr/odp/1.2.4.0-102/spark3/conf
  ```
  
  * edit a krb5.conf
  ```
  cp /etc/krb5/conf /etc/spark3/conf/
  ```
  
  * edit spark-defaults
  ``` bash
    vim /etc/spark3/conf/spark-defaults.conf
    spark.master k8s://https://kubernetes01.dev.clemlab.com
    spark.kubernetes.container.image=nexus.clemlab.com:5000/spark:v3.5.1.1.2.4.0-102-arm
    spark.kubernetes.namespace=default
    spark.kubernetes.authenticate.driver.serviceAccountName=spark
    spark.kubernetes.driver.pod.name=spark-pi-driver
    spark.executor.instances=2
    spark.executor.memory=1g
    spark.executor.cores=1
    spark.driver.memory=1g
  ```

  ** give the right to spark service account
  ```
  kubectl create serviceaccount spark -n default
  ```

  ** create a spark role
  ```yaml vi spark-role.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: spark-role
      namespace: default
    rules:
      - apiGroups: [""]
        resources: ["pods", "pods/log", "services", "configmaps","persistentvolumeclaims"]
        verbs: ["create", "get", "watch", "list", "delete","deletecollection"]
      - apiGroups: [""]
        resources: ["secrets"]
        verbs: ["get"]
  ```

    ```yaml vi spark-role-binding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: spark-role-binding
      namespace: default
    subjects:
      - kind: ServiceAccount
        name: spark
        namespace: default
    roleRef:
      kind: Role
      name: spark-role
      apiGroup: rbac.authorization.k8s.io

  ```

  ```bash
  kubectl apply -f spark-role.yaml
  kubectl apply -f spark-role-binding.yaml
  ```

  test access rights
  ```bash
  kubectl auth can-i deletecollection configmaps --as=system:serviceaccount:default:spark -n default
  ```

  now submit the job
  ```bash
  spark-submit --master k8s://192.168.1.182:6443  --deploy-mode cluster --name spark-pi --class org.apache.spark.examples.SparkPi --conf spark.kubernetes.container.image=nexus.clemlab.com:5000/spark:v3.5.1.1.2.4.0-102-arm local:///opt/spark/examples/jars/spark-examples_2.12-3.5.1.1.2.4.0-102.jar
  ```

  ### Methode 2 - Running Spark 3 using Official Spark Operator on kubernetes (kubectl api)

  * Installation of the spark Operator using helm

  using the ansible playbook 10-setup-k8s-runtime.yml
  
  ```bash
    ansible-playbook -i  environments/hosts_dev_arm_m2 10-setup-k8s-runtime.yml --tags spark_operator --become --limit edge31.clemlab.com --ask-vault-pass
  
  ```

  It looks like
  ```yaml
    - name: Check if Helm is installed
      command: /usr/local/bin/helm version --short
      register: helm_version
      ignore_errors: yes

    - name: Set Helm installed fact
      set_fact:
        helm_installed: "{{ 'v3.17.0' in helm_version.stdout }}"

    - name: Download Helm binary
      get_url:
        url: https://get.helm.sh/helm-v3.17.0-linux-arm64.tar.gz
        dest: /tmp/helm-v3.17.0-linux-arm64.tar.gz
      when: not helm_installed

    - name: create /opt/local/helm
      ansible.builtin.file:
        dest: /opt/local/helm
        state: directory
      when: not helm_installed

    - name: Extract Helm binary
      ansible.builtin.command:
        cmd: tar -xzf /tmp/helm-v3.17.0-linux-arm64.tar.gz --strip-components=1 -C /opt/local/helm/
      when: not helm_installed

    - name: symlink /usr/local/bin/heml to /opt/local/helm/helm
      ansible.builtin.file:
        src: /opt/local/helm/helm
        dest: /usr/local/bin/helm
        state: link

    - name: Ensure pip is installed
      ansible.builtin.raw: |
        if ! command -v pip &> /dev/null; then
          apt-get update && apt-get install -y python3-pip
        fi
      when: ansible_distribution == "Ubuntu"

    - name: Ensure pip is installed
      ansible.builtin.raw: |
        if ! command -v pip &> /dev/null; then
          yum install -y python3-pip
        fi
      when: ansible_distribution == "RedHat"

    - name: install pre-requisites
      pip:
        name:
          - openshift
          - pyyaml
          - kubernetes 

    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: "{{ spark_namespace }}"

    - name: Add Helm Spark Operator repositories
      ansible.builtin.command:
        cmd: /usr/local/bin/helm repo add spark-operator https://kubeflow.github.io/spark-operator

    - name: Update Helm repositories
      ansible.builtin.command:
        cmd: /usr/local/bin/helm repo update

    - name: Install Spark Operator
      ansible.builtin.command:
        cmd: "/usr/local/bin/helm install spark-operator spark-operator/spark-operator --namespace {{spark_namespace}} --create-namespace"
  ```

  en bash

  ```sh
  /usr/local/bin/helm repo add spark-operator https://kubeflow.github.io/spark-operator
  /usr/local/bin/helm repo update
  /usr/local/bin/helm install spark-operator spark-operator/spark-operator --namespace '{{spark_namespace}}' --create-namespace"
  ```

  Then edit the file `vim spark-application-run-method-2-with-operator.yml`
```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: spark-pi-method-two-with-spark-operator
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: nexus.clemlab.com:5000/spark:v3.5.1.1.2.4.0-102-arm
  imagePullPolicy: Always
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.5.1.1.2.4.0-102.jar"
  sparkVersion: "3.5.1.1.2.4.0-102"
  restartPolicy:
    type: Never
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: "3.5.1.1.2.4.0-102"
    serviceAccount: spark
  executor:
    cores: 1
    instances: 2
    memory: "512m"
    labels:
      version: "3.5.1.1.2.4.0-102"
```

  ```bash
  kubectl apply -f spark-application-run-method-2-with-operator.ym
  ```

### Methode 3 - Running Spark 3 using Yunikorn Scheduler (spark-submit)

Documentation for [installing the scheduler is here](https://yunikorn.apache.org/docs/#install)
after running the following commands
```bash
helm repo add yunikorn https://apache.github.io/yunikorn-release
helm repo update
kubectl create namespace yunikorn
helm install yunikorn yunikorn/yunikorn --namespace yunikorn
```

The scheduler is started on one of the nodes
We can enable access to it by checking on which node it does run
and forwarding the port (we could also create a node port)
```bash
kubectl get pods -o wide -n yunikorn-namespace | grep scheduler
kubectl port-forward svc/yunikorn-service 9889:9889 -n yunikorn-namespace --address 0.0.0.0
```

Then access the ui for me the link is http://192.168.1.186:9889/#/error?last=%2Fdashboard

ATTENTION the SCHEDULER YUNIKORN REMPLACE le SCHEDULER KUBE

Then create the queues

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: yunikorn-namespace
  name: yunikorn-configs
data:
  queues.yaml: |
    partitions:
      - name: default
        placementrules:
          - name: tag
            value: developper
            create: true
        queues:
          - name: bdfqueue
            submitacl: '*'
```