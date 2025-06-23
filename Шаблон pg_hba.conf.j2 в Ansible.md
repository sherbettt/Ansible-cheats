
```bash
# TYPE  DATABASE  USER      ADDRESS           METHOD
local   all       all                         peer
host    all       all       127.0.0.1/32      scram-sha-256
host    all       all       ::1/128           scram-sha-256
#host    all       all       192.168.56.0/24   scram-sha-256  # Allow LAN


{% for ip in ['192.168.56.2/32' , '192.168.56.3/32'] %}
host    all       all       {{ip}}            md5
{% endfor %}

```
