---
layout: post
title: "Using Kubernetes Ingress with an On-Prem Cluster For Better URLs Than NodePort"
date: 2021-09-08
categories: kubernetes nginx ingress
---

If you start learning kubernetes on-prem you will quickly be confused on exposing your pods using kubernetes services.

You will have no choice but to use _nodePort_ to get access to your deployments.

This creates a pretty ugly url to share with teammates, along with unsuspecting applications that have trouble working with a port that isn't 443.

`https://vlabxxx.domxxx.lab:31244`

Ideally what you would want is a kubernetes service type of _LoadBalancer_, but this is unavailable in non-cloud kubernetes deployments.

In this article I will teach you how to use ingress to expose your applications with a very friendly URL.

`https://mypod.exampledomain.lab`

```

 *.exampledomain.lab      (nginx)
┌───┐                 ┌─────────────┐   ┌──────────────────────┐
│DNS├───────────────► │load balancer├──►│Kubernetes Worker Node│
└───┘                 └─────────────┘   └──────────┬───────────┘
                      ┌───────────┐  ┌───────┐     │
                      │Application│◄─┤Ingress│ ◄───┘
                      └───────────┘  └───────┘
```

## Create a wildcard DNS A record or CNAME record

You will need to (or request) wildcard dns record(s) for a dns server that your machine and other machines with the network use as a DNS server.

an example record could be

Hostname The Name of the machine. (ex. vlab123456 or vlab123456.subdomain )

`*`

DNS_ZoneThe DNS Zone (ex. dom930041.lab)

`exampledomain.lab`

TargetIPAddress for an A or PTR record. The FQDN target for a CNAME

`10.123.2.1`

Record Type

`A`

What this will do is forward all `*.exampledomain.lab` traffic to `10.123.2.1`

We will create an nginx loadbalancer on the machine located at that IP

## Create a load-balancer with wildcard listener

I chose nginx because of familiarity, however concepts can apply to other load balancers/proxys

Here is an example config.

The important part to note is how I propagate down the host header via
`proxy_set_header Host $host;`

```nginx
load_module /usr/lib/nginx/modules/ngx_stream_module.so;
worker_processes auto;
error_log /var/log/nginx/error.log error;


events {
	worker_connections  1024;
}

http{
	upstream servers{
		keepalive 100;
		server 10.204.238.72:443 weight=5;
		server 10.204.238.67:443 fail_timeout=5s;
		server 10.204.238.71:443 backup;

	server {
		listen 80;
		listen 443 ssl;
		server_name *.exampledomain.lab;
		include snippets/self-signed.conf;
		include snippets/ssl-params.conf;
		location / {
			proxy_set_header Host $host;
			proxy_pass https://servers;
		}
	}

}
```

## Create SSL for Nginx

To satisfy the config snippets

```nginx
include snippets/self-signed.conf;
include snippets/ssl-params.conf;
```

We need to create a self-signed key and certificate pair

```sh
sudo openssl req -x509 \
-nodes -days 365 -newkey \
rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key \
-out /etc/ssl/certs/nginx-selfsigned.crt
```

Create a strong Diffie-Hellman group

`sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048`

Configure Nginx to use new cert

You can put these in a seperate conf file `/etc/nginx/snippets/self-signed.conf`

```nginx
ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

You can also create SSL parameters in `/etc/nginx/snippets/ssl-params.conf`

```nginx
ssl_protocols TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/ssl/certs/dhparam.pem;
ssl_ciphers ALL:!ADH:!EXP:!LOW:!RC2:!3DES:!SEED:!RC4:+HIGH:+MEDIUM;
ssl_ecdh_curve secp384r1; # Requires nginx >= 1.1.0
ssl_session_timeout  10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off; # Requires nginx >= 1.5.9
# ssl_stapling on; # Requires nginx >= 1.3.7
# ssl_stapling_verify on; # Requires nginx => 1.3.7
# add_header X-Frame-Options SAME-ORIGIN;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
```

and "import/include" them in your SSL declaration

```nginx
listen 80;
listen 443 ssl;
server_name *.exampledomain.lab;
include snippets/self-signed.conf;
include snippets/ssl-params.conf;
```

## Kubernetes Ingress

Once the wildcard DNS record and Nginx are setup then you can start resolving applications via Ingress

Create a Pod/Service/Ingress deployment

Lets make one to resolve to an nginx container 😀

```yaml
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Pod
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-deployment
  ports:
  - port: 80
    name: nginx-http
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.exampledomain.lab
    http:
      paths:
      - backend:
          serviceName: nginx-service
          servicePort: 80
EOF
```

DNS and Nginx will pass down `nginx.exampledomain.lab` as the HOST header to Ingress

When this Ingress sees `nginx.exampledomain.lab` it will resolve the application

No need for finding out what nodeport kubernetes gave to deployed pods 👍
