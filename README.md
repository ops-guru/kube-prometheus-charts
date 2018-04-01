# kube-prometheus-charts
Helm charts for installation of kube-prometheus with OpsGuru added value features

# Preparations

Create a role and user for kubernetes following the instructions from [here](https://github.com/kubernetes/kops/blob/master/docs/aws.md#setup-iam-user)

## Install aws cli tool:
```
apt-get install python-pip jq
pip install awscli
# configure aws
aws configure
```

## Install kops
For 1.9 support we need to download the latest alpha
```
wget https://github.com/kubernetes/kops/releases/download/1.9.0-alpha.2/kops-linux-amd64 -O kops
chmod +x kops
mv kops /usr/bin
```

## Install kubectl
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Install helm
```
curl https://storage.googleapis.com/kubernetes-helm/helm-v2.8.2-linux-amd64.tar.gz | tar xvz
mv ./linux-amd64/helm /usr/bin
```

Save your public key to ~/.ssh/id_rsa.pub

## Installation

Those variables are needed by kops:
```
export AWS_ACCESS_KEY_ID=$(grep aws_access_key_id ~/.aws/config | gawk '{print $NF}')
export AWS_SECRET_ACCESS_KEY=$(grep aws_secret_access_key ~/.aws/config | gawk '{print $NF}')
export AWS_REGION=us-east-1
export AWS_VPCID=vpc-8d7cb4f6

# export DOMAIN=cncf-meetup.org # not needed because we use an external domain
export BUCKET_NAME=k8s-cncf
export NAME=cncf-kops.k8s.local
export KOPS_STATE_STORE=s3://$BUCKET_NAME
```

Create needed AWS resources:
```
# the kops s3 bucket:
aws s3api create-bucket --bucket $BUCKET_NAME
# a private hosted zone with the desired domain:
# aws route53 create-hosted-zone --name "${DOMAIN}." --vpc  VPCRegion=${AWS_REGION},VPCId=${AWS_VPCID} --caller-reference $(uuidgen) # not needed because we use an external domain
```

Start the initial config
```
kops create cluster --zones us-east-1b --kubernetes-version 1.9.3 --master-count 1 $NAME  --associate-public-ip=false --topology private --networking flannel
```

We want to update the instancegroups.

Change the nodes to spot:

```
kops edit instancegroup nodes --name $NAME
# add this in spec:
  maxPrice: "0.02"
```

The default instance size wil be m3.medium for masters and t2.medium for nodes.

Also, we are using external-dns depoyment, so we want to give access to route53:

```
kops edit cluster $NAME
# Add the following policies:
  additionalPolicies:
    node: |
      [
        {
          "Effect": "Allow",
          "Action": ["route53:ChangeResourceRecordSets"],
          "Resource": ["arn:aws:route53:::hostedzone/*"]
        },
        {
          "Effect": "Allow",
          "Action": ["route53:ListHostedZones", "route53:ListResourceRecordSets"],
          "Resource": ["*"]
        }
      ]
```

Bootstrap the cluster:
```
kops update cluster --name $NAME --yes
```

After it has finished, perform some checks:
```
until kops validate cluster;do sleep 5;done
kubectl get nodes --show-labels
kubectl get pods --all-namespaces -o wide
```

## Install tiller
```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
# wait for tiller to be deployed
until helm list;do sleep 5;done
```

## Install prometheus

Standalone:
```
# cleanup if we installed the operator before
helm del --purge cncf
helm del --purge cncf-prometheus-operator
# install
helm install ./helm/prometheus-standalone --name cncf-standalone
```

Or prometheus operator:
```
# cleanup if we installed the standalone before
helm del --purge cncf-standalone
# install
helm install ./helm/prometheus-operator --name cncf-prometheus-operator
helm install ./helm/cncf --name cncf
```

## DNS - deprecated by using an external domain

If you are connecting via the vpn link, update your resolver to include the amazon dns server:

* you are lucky and you can add "nameserver 10.8.0.2" to /etc/resolv.conf
```
nameserver 10.8.0.2
```
* systemd-resolved has taken over your computer and you need to update /etc/systemd/resolved.conf
```
[Resolve]
DNS=10.8.0.2
```

Check that you can access:  [alertmanager](http://alertmanager.cncf-meetup.org), [prometheus](http://prometheus.cncf-meetup.org), [grafana](http://grafana.cncf-meetup.org)

## Post config:

For metrics and other connections, open all traffic between your machines security group to master and nodes security groups (etcd metrics need port 4001 which is closed with the default securitygroup created by kops).

### In order to connect to the entire cluster:

Peer the kubernetes vpc with the vpc from where you work. Add routes to each other vpn.

```ssh -i ~/.ssh/id_rsa admin@172.20.46.229```

# Testing

## Operator

* Let's create a deployment with 2 replicas.
* Check that new metrics appear in prometheus with name nginx_http_request_...
* Plot that metrics from prometheus (or grafana): in prometheus graph 'nginx_http_request_duration_seconds_count'.
* Create an alert that checks that the desired number of replicas is deployed.
* Force the alert to fire by modifying the number of replicas.

```
# cleanup
kubectl delete -f example/.yaml
# install
kubectl create -f example/nginx.yaml
# connect to http://echoheaders.opsguru-cncf.com to see the example pod
# generate some traffic:
kubectl create -f example/generate-traffic.yaml
# metrics can be found at http://echoheaders.opsguru-cncf.com/metrics
# add an alert:
kubectl create -f example/alert-operator.yaml
# force the alert to fire:
kubectl patch deployment/echoheaders -p '{"spec":{"replicas":3}}'
```

The deployment we just created uses podAntiAffinity to force the pods to not deploy on the same node, and also to not deploy on any node where etcd is running. By increasing the number of replicas from 2 to 3, we force the alert to fire. This happens because we have 1 master and 2 nodes. When the 3rd pod needs to be scheduled it will fail.

## Standalone

For standalone prometheus we need to update in values.yaml the field:
```
serverFiles:
  alerts:
```
The same example alert is already created when deploying the chart, as it was added in values.yaml.

Since this is the only config map mounted, in order to add new rules, we have to update the yaml file and upgrade the chart.

The same steps as before can be applied (minus the apply for alert-operator.yaml), with the same results. If we use the exact same files we will get an error about not beying able to create : "monitoring.coreos.com/, Kind=ServiceMonitor". Wich is expected, as we don't have this resource now.

## Grafana

Copy the file ./example/example-dashboard.json to ./helm/cncf/charts/grafana/dashboards/

Upgrade the chart in order to take into consideration the new file:
```
helm upgrade cncf ./helm/cncf/
```

# Cleanup

```
helm del --purge cncf
helm del --purge cncf-prometheus-operator
helm del --purge cncf-standalone
kops delete cluster $NAME --yes
```
