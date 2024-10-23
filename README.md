# wg-punchhole

**Route traffic via gateway host into your cluster.**


Use another host as a gateway to route traffic into your cluster. This is useful
if you have a host that is publicly accessible and you want to route traffic
into your cluster without exposing your cluster to the public internet or
forwarding ports on your home network.

## How it works

![](/assets/images/tldr.png)


Suppose you have a service inside your cluster which the reverse proxy resolves
to `svc0.mydomain.com` and a DNS record for this service that points to the
public IP of the host acting as the gateway. Then, the following happens:

0. The gateway host and the wg-punchhole pod are connected via a wireguard vpn tunnel with the gateway acting as the server.
1. clients sends a request to `svc0.mydomain.com`.
2. traffic arrives at the gateway host:
   - HAProxy(Gateway) listens for incoming traffic on port 80 and 443 (or any port).
   - HAProxy(Gateway) adds proxy protocol headers to the incoming traffic.
   - HAProxy(Gateway) forwards the traffic to the wireguard VPN ip of the wg-punchhole pod.
3. traffic arrives at the wg-punchhole via the wireguard client running inside the pod:
   - HAproxy(Cluster) listens for incoming traffic on port 80 and 443 (or any port) and accepts the proxy protocol headers.
   - HAproxy(Cluster) forwards the traffic to the reverse proxy (or any other service) inside the cluster with the proxy protocol headers preserved.
4. The reverse proxy inside the cluster routes the traffic to the appropriate service.


## Example Configuration

### Gateway Host
The following packages are required:
- wireguard
- haproxy

The following ports need to be open:
- 51820/udp (wireguard)
- 80/tcp (haproxy)
- 443/tcp (haproxy)
- (any other port you want to route traffic to)


##### Wireguard Configuration
```conf
[Interface]
# servers ip in VPN network
Address = 10.1.10.1
PrivateKey = <privkey>
# port to open for inbound connections
ListenPort = 51820

# Peer = cluster
[Peer]
# public key of the cluster client
PublicKey = <pubkey>
# the IP and mask the client should be assigned
AllowedIPs = 10.1.10.2/32
```
##### HAproxy Configuration
```conf
# ...
# bind to the public ip of the host
frontend http
    bind 0.0.0.0:80
    mode tcp
    default_backend backend-http

frontend https
    bind 0.0.0.0:443
    mode tcp
    default_backend backend-https

backend backend-http
    mode tcp
    # vpn ip of peer (cluster) and add proxy protocol headers
    server clustervpn 10.1.10.2:80 send-proxy-v2

backend backend-https
    mode tcp
    # vpn ip of peer (cluster) and add proxy protocol headers
    server clustervpn 10.1.10.2:443 send-proxy-v2
```

### Cluster
Install this helm chart on your cluster and configure it using the following:

#### Wireguard config
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wireguard
  namespace: some-namespace
type: Opaque
stringData:
  wg0.conf: |
    [Interface]
    # ip of client in vpn network
    Address = 10.1.10.2
    PrivateKey = <privkey>

    # Peer = Gateway
    [Peer]
    # public key of the gateway
    PublicKey = <pubkey>
    # this can be the public ip or domain pointing of the gateway
    Endpoint = mydomain.com:51820
    # allow entire subnet
    AllowedIPs = 10.1.10.2/24
    # keep the connection alive
    PersistentKeepalive = 25
```

#### HAproxy config
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config
  namespace: dmz
data:
  haproxy.cfg: |

    frontend vpn_frontend_http
        # accept proxy protocol headers ("passthrough")
        bind 10.1.10.2:80 accept-proxy
        mode tcp
        default_backend cluster_backend_http

    frontend vpn_frontend_https
        # accept proxy protocol headers ("passthrough")
        bind 10.1.10.2:443 accept-proxy
        mode tcp
        default_backend cluster_backend_https

    backend cluster_backend_http
        mode tcp
        # Forward traffic to traefik via cluster dns with proxy headers
        server cluster_dns traefik-public.dmz.svc.cluster.local:80 send-proxy-v2

    backend cluster_backend_https
        mode tcp
        # Forward traffic to traefik via cluster dns with proxy headers
        server cluster_dns traefik-public.dmz.svc.cluster.local:443 send-proxy-v2
```

#### Reverse Proxy
Make sure to accept proxy protocol headers in your reverse proxy configuration
from the IP of the wg-punchhole pod or the internal Kubernetes subnet.
