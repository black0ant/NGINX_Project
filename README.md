# NGINX HA Reverse Proxy with Load Balancing
 
## Overview
High availability NGINX setup with:
- 2 backend servers serving static pages
- 2 load balancers with NGINX + Keepalived for VIP failover
- load balancing
- VIP failover on LB failure
 
## Infrastructure
- 4 EC2 Instances on AWS Free Tier
- Ubuntu 22.04
- Keepalived + NGINX setup
 
## Setup Instructions
 
### Start Backend Servers
1. Install NGINX
   sudo apt update -y
   sudo apt install nginx -y
   sudo vi /etc/nginx/conf.d/load_balancer.conf
           upstream backend {
              server 172.31.24.119;  #Backend noda A private IP
              server 172.31.91.89;   #Backend noda B private IP
            }
          server {
              listen 80;
 
              location / {
                  proxy_pass http://backend;
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
                  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               }
             }
   sudo nginx -t
   sudo systemctl restart nginx
2. Set custom HTML for Node A and Node B
   sudo apt update -y
   sudo apt install nginx -y
   echo "<h1>This is Backend Node A</h1>" | sudo tee /var/www/html/index.html   #run on backend node A
   echo "<h1>This is Backend Node B</h1>" | sudo tee /var/www/html/index.html   #run on backend node B
 
### Start Load Balancers
1. Install NGINX + Keepalived
   sudo apt install keepalived -y
2. Configure NGINX reverse proxy
3. Add Keepalived config for VIP
   sudo vi /etc/keepalived/keepalived.conf
   vrrp_instance VI_1 {
    state MASTER   #master on lb-1 and backup on lb-2
    interface enX0    # ip -o link show | awk -F': ' '{print$2}'
    virtual_router_id 51
    priority 100    #100 on lb-1 and 90 on lb-2
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass secret1
    }
    virtual_ipaddress {
    172.31.82.150        #for this check if IP is free or not using the following command on your instance sudo ping -c 2 172.31.82.150
    }
  }
   sudo systemctl start keepalived
   #in case of any issues while starting keepalived use below command to debug
   sudo keepalived -n -l -D
   sudo rm /etc/nginx/sites-enabled/default  #to remove the default template
 
### Test VIP and Load Balancing
```bash
curl http://172.31.82.150
<h1>This is Backend Node A</h1>  # Alternates between Node A / B
