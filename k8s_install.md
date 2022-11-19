1. Certificate Authority
In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Generate the CA configuration file, certificate, and private key:

{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"             #1.year
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",                 #C:表示国家
      "L": "Portland",           #L:城市
      "O": "Kubernetes",         #O:组织、公司
      "OU": "CA",                #OU:单位、部门
      "ST": "Oregon"             #ST:省份
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
Results:

ca-key.pem
ca.pem

2. The Admin Client Certificate
Generate the admin client certificate and private key:

{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
Results:

admin-key.pem
admin.pem
  
  
3. 生成Kubelet Client Certificates
#说明批量生成Node节点证书
#!/bin/bash

NODES="ip1:worker001.pe.x.cn \
ip2:worker002.pe.x.cn \
ip3:worker003.pe.x.cn"



function GenInstanceCsr() {
    local instance=$1
    local instanceCsr=${instance}-csr.json
    cat > ${instanceCsr} <<EOF
{
  "CN": "system:node:${instance}",             #此处CN指定为 system:node:${instance}
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "CS",
      "O": "system:nodes",                    #此处指定为system:nodes
      "OU": "Kubernetes XXX",
      "ST": "HUNAN"
    }
  ]
}
EOF
}


function genCert(){
    local instance=$1
    local INTERNAL_IP=$2
    #local EXTERNAL_IP=$3
    cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${INTERNAL_IP} \                  #此处只指定了Node节点主机名和ip地址？
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
}

function main(){
    PATH=.:$PATH
    for i in $NODES; do
        i_ip=$(echo  $i |awk -F':' '{print $1}')
        instance=$(echo  $i |awk -F':' '{print $2}')
        GenInstanceCsr $instance
        genCert $instance $i_ip
    done
}


main
  
  
work001.xxx-csr.json   //证书请求json文件
work001.xxx.csr        //证书请求文件
work001.xxx-key.pem    //证书-私钥
work001.xxx.pem        //证书-公钥
  
4. The Controller Manager Client Certificate
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
Results:

kube-controller-manager-key.pem
kube-controller-manager.pem
  
  
5. The Kube Proxy Client Certificate  #通用的？
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
Results:

kube-proxy-key.pem
kube-proxy.pem

  
6. The Scheduler Client Certificate
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
Results:

kube-scheduler-key.pem
kube-scheduler.pem
  
  
  
7. The Kubernetes API Server Certificate
{

KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
The Kubernetes API server is automatically assigned the kubernetes internal dns name, which will be linked to the first IP address (10.32.0.1) from the address range (10.32.0.0/24) reserved for internal cluster services during the control plane bootstrapping lab.

Results:

kubernetes-key.pem
kubernetes.pem
  
  
8. The Service Account Key Pair
  
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
Results:

service-account-key.pem
service-account.pem
  
  
9. Distribute the Client and Server Certificates
  
