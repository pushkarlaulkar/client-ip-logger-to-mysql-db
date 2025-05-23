Instructions to deploy **Client IP Logger in MySQL DB** on AWS EKS Auto Mode
  1. Deploy EKS cluster through Auto Mode through AWS Console. Add the ` CoreDNS `, ` VPC CNI `, ` Kube Proxy `, `Amazon EBS CSI Driver` add ons. NAT Gateway is also needed since the worker nodes will be deployed in private subnets.
  2. Deploy **MySQL Replication Cluster** ( 1 primary, 2 secondary ) helm chart. MySQL will be deployed as a **StatefulSet**. Set the **auth.rootPassword** & **auth.replicationPassword** of your choice.
     
     ```
     helm repo add bitnami https://charts.bitnami.com/bitnami
     helm repo update
     
     helm install mysql-cluster bitnami/mysql \
        --namespace db \
        --create-namespace \
        --set architecture=replication \
        --set auth.rootPassword= \
        --set auth.replicationPassword= \
        --set primary.persistence.enabled=true \
        --set primary.persistence.size=8Gi \
        --set primary.persistence.storageClass=aws-ebs \
        --set secondary.replicaCount=2 \
        --set secondary.persistence.enabled=true \
        --set secondary.persistence.size=8Gi \
        --set secondary.persistence.storageClass=aws-ebs \
        --set volumePermissions.enabled=true
     ```
  3. Create the Table in MySQL which will store the Client IP

     ```
     kubectl exec -it -n db mysql-cluster-primary-0 -- bash
     mysql -uroot -p ( Enter root user password when prompted )

     CREATE DATABASE IF NOT EXISTS flaskdb;
     USE flaskdb;

     CREATE TABLE IF NOT EXISTS client_ips ( id INT AUTO_INCREMENT PRIMARY KEY, ip_address VARCHAR(45) );
     ```
  4. Create a namespace **web** where the application will be deployed.
  5. Put the base64 encoded value of the **auth.rootPassword** set above in **mysql-secret.yaml** & then create the **Secret**.
     
     ```
     kubectl -n web apply -f mysql-secret.yaml
     ```
  6. Create the **ConfigMap** defined in **mysql-configmap.yaml**. The ConfigMap has the values for database host, name & user.
     
     ```
     kubectl -n web apply -f mysql-configmap.yaml
     ```
  7. Create the **Deployment** & **Service**. 

     ```
     kubectl -n web apply -f flask-deployment.yaml -f flask-service.yaml
     ```
  8. Deploy **Ingress**, **Ingress Class** & **Ingress Params** which will create an ALB listening on port 443. We will need to provide the arn of the certificate in the **Ingress** object. The certificate for the domain name needs to be created in ACM and DNS validation or Email validation needs to be done prior to creating these resources.
  9. Run the command ` kubectl -n web apply -f ingress-all.yml `. Run ` kubectl -n web get ingress ` to retrieve the ALB DNS. Point the domain name in Route 53 to the ALB as an A (alias) record.
 10. Access the app using `https://your_domain_name`.