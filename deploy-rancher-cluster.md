# Deploy Rancher Cluster on AWS

## Requirement

1. Deploy cluster in Private Subnet
2. Cluster include 1 Master + 1 Worker
3. AWS ALB to access Master UI.
4. User Rancher CLI rancher-compose to deploy stack to Rancher cluster:
    * A rancher load balancer
    * A tomcat 9 server behind the load balancer
    * Expose stack behind AWS ALB https - 443

## Deploy Rancher Cluster: Step by Step 

1. Create VPC with 3 subnets: 2 Public (2 different AZ) and 1 Private (because we want to use ALB and it requires to pick at least 2 az).
2. Create following Security Group:
* **internal-private-subnet-sg**: Allow instances associated with this SG talk to others. It's obviously not the recommended practice but this is for our convenience.

* **bastion-ssh-sg**: allow  SSH - Port 22 - Your IP.

* **allow-ssh-from-bastion-sg**: SSH - Port 22 - bastion-ssh-sg

* **rancher-admin-alb-sg**: HTTP/HTTPS - 80/443 - Your IP.

* **allow-trafic-from-alb-sg**: tcp - 8080 - alb sg

3. Create following Key-pair:

* bastion-key.pem

* rancher-master-key.pem

* rancher-worker-key.pem

4. Create Bastion instance:

* OS: amazon linux
* Instance type: t2.micro
* Public Subnet
* Security Group: bastion-ssh-sg
* Key-pair: bastion-key.pem
* Everything else: default

5. Create Rancher Master instance:

* OS: RancherOS. AMI: ami-9d8293e6
* Instance Type: t2.small
* Private Subnet
* Storage: 20GB ssd
* Security Group: internal-private-subnet-sg; allow-ssh-from-bastion-sg; allow-trafic-from-alb-sg
* Key-pair: rancher-master-key.pem
* Everything else: default

6. Create Rancher Worker instance:

* OS: RancherOS. AMI: ami-9d8293e6
* Instance Type: t2.small
* Private Subnet
* Storage: 20GB ssd
* Security Group: internal-private-subnet-sg; allow-ssh-from-bastion-sg;
* Key-pair: rancher-worker-key.pem
* Everything else: default

7. Create the ALB:

* Step 1: Configure Load Balancer
    * Name: rancher-master-alb
    * Scheme: internet facing
    * Listener: http - port 80
    * AZ: choose 2 public subnets in your created VPC
* Step 2: Configure Security Settings: NONE
* Step 3: Configure Security Groups
    * Select existing Security Group: rancher-admin-alb-sg
* Step 4: Configure Routing
    * Name: rancher-master-tg
    * Protocol: http
    * Port: 80
    * Target Type: instance
    * Health Check:  HTTP
    * Path: /ping
* Step 5: Register Targets
    * Add rancher master instance - port 8080

7. Config SSH ProxyCommand to ssh to both private instances. Go [here] (bastion-ssh-proxycommand.md) to be fabulous!

8. SSH to rancher master instance. Run:

```
sudo docker run -d --restart=always -p 8080:8080 rancher/server
```
Wait for the command to finish.

9. Open browser -> copy the ALB DNS. It should load the Rancher admin UI. FEELING GOOD?!

10. From Rancher UI -> Click Infrastructure -> Host -> Add Host
* Host Registration URL: 
    * Something Else: rancher_master_private_ip:8080
* Choose Custom Host:
    * No4: rancher_worker_private_ip
    * Copy the command
    * Click Close

11. SSH to rancher worker instance. Run the copy command. Wait for the command to finish.

12. Check The Infrastructure > Host. A new Host should appear. 

13. Done deploying Rancher Cluster.


## Deploy Tomcat9 behind a Rancher Load Balancer using rancher-compose: Step-by-step

1. Create a Self Signed Certificate then import to AWS Certificate Manager.

2. Create rancher worker ALB:

* Step 1: Configure Load Balancer
    * Name: rancher-worker-alb
    * Scheme: internet facing
    * Listener: https - port 443
    * AZ: choose 2 public subnets in your created VPC
* Step 2: Configure Security Settings:
    * Choose the Certificate in #1
* Step 3: Configure Security Groups
    * Select existing Security Group: rancher-worker-alb-sg
    * If not existed: create new one to allow HTTPS/443
* Step 4: Configure Routing
    * Name: rancher-worker-tg
    * Protocol: http
    * Port: 8080
    * Target Type: instance
    * Health Check:  HTTP
    * Path: /ping
* Step 5: Register Targets
    * Add rancher worker instance - port 8080

3. Go to Rancher Admin UI > Stacks > Add Stack.

* Name: tomcat9-with-load-balancer
* docker-compose.yml

```
version: '2'
services:
  web:
    image: tomcat:9-jre8-alpine
  lb:
    image: rancher/lb-service-haproxy
    ports:
    - 8080:8080/tcp
    labels:
      io.rancher.container.agent.role: environment
      io.rancher.container.create_agent: 'true'

```

* rancher-compose.yml

```
version: '2'
services:
  web:
    drain_timeout_ms: 10000
    scale: 1
    start_on_create: true
  lb:
    scale: 1
    start_on_create: true
    lb_config:
      certs: []
      port_rules:
      - priority: 1
        protocol: http
        service: web
        source_port: 8080
        target_port: 8080
    health_check:
      healthy_threshold: 2
      response_timeout: 2000
      port: 42
      unhealthy_threshold: 3
      interval: 2000
      strategy: recreate
```

Wait for stack to be created. When stack is up - Green. Go to rancher-worker alb dns. It should show the tomcat default page.