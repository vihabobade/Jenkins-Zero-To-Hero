How To Deploy Spring Boot With Amazon EKS

Prerequisites
Follow the getting started guide and make sure the following are installed:

    AWS CLI
    kubectl
    eksctl

https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html

Initial Setup

Set up Spring Boot.

    Use Spring Initializr to generate a new project.
    Select Spring Web as dependency.
    (Use any springboot JAVA project)

Create Dockerfile.

   # You can change this base image to anything else
   # But make sure to use the correct version of Java
   FROM folioci/alpine-jre-openjdk11

   # Simply the artifact path
   ARG artifact=target/spring-boot-web.jar

   WORKDIR /opt/app

   COPY ${artifact} app.jar

   # This should not be changed
   ENTRYPOINT ["java","-jar","app.jar"]


Create an executable jar.

mvn clean package

Run container.

docker run -d -p 8080:8080 spring-boot-web-app


Elastic Container Registry (ECR) Setup

    Go to Amazon ECR.
    Create a repository.
    Click on "View push commands".
    Follow the instructions to push image.
    Copy the image url.

Elastic Kubernetes Service (EKS) Setup


Create EKS cluster.

   eksctl create cluster --name spring-boot-app-cluster --region us-west-2

   Confirm communication with cluster.

   kubectl get svc


Create yaml configuration. k8s.yaml

Apply manifest.

   kubectl apply -f k8s.yaml

Confirm pods.

   kubectl get pod

Open shell into pod.

   kubectl exec -it myapp-7487b94844-wvfmn -- bash # (Change pod name)

Confirm load balancer.

   kubectl get svc

Confirm load balancer domain.

   nslookup af220a880523a491a991ecc793ae1da2-125775810.us-west-2.elb.amazonaws.com (Change svc name)

   **Note: This might take a few minutes.

Set up AWS Load Balancer Controller

For more complex traffic routing, see https://www.fullstackbook.com/devops/tutorials/how-to-set-up-aws-load-balancer-controller

Get arn account number by running command:

   aws sts get-caller-identity

  **Note- This is needed for creating the iamserviceaccount.


Here is a condensed version of the instructions:

   oidc_id=$(aws eks describe-cluster --name spring-boot-app-cluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

   aws iam list-open-id-connect-providers | grep $oidc_id

   eksctl utils associate-iam-oidc-provider --cluster spring-boot-app-cluster --approve

Set up Load Balancer. Note: Replace the arn account number and cluster name where applicable.

   curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json

   aws iam create-policy \
      --policy-name AWSLoadBalancerControllerIAMPolicy \
      --policy-document file://iam_policy.json

   eksctl create iamserviceaccount \
   --cluster=spring-boot-app-cluster \
   --namespace=kube-system \
   --name=aws-load-balancer-controller \
   --role-name "AmazonEKSLoadBalancerControllerRole" \
   --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
   --approve

   kubectl apply \
      --validate=false \
      -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml

   curl -Lo v2_4_4_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_full.yaml

   sed -i.bak -e '480,488d' ./v2_4_4_full.yaml

   sed -i.bak -e 's|your-cluster-name|spring-boot-app-cluster|' ./v2_4_4_full.yaml

   kubectl apply -f v2_4_4_full.yaml

   curl -Lo v2_4_4_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_ingclass.yaml

   kubectl apply -f v2_4_4_ingclass.yaml


Verify controller is installed.

   kubectl get deployment -n kube-system aws-load-balancer-controller

Set up Ingress

Go to https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/examples/2048/2048_full.yaml. Note the Ingress section.


   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
   namespace: default
   name: my-ingress
   annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
   spec:
   ingressClassName: alb
   rules:
      - http:
         paths:
            - path: /
               pathType: Prefix
               backend:
               service:
                  name: spring-boot-app-service
                  port:
                     number: 80


Apply the ingress.

   kubectl apply -f ingress.yaml

Confirm load balancer. This may take a few minutes.

   kubectl get ingress

Delete cluster

Delete the cluster avoid large AWS bill.

eksctl delete cluster --name spring-boot-app-cluster