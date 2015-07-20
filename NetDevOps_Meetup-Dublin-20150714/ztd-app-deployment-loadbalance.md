footer: _Vicente De Luca© 2015_ - vdeluca@zendesk.com
slidenumbers: true

#[FIT]ZER0 TOUCH
#[FIT]App Deployment
#[FIT]and Load Balancing

---
# whoami
## Vicente De Luca
![right](yo2.jpg)

_Network Engineer @ Zendesk_
- Unix since 90's
- IPv6 addicted
- RC flying (and crashing) passionate

http://nethero.org
@zenvdeluca

---

#[FIT] Problems
# Traditional App Deploy - NetOps
![](http://f.fastcompany.net/multisite_files/fastcompany/poster/2014/09/3035544-poster-p-1-gaming-your-way-to-better-problem-solving-at-work.jpg)

---

# What was wrong? 

- Ticket asking LB resources
- Multiple teams
- Complex and manual process
- Prone to human errors
- Painful on growing companies

---

#[FIT] Solution
![](http://thoughtstreamconsulting.com/everett/wp-content/uploads/2013/11/Solution.jpg)

---

# How to solve? 

- End to end automation
- Pain less migration
- Known tool
- Not increasing architecture complexity

---
#[FIT]HOW? 
#[FIT]_(NetOps tasks)_
![](http://baxian.ch/wp-content/uploads/2014/01/IT-Solutions_27197562.jpg)

---
# Ingredients

- Load Balancer :+1:
- Service Discovery :+1:
- Template Config Render :+1:

---
# Load Balancer

_What we already had_
F5 BIG-IP Viprion 2400 (http://f5.com/)

_Why we use it?_
ASIC Routing / Linux / Partition

_Alternatives?_
Haproxy / Keepalived

---

# Service Discovery
_Consul_
(http://consul.io/)

_Why we use it?_
Service catalog / Distributed Health Checks
query via REST or DNS / Agents are clients

_Alternatives?_
etcd / zookeeper, but just use consul :)

---

# Consul
## How we use it!

- Server register itself to Consul local datacenter cluster
- Publish own services with health checks
- Maintain a distributed key/value store
- lightweight binary that runs inside BIG-IP

---

# Template Config Rendering
_What we used?_
consul-template (https://github.com/hashicorp/consul-template)

_How we use it!_
- lightweight binary that runs inside BIG-IP
- Subscribes to Consul services / hc / kv changes
- Generate new configuration at change
- reload load balancer with new config


---
# Service Registration
## Example
```json
{
    "service": {
        "name": "http",
        "tags": ["primary"],
        "port":80,
        "check": {
                "id": "http_check",
                "name": "HTTP Health Check",
   "script": "curl -H 'Host=www.mydomain.com' http://localhost",
         "interval": "5s"
        }
    }
}
```

---

# Service Query
## Example

_DNS_
```
$ host http.service.consul
http.service.consul has address 10.0.1.10
```

---

# Service Query
## Example

_HTTP API_

```json
$ curl localhost:8500/v1/catalog/service/http?pretty

    {
        "Node": "srv1.nethero.org",
        "Address": "10.0.1.10",
        "ServiceName": "http",
        "ServiceTags": [
            "primary"
        ],
        "ServiceAddress": "10.0.1.10",
        "ServicePort": 80
    }
```

---
# Consul Template
## Config Example

```
consul = "127.0.0.1:8500"
retry = "10s"
max_stale = "10m"
log_level = "warn"
pid_file = "/var/run/consul-template.pid"
syslog {
  enabled = true
  facility = "user"
}
template {
  source = "/etc/consul-templates/production.ctmpl"
  destination = "/config/partitions/CONSUL/rendered-config"
  command = "/sbin/zconsul_postrender CONSUL"
}
```
---

#BIG-IP Consul Template (Golang)
```go
{{ range services }}{{ with $serviceMap := .}}
{{if and (.Tags.Contains "load_balance") }}
{{ range service $serviceMap.Name }}
{{if and (.Tags.Contains "load_balance") }}
  ltm node {{ .Node }} {
    address {{ .Address }}
  }

  ltm pool POOL-{{ $serviceMap.Name | toUpper }} {
    members {  {{range service $serviceMap.Name }}{{if and (.Tags.Contains "load_balance") }}
        {{.Node}}:{{.Port}} {
          address {{.Address}}
        } {{end}}{{end}}
    }
    monitor /Common/tcp
  }

{{ range service $serviceMap.Name }}{{if (.Tags.Contains "vip") }}
  ltm virtual VS-{{ $serviceMap.Name | toUpper }}-VIP-{{.Port}} {
    destination {{.Address}}:{{.Port}}
    ip-protocol tcp
    mask 255.255.255.255
    pool POOL-{{ $serviceMap.Name | toUpper }}
    profiles {
        fastL4 { }
    }
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
  }
{{ end }}{{end}}{{end}}{{end}}{{end}}{{end}}{{end}}
```
---

#BIG-IP Rendered Config example

```ruby
ltm node /CONSUL/docker3.aws1.nethero.org {
    address 10.0.100.6
}
ltm node /CONSUL/docker4.aws1.nethero.org {
    address 10.0.100.7
}
ltm pool /CONSUL/POOL-VOLUNTEER-WEB {
    members {
        /CONSUL/docker3.aws1.nethero.org:58358 {
            address 10.0.100.6
        }
		/CONSUL/docker4.aws1.nethero.org:58358 {
            address 10.0.100.7
        }

    }
    monitor /Common/tcp
}
ltm virtual /CONSUL/VS-VOLUNTEER-WEB-VIP-9080 {
    destination /CONSUL/10.0.250.10:9080
    ip-protocol tcp
    mask 255.255.255.255
    pool /CONSUL/POOL-VOLUNTEER-WEB
    profiles {
        /Common/fastL4 { }
    }
    source 0.0.0.0/0
    source-address-translation {
        type automap
    }
    translate-address enabled
    translate-port enabled
}

```

---

# BIG-IP: /var/log/user.log

```bash
Thu Jul  9 20:50:04 PDT 2015 - [CONSUL] config sync started
Loading configuration... /config/partitions/CONSUL/bigip.conf
Saving running configuration... /config/partitions/CONSUL/bigip.conf
Thu Jul  9 20:50:06 PDT 2015 - [CONSUL] Virtual Servers loaded from Consul app catalog:
/CONSUL/VS-ALPHA-WEB-VIP-6090 (/CONSUL/10.0.250.10:6090)
/CONSUL/VS-LEGION-WEB-VIP-9033 (/CONSUL/10.0.250.10:9033)
/CONSUL/VS-VOLUNTEER-WEB-VIP-9080 (/CONSUL/10.0.250.10:glrpc)
/CONSUL/VS-XPTO-WEB-VIP-6080 (/CONSUL/10.0.250.10:6080)
Thu Jul  9 20:50:06 PDT 2015 - [CONSUL] import completed
```

```bash
$ curl 10.0.250.10:6080
Hello world%
```

---

![center](zendesk-lb.jpg)

---

# TL;DR

- Consul for Service Discovery
- consul-template to render config and reload BIG-IP partition
- F5 BIG-IP as our Load Balancer choice
- new partition for each consul template instance

---

#[FIT]QUESTIONS?

---
#[FIT]THANKS:+1:
