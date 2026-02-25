Airlanggayudhoyono@Intel-Mil.Info
https://id.wikisource.org/wiki/Kitab_Undang-Undang_Hukum_Pidana/Buku_Kedua
BUKU KEDUA
KEJAHATAN




BAB I
KEJAHATAN TERHADAP KEAMANAN NEGARA

Pasal 104
Makar dengan maksud untuk membunuh, atau merampas kemerdekaan, atau meniadakan kemampuan Presiden atau Wakil Presiden memerintah, diancam dengan pidana mati atau pidana penjara seumur hidup atau pidana penjara sementara paling lama dua puluh tahun.

Pasal 105
Pasal ini ditiadakan berdasarkan Undang-undang No. 1. Tahun 1946, pasal VIII, butir 13.

Pasal 106
Makar dengan maksud supaya seluruh atau sebagian wilayah negara jatuh ke tangan musuh atau memisahkan sebagian dan wilayah negara, diancam dengan pidana penjara seumur hidup atau pidana penjara sementara paling lama dua puluh tahun.

Pasal 107






Home
Blog
Cloud Native
Understanding the Sidecar Injection, Traffic Intercepting & Routing Process in Istio
Understanding the Sidecar Injection, Traffic Intercepting & Routing Process in Istio
Learn the sidecar pattern, transparent traffic intercepting and routing in Istio.

May 12, 2022
Cloud Native
17 Minute
Updated This Year

Sidecar
Sidecar pattern
Microservices
Advantages of using the Sidecar pattern
Service Mesh
iptables manipulation analysis
Container
Init Container
Namespace
The traffic routing process explained
Understand Inbound Handler
Understand Outbound Handler
Summary
Problems with using iptables for traffic intercepting
Transparent intercepting optimization
eBPF
References
Based on Istio version 1.13, this article will present the following.

What is the sidecar pattern and what advantages does it have?
How are the sidecar injections done in Istio?
How does the sidecar proxy do transparent traffic intercepting?
How is the traffic routed to upstream?
The figure below shows how the productpage service requests access to http://reviews.default.svc.cluster.local:9080/ and how the sidecar proxy inside the reviews service does traffic blocking and routing forwarding when traffic goes inside the reviews service.

Figure 1: Istio transparent traffic intercepting and traffic routing diagram
Figure 1: Istio transparent traffic intercepting and traffic routing diagram
At the beginning of the first step, the sidecar in the productpage pod has selected a pod of the reviews service to be requested via EDS, knows its IP address, and sends a TCP connection request.

There are three versions of the reviews service, each with an instance, and the sidecar work steps in the three versions are similar, as illustrated below only by the sidecar traffic forwarding step in one of the Pods.

Sidecar pattern
Dividing the functionality of an application into separate processes running in the same minimal scheduling unit (e.g. Pod in Kubernetes) can be considered sidecar mode. As shown in the figure below, the sidecar pattern allows you to add more features next to your application without additional third-party component configuration or modifications to the application code.

Figure 2: Sidecar pattern
Figure 2: Sidecar pattern
The Sidecar application is loosely coupled to the main application. It can shield the differences between different programming languages and unify the functions of microservices such as observability, monitoring, logging, configuration, circuit breaker, etc.

Advantages of using the Sidecar pattern
When deploying a service mesh using the sidecar model, there is no need to run an agent on the node, but multiple copies of the same sidecar will run in the cluster. In the sidecar deployment model, a companion container (such as Envoy or MOSN) is deployed next to each application’s container, which is called a sidecar container. The sidecar takes overall traffic in and out of the application container. In Kubernetes’ Pod, a sidecar container is injected next to the original application container, and the two containers share storage, networking, and other resources.

Due to its unique deployment architecture, the sidecar model offers the following advantages.

Abstracting functions unrelated to application business logic into a common infrastructure reduces the complexity of microservice code.
Reduce code duplication in microservices architectures because it is no longer necessary to write the same third-party component profiles and code.
The sidecar can be independently upgraded to reduce the coupling of application code to the underlying platform.
iptables manipulation analysis
In order to view the iptables configuration, we need to nsenter the sidecar container using the root user to view it, because kubectl cannot use privileged mode to remotely manipulate the docker container, so we need to log on to the host where the productpage pod is located.

If you use Kubernetes deployed by minikube, you can log directly into the minikube’s virtual machine and switch to root. View the iptables configuration that lists all the rules for the NAT (Network Address Translation) table because the mode for redirecting inbound traffic to the sidecar is REDIRECT in the parameters passed to the istio-iptables when the Init container is selected for the startup, so there will only be NAT table specifications in the iptables and mangle table configurations if TPROXY is selected. See the iptables command for detailed usage.

We only look at the iptables rules related to productpage below.

# login to minikube, change user to root
$ minikube ssh
$ sudo -i

# See the processes in the productpage pod's istio-proxy container
$ docker top `docker ps|grep "istio-proxy_productpage"|cut -d " " -f1`
UID PID PPID C STIME TTY TIME CMD
1337 10576 10517 0 08:09 ? 00:00:07 /usr/local/bin/pilot-agent proxy sidecar --domain default.svc.cluster.local --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster productpage.default --drainDuration 45s --parentShutdownDuration 1m0s --discoveryAddress istiod.istio-system.svc:15012 --zipkinAddress zipkin.istio-system:9411 --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --connectTimeout 10s --proxyAdminPort 15000 --concurrency 2 --controlPlaneAuthPolicy NONE --dnsRefreshRate 300s --statusPort 15020 --trust-domain=cluster.local --controlPlaneBootstrap=false
1337 10660 10576 0 08:09 ? 00:00:33 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster productpage.default --service-node sidecar~172.17.0.16~productpage-v1-7f44c4d57c-ksf9b.default~default.svc.cluster.local --max-obj-name-len 189 --local-address-ip-version v4 --log-format [Envoy (Epoch 0)] [%Y-%m-%d %T.%e][%t][%l][%n] %v -l warning --component-log-level misc:error --concurrency 2

# Enter the nsenter into the namespace of the sidecar container (any of the above is ok)
$ nsenter -n --target 10660

View the process’s iptables rule chain under its namespace.

# View the details of the rule configuration in the NAT table.
$ iptables -t nat -L -v
# PREROUTING chain: Used for Destination Address Translation (DNAT) to jump all incoming TCP traffic to the ISTIO_INBOUND chain.
Chain PREROUTING (policy ACCEPT 2701 packets, 162K bytes)
 pkts bytes target prot opt in out source destination
 2701 162K ISTIO_INBOUND tcp -- any any anywhere anywhere

# INPUT chain: Processes incoming packets and non-TCP traffic will continue on the OUTPUT chain.
Chain INPUT (policy ACCEPT 2701 packets, 162K bytes)
 pkts bytes target prot opt in out source destination

# OUTPUT chain: jumps all outbound packets to the ISTIO_OUTPUT chain.
Chain OUTPUT (policy ACCEPT 79 packets, 6761 bytes)
 pkts bytes target prot opt in out source destination
   15 900 ISTIO_OUTPUT tcp -- any any anywhere anywhere

# POSTROUTING CHAIN: All packets must first enter the POSTROUTING chain when they leave the network card, and the kernel determines whether they need to be forwarded out according to the packet destination.
Chain POSTROUTING (policy ACCEPT 79 packets, 6761 bytes)
 pkts bytes target prot opt in out source destination

# ISTIO_INBOUND CHAIN: Redirects all inbound traffic to the ISTIO_IN_REDIRECT chain, except for traffic destined for ports 15090 (used by Prometheus) and 15020 (used by Ingress gateway for Pilot health checks), and traffic sent to these two ports will return to the call point of the iptables rule chain, the successor POSTROUTING to the INPUT chain.
Chain ISTIO_INBOUND (1 references)
 pkts bytes target prot opt in out source destination
    0 0 RETURN tcp -- any any anywhere anywhere tcp dpt:ssh
    2 120 RETURN tcp -- any any anywhere anywhere tcp dpt:15090
 2699 162K RETURN tcp -- any any anywhere anywhere tcp dpt:15020
    0 0 ISTIO_IN_REDIRECT tcp -- any any anywhere anywhere

# ISTIO_IN_REDIRECT chain: jumps all inbound traffic to the local 15006 port, thus successfully blocking traffic to the sidecar.
Chain ISTIO_IN_REDIRECT (3 references)
 pkts bytes target prot opt in out source destination
    0 0 REDIRECT tcp -- any any anywhere anywhere redir ports 15006

# ISTIO_OUTPUT chain: see the details bellow
Chain ISTIO_OUTPUT (1 references)
 pkts bytes target prot opt in out source destination
    0 0 RETURN all -- any lo 127.0.0.6 anywhere
    0 0 ISTIO_IN_REDIRECT all -- any lo anywhere !localhost owner UID match 1337
    0 0 RETURN all -- any lo anywhere anywhere ! owner UID match 1337
   15 900 RETURN all -- any any anywhere anywhere owner UID match 1337
    0 0 ISTIO_IN_REDIRECT all -- any lo anywhere !localhost owner GID match 1337
    0 0 RETURN all -- any lo anywhere anywhere ! owner GID match 1337
    0 0 RETURN all -- any any anywhere anywhere owner GID match 1337
    0 0 RETURN all -- any any anywhere localhost
    0 0 ISTIO_REDIRECT all -- any any anywhere anywhere

# ISTIO_REDIRECT chain: redirects all traffic to Sidecar (i.e. local) port 15001.
Chain ISTIO_REDIRECT (1 references)
 pkts bytes target prot opt in out source destination
    0 0 REDIRECT tcp -- any any anywhere anywhere redir ports 15001

The focus here is on the 9 rules in the ISTIO_OUTPUT chain. For ease of reading, I will show some of the above rules in the form of a table as follows.

Rule target in out source destination
1 RETURN any lo 127.0.0.6 anywhere
2 ISTIO_IN_REDIRECT any lo anywhere !localhost owner UID match 1337
3 RETURN any lo anywhere anywhere !owner UID match 1337
4 RETURN any any anywhere anywhere owner UID match 1337
5 ISTIO_IN_REDIRECT any lo anywhere !localhost owner GID match 1337
6 RETURN any lo anywhere anywhere !owner GID match 1337
7 RETURN any any anywhere anywhere owner GID match 1337
8 RETURN any any anywhere localhost
9 ISTIO_REDIRECT any any anywhere anywhere
The following diagram shows the detailed flow of the ISTIO_ROUTE rule.

Figure 3: ISTIO_ROUTE iptables rules
Figure 3: ISTIO_ROUTE iptables rules
I will explain the purpose of each rule, corresponding to the steps and details in the illustration at the beginning of the article, in the order in which they appear. Where rules 5, 6, and 7 are extensions of the application of rules 2, 3, and 4 respectively (from UID to GID), which serve similar purposes and will be explained together. Note that the rules therein are executed in order, meaning that the rule with the next highest order will be used as the default. When the outbound NIC (out) is lo (local loopback address, loopback interface), it means that the destination of the traffic is the local Pod, and traffic sent from the Pod to the outside, will not go through this interface. Only rules 4, 7, 8, and 9 apply to all outbound traffic from the review Pod.

Rule 1

Purpose: To pass through traffic sent by the Envoy proxy to the local application container, so that it bypasses the Envoy proxy and goes directly to the application container.
Corresponds to steps 6 through 7 in the illustration.
Details: This rule causes all requests from 127.0.0.6 (this IP address will be explained below) to jump out of the chain, return to the point of invocation of iptables (i.e. OUTPUT) and continue with the rest of the routing rules, i.e. the POSTROUTING rule, which sends traffic to an arbitrary destination, such as the application container within the local Pod. Without this rule, traffic from the Envoy proxy within the Pod to the Pod container will execute the next rule, rule 2, and the traffic will enter the Inbound Handler again, creating a dead loop. Putting this rule in the first place can avoid the problem of traffic dead-ending in the Inbound Handler.
Rule 2, 5

Purpose: Handle inbound traffic (traffic inside the Pod) from the Envoy proxy, but not requests to the localhost, and forward it to the Envoy proxy’s Inbound Handler via a subsequent rule. This rule applies to scenarios where the Pod invokes its own IP address, i.e., traffic between services within the Pod.
Details: If the destination of the traffic is not localhost and the packet is sent by 1337 UID (i.e. istio-proxy user, Envoy proxy), the traffic will be forwarded to Envoy’s Inbound Handler through ISTIO_IN_REDIRECT eventually.
Rule 3, 6

Purpose: To pass through the internal traffic of the application container within the Pod. This rule applies to traffic within the container. For example, access to Pod IP or localhost within a Pod.
Corresponds to steps 6 through 7 in the illustration.
Details: If the traffic is not sent by an Envoy user, then jump out of the chain and return to OUTPUT to call POSTROUTING and go straight to the destination.
Rule 4, 7

Purpose: To pass through outbound requests sent by Envoy proxy.
Corresponds to steps 14 through 15 in the illustration.
Details: If the request was made by the Envoy proxy, return OUTPUT to continue invoking the POSTROUTING rule and eventually access the destination directly.
Rule 8

Purpose: Passes requests from within the Pod to the localhost.
Details: If the destination of the request is localhost, return OUTPUT and call POSTROUTING to access localhost directly.
Rule 9

Purpose: All other traffic will be forwarded to ISTIO_REDIRECT after finally reaching the Outbound Handler of Envoy proxy.
Corresponds to steps 10 through 11 in the illustration.
The above rule avoids dead loops in the iptables rules for Envoy proxy to application routing, and guarantees that traffic can be routed correctly to the Envoy proxy, and that real outbound requests can be made.

About RETURN target

You may notice that there are many RETURN targets in the above rules, which means that when this rule is specified, it jumps out of the rule chain, returns to the call point of iptables (in our case OUTPUT) and continues to execute the rest of the routing rules, in our case the POSTROUTING rule, which sends traffic to any destination address, you can think of This is intuitively understood as pass-through.

About the 127.0.0.6 IP address

The IP 127.0.0.6 is the default InboundPassthroughClusterIpv4 in Istio and is specified in the code of Istio. This is the IP address to which traffic is bound after entering the Envoy proxy, and serves to allow Outbound traffic to be re-sent to the application container in the Pod, i.e. Passthought, bypassing the Outbound Handler. this traffic is access to the Pod itself, and not real outbound traffic. See Istio Issue-29603 for more information on why this IP was chosen as the traffic passthrough.

The traffic routing process explained
Traffic routing is divided into two processes, Inbound and Outbound, which will be analyzed in detail for the reader below based on the example above and the configuration of the sidecar.

Understand Inbound Handler
The role of the Inbound handler is to pass traffic from the downstream blocked by iptables to the localhost and establish a connection to the application container within the Pod. Assuming the name of one of the Pods is reviews-v1-545db77b95-jkgv2, run istioctl proxy-config listener reviews-v1-545db77b95-jkgv2 --port 15006 to see which Listener is in that Pod.

ADDRESS PORT MATCH DESTINATION
0.0.0.0 15006 Addr: *:15006 Non-HTTP/Non-TCP
0.0.0.0 15006 Trans: tls; App: istio-http/1.0,istio-http/1.1,istio-h2; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; App: http/1.1,h2c; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: TCP TLS; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: raw_buffer; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; Addr: 0.0.0.0/0 InboundPassthroughClusterIpv4
0.0.0.0 15006 Trans: tls; App: istio,istio-peer-exchange,istio-http/1.0,istio-http/1.1,istio-h2; Addr: *:9080 Cluster: inbound|9080||
0.0.0.0 15006 Trans: raw_buffer; Addr: *:9080 Cluster: inbound|9080||

The following lists the meanings of the fields in the above output.

ADDRESS: downstream address
PORT: The port the Envoy listener is listening on
MATCH: The transport protocol used by the request or the matching downstream address
DESTINATION: Route destination
The Iptables in the reviews Pod intercept inbound traffic to port 15006, and from the above output we can see that Envoy’s Inbound Handler is listening on port 15006, and requests to port 9080 destined for any IP will be routed to the inbound|9080|| Cluster.

As you can see in the last two rows of the Pod’s Listener list, the Listener for 0.0.0.0:15006/TCP (whose actual name is virtualInbound) listens for all Inbound traffic, which contains matching rules, and traffic to port 9080 from any IP will be routed. If you want to see the detailed configuration of this Listener in Json format, you can execute the istioctl proxy-config listeners reviews-v1-545db77b95-jkgv2 --port 15006 -o json command. You will get an output similar to the following.

[
    /*omit*/
    {
        "name": "virtualInbound",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 15006
            }
        },
        "filterChains": [
            /*omit*/
            {
                "filterChainMatch": {
                    "destinationPort": 9080,
                    "transportProtocol": "tls",
                    "applicationProtocols": [
                        "istio",
                        "istio-peer-exchange",
                        "istio-http/1.0",
                        "istio-http/1.1",
                        "istio-h2"
                    ]
                },
                "filters": [
                    /*omit*/
                    {
                        "name": "envoy.filters.network.http_connection_manager",
                        "typedConfig": {
                            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                            "statPrefix": "inbound_0.0.0.0_9080",
                            "routeConfig": {
                                "name": "inbound|9080||",
                                "virtualHosts": [
                                    {
                                        "name": "inbound|http|9080",
                                        "domains": [
                                            "*"
                                        ],
                                        "routes": [
                                            {
                                                "name": "default",
                                                "match": {
                                                    "prefix": "/"
                                                },
                                                "route": {
                                                    "cluster": "inbound|9080||",
                                                    "timeout": "0s",
                                                    "maxStreamDuration": {
                                                        "maxStreamDuration": "0s",
                                                        "grpcTimeoutHeaderMax": "0s"
                                                    }
                                                },
                                                "decorator": {
                                                    "operation": "reviews.default.svc.cluster.local:9080/*"
                                                }
                                            }
                                        ]
                                    }
                                ],
                                "validateClusters": false
                            },
                            /*omit*/
                        }
                    }
                ],
            /*omit*/
        ],
        "listenerFilters": [
        /*omit*/
        ],
        "listenerFiltersTimeout": "0s",
        "continueOnListenerFiltersTimeout": true,
        "trafficDirection": "INBOUND"
    }
]

Since the Inbound Handler traffic routes traffic from any address to this Pod port 9080 to the inbound|9080|| Cluster, let’s run istioctl pc cluster reviews-v1-545db77b95-jkgv2 --port 9080 --direction inbound -o json to see the Cluster configuration and you will get something like the following output.

[
    {
        "name": "inbound|9080||",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "cleanupInterval": "60s",
        "upstreamBindConfig": {
            "sourceAddress": {
                "address": "127.0.0.6",
                "portValue": 0
            }
        },
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "services": [
                        {
                            "host": "reviews.default.svc.cluster.local",
                            "name": "reviews",
                            "namespace": "default"
                        }
                    ]
                }
            }
        }
    }
]

We see that the TYPE is ORIGINAL_DST, which sends the traffic to the original destination address (Pod IP), because the original destination address is the current Pod, you should also notice that the value of upstreamBindConfig.sourceAddress.address is rewritten to 127.0.0.6, and for Pod This echoes the first rule in the iptables ISTIO_OUTPUT the chain above, according to which traffic will be passed through to the application container inside the Pod.

Understand Outbound Handler
Because reviews send an HTTP request to the ratings service at http://ratings.default.svc.cluster.local:9080/, the role of the Outbound handler is to intercept traffic from the local application to which iptables has intercepted, and determine how to route it to the upstream via the sidecar.

Requests from application containers are Outbound traffic, intercepted by iptables and transferred to the Outbound handler for processing, which then passes through the virtualOutbound Listener, the 0.0.0.0_9080 Listener, and then finds the upstream cluster via Route 9080, which in turn finds the Endpoint via EDS to perform the routing action.

Route ratings.default.svc.cluster.local:9080

reviews requests the ratings service and runs istioctl proxy-config routes reviews-v1-545db77b95-jkgv2 --name 9080 -o json. View the route configuration because the sidecar matches VirtualHost based on domains in the HTTP header, so only ratings.default.svc.cluster.local:9080 is listed below for this VirtualHost.

[{
  {
      "name": "ratings.default.svc.cluster.local:9080",
      "domains": [
          "ratings.default.svc.cluster.local",
          "ratings.default.svc.cluster.local:9080",
          "ratings",
          "ratings:9080",
          "ratings.default.svc.cluster",
          "ratings.default.svc.cluster:9080",
          "ratings.default.svc",
          "ratings.default.svc:9080",
          "ratings.default",
          "ratings.default:9080",
          "10.98.49.62",
          "10.98.49.62:9080"
      ],
      "routes": [
          {
              "name": "default",
              "match": {
                  "prefix": "/"
              },
              "route": {
                  "cluster": "outbound|9080||ratings.default.svc.cluster.local",
                  "timeout": "0s",
                  "retryPolicy": {
                      "retryOn": "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
                      "numRetries": 2,
                      "retryHostPredicate": [
                          {
                              "name": "envoy.retry_host_predicates.previous_hosts"
                          }
                      ],
                      "hostSelectionRetryMaxAttempts": "5",
                      "retriableStatusCodes": [
                          503
                      ]
                  },
                  "maxGrpcTimeout": "0s"
              },
              "decorator": {
                  "operation": "ratings.default.svc.cluster.local:9080/*"
              }
          }
      ]
  },
..]

From this VirtualHost configuration, you can see routing traffic to the cluster outbound|9080||ratings.default.svc.cluster.local.

Endpoint outbound|9080||ratings.default.svc.cluster.local

Running istioctl proxy-config endpoint reviews-v1-545db77b95-jkgv2 --port 9080 -o json --cluster "outbound|9080||ratings.default.svc.cluster.local" to view the Endpoint configuration, the results are as follows.

{
  "clusterName": "outbound|9080||ratings.default.svc.cluster.local",
  "endpoints": [
    {
      "locality": {

      },
      "lbEndpoints": [
        {
          "endpoint": {
            "address": {
              "socketAddress": {
                "address": "172.33.100.2",
                "portValue": 9080
              }
            }
          },
          "metadata": {
            "filterMetadata": {
              "istio": {
                  "uid": "kubernetes://ratings-v1-8558d4458d-ns6lk.default"
                }
            }
          }
        }
      ]
    }
  ]
}

We see that the endpoint address is 10.4.1.12. In fact, the Endpoint can be one or more, and the sidecar will select the appropriate Endpoint to route based on certain rules. At this point the review Pod has found the Endpoint for its upstream service rating.

Summary
This article uses the bookinfo example provided by Istio to guide readers through the implementation details behind the sidecar injection, iptables transparent traffic intercepting, and traffic routing in the sidecar. The sidecar mode and traffic transparent intercepting are the features and basic functions of Istio service mesh, understanding the process behind this function and the implementation details will help you understand the principle of service mesh and the content in the later chapters of the Istio Handbook, so I hope readers can try it from scratch in their own environment to deepen their understanding.

Using iptables for traffic intercepting is just one of the ways to do traffic intercepting in the data plane of a service mesh, and there are many more traffic intercepting scenarios, quoted below from the description of the traffic intercepting section given in the MOSN official network of the cloud-native network proxy.

Problems with using iptables for traffic intercepting
Currently, Istio uses iptables for transparent intercepting and there are three main problems.

The need to use the conntrack module for connection tracking, in the case of a large number of connections, will cause a large consumption and may cause the track table to be full, in order to avoid this problem, the industry has a practice of closing conntrack.
iptables is a common module with global effect and cannot explicitly prohibit associated changes, which is less controllable.
iptables redirect traffic is essentially exchanging data via a loopback. The outbound traffic will traverse the protocol stack twice and lose forwarding performance in a large concurrency scenario.
Several of the above problems are not present in all scenarios, let’s say some scenarios where the number of connections is not large and the NAT table is not used, iptables is a simple solution that meets the requirements. In order to adapt to a wider range of scenarios, transparent intercepting needs to address all three of these issues.

Transparent intercepting optimization
In order to optimize the performance of transparent traffic intercepting in Istio, the following solutions have been proposed by the industry.

Traffic intercepting with eBPF using the Merbridge Open Source Project

Merbridge is a plug-in that leverages eBPF to accelerate the Istio service mesh, which was open sourced by DaoCloud in early 2022. Using Merbridge can optimize network performance in the data plane to some extent.

Merbridge leverages the sockops and redir capabilities of eBPF to transfer packets directly from inbound sockets to outbound sockets. eBPF provides the bpf_msg_redirect_hash function to forward application packets directly.

Handling inbound traffic with tproxy

tproxy can be used for redirection of inbound traffic without changing the destination IP/port in the packet, without performing connection tracking, and without the problem of conntrack modules creating a large number of connections. Restricted to the kernel version, tproxy’s application to outbound is flawed. Istio currently supports handling inbound traffic via tproxy.

Use hook connect to handle outbound traffic

In order to adapt to more application scenarios, the outbound direction is implemented by hook connect, which is implemented as follows.

Figure 4: Hook Connect Diagram
Figure 4: Hook Connect Diagram
Whichever transparent intercepting scheme is used, the problem of obtaining the real destination IP/port needs to be solved, using the iptables scheme through getsockopt, tproxy can read the destination address directly, by modifying the call interface, hook connect scheme reads in a similar way to tproxy.

After the transparent intercepting, the sockmap can shorten the packet traversal path and improve forwarding performance in the outbound direction, provided that the kernel version meets the requirements (4.16 and above).

References
Debugging Envoy and Istiod - istio.io
Demystifying Istio’s Sidecar Injection Model - istio.io
The traffic intercepting solution when MOSN is used as a sidecar - mosn.io
Jimmy Song
Jimmy Song
Focusing on research and open source practices in AI-Native Infrastructure and cloud native application architecture.

 Updated on Aug 24, 2025
Envoy Proxy
IPtables
Istio

Post Navigation

Previous Post
Why Would You Need Spire for Authentication With Istio?
Next Post
Istio Data Plane Pod Startup Process Explained
Related Articles

Understanding IPTables
2022-05-12
Cloud Native
4min
Traffic Types and Iptables Rules in Istio Sidecar Explained
2022-05-07
Cloud Native
4min
Istio Data Plane Pod Startup Process Explained
2022-05-12
Cloud Native
10min
Quick Links
AI OSS Landscape
AI Infra Brief
AI Native Infrastructure
HAMi Project
Jimmy
Contact
© 2017-2026 Jimmy Song All Right Reserved · CC BY 4.0



"FA's Linux Blog This blog is not about reinventing the wheel. I blog about Linux troubleshooting tips/solutions that I could not find elsewhere

JUL
31
Fixing pesky Windows line breaks in DOS format text file (dos2unix command)
While working with source code in git repositories I often come across source code with pesky Windows line breaks (aka DOS file format). Some *nix tools complains with files containing Windows like breaks. (The source code was most likely created initially by someone using the horrid Windows OS.)
 

To detect the file has Windows line endings (carriage return + line feed), use the cat command to detect (Windows line endings will contain the visible ^M at the end of the line)
  
$ cat -v file
This is line 1^M
This is line 2^M
This is line 3^M
This is line 4^M
This is line 5^M
$
 
The easiest option to get rid of the ^M is to use the dos2unix command. However, on your system you might not have it installed. The alternative option is to use the sed command which is installed by default on all Linux and macOS:
 
 
$ sed -i -e 's/\r//g' file
$
$ cat -v file
This is line 1
This is line 2
This is line 3
This is line 4
This is line 5

 
The ^M are now gone
 
 
Getting the dos file format back
 
For some reason if you want to get back the horrid Windows line breaks, use the unix2dos command. In case it is not available, use the trusty sed
  

$ cat -v file
This is line 1
This is line 2
This is line 3
This is line 4
This is line 5
$
$ sed -i -e 's/\r*$/\r/g' file
$
$ cat -v file
This is line 1^M
This is line 2^M
This is line 3^M
This is line 4^M
This is line 5^M
$  



Posted 31st July 2021 by Fazle Arefin
Labels: crlf dos2unix line feed unix2dos

0 Add a comment
JUN
21
Nikon Firmware Update from Linux
This is a simple guide to show how you can extract the firmware from the official source of Nikon using Linux. Nikon unfortunately does not provide any firmware update file which you can use from Linux.

Firmware can be downloaded from https://downloadcenter.nikonimglib.com/en/index.html depending on your Nikon model.

Once you are on the download page for your specific model, download the firmware for updating from Windows. It will be a exe file but is actually a self-extracting archive. Use the unrar command to extract the firmware as shown below for the Nikon D850 model.

unrar x  F-D850-V120W.exe




Afterwards, you can follow the instructions on Nikon website on how to complete the rest of the steps.

Posted 21st June 2021 by Fazle Arefin
Labels: firmware linux nikon

0 Add a comment
MAY
9
HTTP and HTTPS simple web server in bash or any shell using netcat
There are times we need to spin up a simple web server to test connectivity. Often times, we end up installing apache or nginx. If you want to skip installing any of those web servers and simply use your shell to spin up a very simple web server, this is how you do it.

Note that there are different version of netcat. I am using the netcat from nmap.org. In Ubuntu you can install it by sudo apt install ncat and the binary will be /usr/bin/ncat. The other versions may not have the ssl options.

HTTP Server
01. Run netcat in listening mode
echo -e "HTTP/1.1 200 OK\r\n\r\n<h1>hello world from $(hostname) on $(date)</h1>\r\n" |  ncat -vl -p 8080

02. Try to connect to it using your web browser or curl
curl -v -i http://127.0.0.1:8080

HTTPS Server
There is a --ssl option in netcat you can use! But make sure you are using the right version of netcat mentioned above.

01. Run netcat in listening mode
echo -e "HTTP/1.1 200 OK\r\n\r\n<h1>hello world from $(hostname) on $(date)</h1>\r\n" |  ncat -vl -p 8443 --ssl

You can also use the option --ssl-key to specify a private key and the --ssl-cert option to specify a certificate rather than netcat automatically generating one for you (demonstrated below)
02. Try to connect to it using your web browser or curl
curl -k -v -i https://127.0.0.1:8443




Making the server run continuously
# http
HTTP_RESP="HTTP/1.1 200 OK\r\ndate: $(date)\r\nserver: basher\r\n\r\n${2:-"OK"}\r\n"
 
while { echo -en "$HTTP_RESP"; } | ncat -lp "${1:-8080}"; do
  echo -e "***\n"
done

# https
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost' -nodes
 
HTTP_RESP="HTTP/1.1 200 OK\r\ndate: $(date)\r\nserver: basher\r\n\r\n${2:-"OK"}\r\n"
 
while { echo -en "$HTTP_RESP"; } | ncat --ssl --ssl-key ./key.pem --ssl-cert ./cert.pem -lp "${1:-8443}"; do
  echo -e "***\n"
done

Extending the functionality
It is possible to serve static files or even php/python etc scripts by executing them inside the echo string.

Posted 9th May 2021 by Fazle Arefin
Labels: apache bash http httss netcat nginx nmap shell ssl web server

0 Add a comment
JUL
11
Tunneling traffic over tor network using proxychains
Mirror Link

VPNs are great for hiding your IP address. However, you are at the mercy of the VPN provider to protect your identity. This is where tor comes in. You can route your traffic over the tor network without any costly VPN subscription. The tor network is decentralized and hence it is harder to track traffic over the tor network and hence it is better at protecting your identity. tor is not just for browsing the dark web.


Note that tor network can be very slow since the traffic is passing through 3 different nodes. It may not be ideal for hacking your geolocation to watch netflix content from another country.

Instructions below are for Ubuntu 20.04. You will need to do your homework for other distributions.
00. Install tor and make sure it is running
sudo apt install tor
sudo systemctl enable tor --now
systemctl status tor

Optionally, specify the exit node country/countries (RU is Russia, look up 2 letter ISO code for other countries)

echo 'ExitNodes {RU}' | sudo tee --append /etc/tor/torrc
sudo systemctl restart tor

01. Install proxychains4
sudo apt install proxychains4

Optionally, make proxychains4 use SOCKS5 protocol

sudo sed -i -E 's|^socks[0-9]?.*|socks5  127.0.0.1 9050|' /etc/proxychains4.conf
02. Let's see it in action
Check your location without going through tor network
vagrant@ubuntu-focal:~$ curl https://ipinfo.io/city
Warsaw
vagrant@ubuntu-focal:~$

I am from Warsaw, Poland(?)
Check your location going through tor network
vagrant@ubuntu-focal:~$ proxychains4 curl ipinfo.io/city
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.14
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  ipinfo.io:80  ...  OK
Moscow
vagrant@ubuntu-focal:~$

Now I am from Moscow, Russia!
ssh can be tunnelled through tor as well
vagrant@ubuntu-focal:~$ nc -vz github.com 22
Connection to github.com 22 port [tcp/ssh] succeeded!
vagrant@ubuntu-focal:~$

vagrant@ubuntu-focal:~$ proxychains nc -vz github.com 22
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.14
[proxychains] Strict chain  ...  127.0.0.1:9050  ...  github.com:22  ...  OK
Connection to github.com 22 port [tcp/ssh] succeeded!
vagrant@ubuntu-focal:~$
Make firefox (or google chrome) use the tor network
Close all open browser windows of firefox (or google chrome) first and then launch firefox (or google chrome) from the terminal

proxychains4 firefox
 
An easier approach is to use the FoxyProxy extension for Firefox. That way you won't have to close your existing browser session and relaunch.
 
After you have installed FoxyProxy, you need to use the following settings in FoxyProxy to route traffic through Tor:
 
Proxy Type: SOCKS5
Proxy IP address: 127.0.0.1
Port: 9050
Troubleshooting
If you get timed out when using proxychains, simply restart the tor service to create a new circuit or use a new country in /etc/tor/torrc and restart tor service

sudo systemctl restart tor
Disclaimer
All information here is for educational purpose.

Posted 11th July 2020 by Fazle Arefin
Labels: proxychains tor
0 Add a comment
JUL
5
Running Metasploit in Parrot OS docker container
Unlike the Kali Linux docker image, Parrot OS docker image comes with a more complete set of pentest tools. The downside is that Parrot OS docker image is quite big. Since the Parrot OS docker container does not run systemd, here is how you can run Metasploit with a PostgreSQL db connection:

# get the Parrot OS image if you have not already done so; other images can be found in https://github.com/ParrotSec/docker-images
docker pull parrotsec/security

# start the container
docker run -ti --rm --network host parrotsec/security

# start postgresql
service postgresql start

# initialize database
msfdb init

# finally msfconsole
msfconsole

# check connection to postgresql from msfconsole
msf5 > db_status
[*] Connected to msf. Connection type: postgresql.
msf5 >
Posted 5th July 2020 by Fazle Arefin
Labels: docker

0 Add a comment
FEB
22
Remove snap / snapd from Ubuntu
(Warning: Contains strong language. Do not read further if you are a minor or get offended easily)

I hate snaps on Ubuntu. They are a pain to load even on modern systems. Even a small utility like calculator installed as a snap takes several seconds to load. This is how you remove snaps from your Ubuntu system and make sure this pesky mofo does not get installed again.

sudo apt autopurge snap  # this will remove all installed snaps as well

rm -fr ~/snap

echo "Package: snapd
Pin: release a=*
Pin-Priority: -10" | sudo tee /etc/apt/preferences.d/nosnap.pref​

Posted 22nd February 2020 by Fazle Arefin
Labels: linux snapd snappy snaps ubuntu

0 Add a comment
JUN
15
Run docker commands in Linux without sudo
Having to use sudo for running docker and docker-compose can be frustrating. All you need to do is add yourself (your user) to the docker group. Here's what you need to do (tested on Ubuntu with docker installed from the official Ubuntu repos):

id  # check if you belong to docker group
sudo usermod -a -G docker $username
sudo grpck  # optional, just to check the integrity of group files
newgrp docker  # no need to log out and back in to get belong to the docker group
id  # check if you are now in docker group

Note that you have have other terminal sessions open you need run newgrp docker run in those terminals as well. Logging out and back in will fix your group permanently.


Posted 15th June 2019 by Fazle Arefin
Labels: docker linux root

0 Add a comment
APR
20
Secure Dual or Multi Disk RAID Installation of Ubuntu LTS 18.04
What is this guide about?
This guide is about installing Ubuntu on a system with 2 or more hard disks using RAID (software). You may have a system with 2 or more disks and want to take advantage of RAID0 for faster disk read/write or RAID1 better reliability. 

I have a desktop computer with 2 magnetic hard disks and I put them on RAID0 to get faster read/write performance. The RAID0 setup is not limited to magnetic hard disks but also applies to SSD.

Ubuntu installer disk for the desktop still do not come with advanced disk partitioning tools, and hence is the need for this guide.

This is a fairly advanced guide. If you struggle with disk partitioning concepts, it's better to stay away from this guide as it will just lead of frustration. 

This is my setup. There are many other different variations possible. This setup was done on a Ubuntu LTS 18.04 but may work on other versions of Ubuntu.
What will you need?
2 or more disks of similar disk space
Ubuntu ISO
Visual representation of the setup


Step 0: Pre-requisites
Insert Ubuntu installer disk and choose Try Ubuntu instead of installing it. And then open a terminal from the GUI desktop.

Following is for a 2 disk install. You can adjust the commands for other multi disk installs.

# install mdadm
apt-get install mdadm

# stop raid and remove raid metadata from disk, if any existing
mdadm --stop /dev/md0
mdadm --stop /dev/md1
for i in sd{a,b}{1,2}; do
  mdadm --zero-superblock /dev/$i
done

# partition the first disk
# delete any existing partitions
# create the first primary partition with 512MB
# create the second primary partition with the rest of the disk
fdisk /dev/sda

# copy the label to the second hdd
sfdisk -d /dev/sda | sfdisk /dev/sdb

# create the raids
mdadm --create /dev/md0 --level=1 --bitmap=internal --raid-devices=2 --metadata=0.90 /dev/sda1 /dev/sdb1
mdadm --create /dev/md1 --level=0 --raid-devices=2 /dev/sda2 /dev/sdb2

# create and open the encryption layer
cryptsetup luksFormat -v --cipher aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 5000 --use-random /dev/md1
cryptsetup luksOpen /dev/md1 pv0

#create LVM
pvcreate /dev/mapper/pv0
vgcreate vg0 /dev/mapper/pv0

# create the logical volumes
# adjust the numbers for your disk
lvcreate -nlvroot -L20G vg0
lvcreate -nlvhome -L80G vg0
lvcreate -nlvswap -L4G vg0

Step 1: Install Ubuntu
From the same terminal, launch the installer. The -b option is to skip the installer from installing GRUB.

ubiquity -b

When you get to the Disk Partitioning section, choose Something Else. In the partitioning,

/dev/md0 as ext4 and /boot mount point /boot
/dev/mapper/vg0-lvroot as ext4 and mount point /
/dev/mapper/vg0-lvhome as ext4 and mount point /home
Once the installation is finished, do not reboot and choose continue testing.
Step 2: Post-install Tasks
Go back to the terminal

# mount the installed system
mount /dev/vg0/lvroot /target
mount /dev/vg0/lvhome /target/home
mount -o bind /proc /target/proc
mount -o bind /sys /target/sys
mount -o rbind /dev /target/dev
mount /dev/md0p1 /target/boot

# change into the installed system
chroot /target/

# configure dns
echo "nameserver 8.8.8.8">/etc/resolv.conf

# install the packages
apt-get install mdadm

# create the crypttab
echo "pv0 /dev/md1 none luks">>/etc/crypttab

# reinstall dpkg and upstart to avoid some bug
apt-get install --reinstall dpkg 

# disable lvmetad
sed -E -i 's/(^\s)use_lvmetad.*/\1use_lvmetad = 0/' /etc/lvm/lvm.conf

# update the initramfs
update-initramfs -u -k all

# install grub
update-grub
grub-install /dev/sda
grub-install /dev/sdb

# encrypt home dir if there was no option in ubiquity (optional)
apt-get install ecryptfs-utils cryptsetup
ecryptfs-migrate-home -u USERNAME
rm -rf /home/USERNAME.*

# reboot
exit
reboot

Once it reboots, you will be asked for the password for unlocking the encrypted device /dev/md1 which you set in Step 0 with cryptsetup.

Troubleshooting nvidia cards
If you happen to be using nvidia card and unable to get the login screen with your username, install the proprietary nvidia drivers.

Press Ctrl - Alt - F3 and login to the tty with your username. Install nvidia-driver-xxx using apt. Reboot.
Posted 20th April 2019 by Fazle Arefin
Labels: 18.04 cryptsetup multi-disk raid secure ubuntu
0 Add a comment
AUG
7
Bypassing dangerous attachment restrictions in email
Recently I had to transfer some bash and python scripts to my work email from my personal email. I placed the scripts in a directory and created a .tar.gz file and sent it to my work email, only to receive an email soon afterwards saying the attrachments were blocked because they were dangerous. So here's what I did to get around this annoying restriction:

In my personal computer...

~/work
❯ ls
scripts  scripts.tar.gz

~/work
❯ tar tvfz scripts.tar.gz
drwxr-xr-x fazle/fazle       0 2018-08-07 21:48 scripts/
-rw-r--r-- fazle/fazle     870 2018-08-07 21:48 scripts/run.sh
-rw-r--r-- fazle/fazle     350 2018-08-07 21:48 scripts/generate.py

~/work
❯ base64 scripts.tar.gz
H4sIAC+HaVsAA+3UTQqDMBQE4Kw9RU5g8/J7kV5ASloFEUl00Z6+ChVswboxleJ8CGEwkMDoi5dQ
tV08sYTEwBkzruSMmK8TRkopqYTSZJkgMtIxblJeatLHrgics2vxqP2XfWvv/1R89R/6Jo9lmjPG
gq3WS/2T0fajf2mdYVykuc67g/d/LqvIh6euGs9lhoiIuHnc+y9fNs3/m298KDqft/fNz1iZ/8Ko
2fwXwz5S1mnM/1+Yf6iUIW2b9m4XAAAAAAAAAAAAAAAAAI7kCZ3AxCMAKAAA

~/work
❯ 

Now I copied this generated base64 encoded message and pasted it in the body of the email and sent the email to my work email. The next day I went to work and got back the scripts...

✔  ~/email-attachments
21:55 $ ls
✔  ~/email-attachments
21:56 $ echo "H4sIAC+HaVsAA+3UTQqDMBQE4Kw9RU5g8/J7kV5ASloFEUl00Z6+ChVswboxleJ8CGEwkMDoi5dQ
> tV08sYTEwBkzruSMmK8TRkopqYTSZJkgMtIxblJeatLHrgics2vxqP2XfWvv/1R89R/6Jo9lmjPG
> gq3WS/2T0fajf2mdYVykuc67g/d/LqvIh6euGs9lhoiIuHnc+y9fNs3/m298KDqft/fNz1iZ/8Ko
> 2fwXwz5S1mnM/1+Yf6iUIW2b9m4XAAAAAAAAAAAAAAAAAI7kCZ3AxCMAKAAA" | base64 -d | tar xvz
scripts/
scripts/run.sh
scripts/generate.py
✔  ~/email-attachments
21:56 $ ls
scripts
✔  ~/email-attachments
21:56 $ ls -l scripts/
total 24
-rw-r--r-- 1 fazle fazle 350 Aug  7 21:48 generate.py
-rw-r--r-- 1 fazle fazle 870 Aug  7 21:48 run.sh
✔  ~/email-attachments
21:56 $

Yes, I could just use Dropbox or carry the files in a flash drive. But in many workplaces you do not get that luxury as Internet is restricted and using usb flash disks are disabled in the laptop. The base64 encoding trick is a workaround in those cases.
Posted 7th August 2018 by Fazle Arefin

0 Add a comment
DEC
28
Dell XPS 15 9550 Bluetooth Fix | Linux
Dell XPS 15 9550 is missing the Linux firmware for the bluetooth device. This affects at least Fedora 27 and Ubuntu 17.10. Some other blogs have instructions on a fix but they ask you to download the firmware file from someone's dropbox share. I found a more reliable source to download the firmware from. Credit goes to Fedora Wiki for providing the link to download the firmware file from a more reliable source.

All you need to fix the bluetooth issue on your Dell XPS 15 9550 is

sudo wget -O /lib/firmware/brcm/BCM-0a5c-6410.hcd https://github.com/winterheart/broadcom-bt-firmware/blob/master/brcm/BCM20703A1-0a5c-6410.hcd\?raw\=true
And then reboot the system.

Posted 28th December 2017 by Fazle Arefin

2 View comments
OCT
22
Automated setup of your development laptop using Ansible
Ubuntu 17.10 has just been released. Every time there is a new release I end up doing a clean install of my office laptop, home laptop, and home desktop. Sure I can upgrade from the previous version 17.04 but I prefer doing a clean install to get rid of unwanted files and packages. Since I have 3 different systems to do a clean upgrade, manual upgrade is not practical. I ended up writing some Ansible to automate the process. I guess you might also find it useful and so I am sharing the Ansible playbook with you.

The Ansible playbooks to setup and configure your system as a development machine is available from my github repo https://github.com/fazlearefin/ubuntu-dev-machine-setup . Details are in the README of the repo.
Posted 22nd October 2017 by Fazle Arefin

0 Add a comment
MAR
25
Scrap your /bin/bash shell and use /bin/zsh in Linux or macOS



I love bash scripting. But when it comes to using bash for everyday CLI tasks, it can be a pain. I have been using zsh shell for the last one year and I can tell you that if you are not using it you are missing out from the fun of using the CLI. 

Zsh alone is not that exciting. There are several zsh plugins, like https://github.com/robbyrussell/oh-my-zsh (look at the number of stars), that makes it exciting.

I still use /bin/bash for scripting but /bin/zsh for cli tasks.
How do I start using /bin/zsh with the "revolutionary" oh-my-zsh?
1. Install zsh
Debian/Ubuntu

$ sudo apt install zsh

Fedora

$ sudo dnf install zsh

macOS

$ brew install zsh # you may skip this since macOS already ships with a version of zsh recent enough


2. Change your shell to zsh
Debian/Ubuntu/Fedora

$ chsh -s /bin/zsh

macOS


$ echo "$(which zsh)" | sudo tee -a /etc/shells
$ chsh -s $(which zsh)

Open and a new terminal and choose not to generate any ~/.zshrc


3. Get antigen
Antigen ( https://github.com/zsh-users/antigen ) is a zsh plugin manager. Although you can use oh-my-zsh without using antigen, using antigen lets your manage the zsh plugins easily.

$ mkdir ~/.antigen
$ curl -L git.io/antigen > ~/.antigen/antigen.zsh


4. Populate your ~/.zshrc
$ echo "export TERM='xterm-256color'

source ~/.antigen/antigen.zsh

# Load the oh-my-zsh's library.
antigen use oh-my-zsh

# Bundled plugins from the default repo (robbyrussell's oh-my-zsh)
# see https://github.com/robbyrussell/oh-my-zsh/tree/master/plugins for other ones
antigen bundle brew    # macOS only
antigen bundle colored-man-pages
antigen bundle colorize
antigen bundle docker
antigen bundle encode64
antigen bundle gem
antigen bundle git
antigen bundle git-extras
antigen bundle httpie
antigen bundle kubectl
antigen bundle node
antigen bundle npm
antigen bundle pep8
antigen bundle pip
antigen bundle pipenv
antigen bundle pyenv
antigen bundle pylint
antigen bundle python
antigen bundle ruby
antigen bundle sudo
antigen bundle terraform
antigen bundle vagrant
antigen bundle vault
antigen bundle yarn

# use the agnoster theme from oh-my-zsh
antigen theme agnoster

# Syntax highlighting bundle
antigen bundle zsh-users/zsh-syntax-highlighting

# pure theme (minimalistic and pretty)
#antigen bundle mafredri/zsh-async
#antigen bundle sindresorhus/pure

# powerlevel9k theme
#antigen theme bhilburn/powerlevel9k powerlevel9k

# bullet train theme
#antigen theme caiogondim/bullet-train.zsh bullet-train

# Tell antigen that you're done.
antigen apply" > ~/.zshrc


Now open a new terminal and wait a while for antigen to install the plguins. Try your nomal CLI tasks and see how much intelligent your shell is. Try going to a git repository and watch the prompt. (I personally use the pure theme because it is minimalist and it has asynchronous git fetch.)


Font Installation (optional)
If you find using some themes causing some weird characters in the terminal, try installing the patched fonts and then changing the fonts from the terminal menu. Following are instructions on installing the patched fonts:

# awesome fonts
git clone git@github.com:gabrielelana/awesome-terminal-fonts.git -b patching-strategy
cd awesome-terminal-fonts
if [[ "$(uname -s)" == "Linux" ]]; then
  cp -av patched/*.ttf $HOME/.local/share/fonts/
  fc-cache -fv
elif [[ "$(uname -s)" == "Darwin" ]]; then
  cp -av patched/*.ttf $HOME/Library/Fonts
fi

# powerline fonts
cd /tmp
git clone git@github.com:powerline/fonts.git
cd fonts
./install.sh
cd ..
rm -rf fonts

Posted 25th March 2017 by Fazle Arefin
Labels: /bin/bash /bin/zsh bash cli debian fedora macos oh-my-zsh shell theme ubuntu zsh
0 Add a comment
MAR
25
Running vagrant+virtualbox entirely from RAM
Background
I use vagrant (with VirtualBox as provider) on a daily basis at work to try out different things, especially trying out my Ansible playbooks for OS automation. One of the issues I was concerned about was wearing down the SSD and long wait times for a VM to boot up. The solution was to create a mount point in tmpfs and set the VirtualBox machine folder to that mount point in tmpfs. This makes VBox store the VM disk images entirely in RAM.

If you are not using a SSD and using the spinning hard disks, you will definitely see a huge boost in the startup/shutdown of your VMs.


Pros
Save SSD from wear
Improve VM speed
Cons
On host reboot, you will lose your VBox VMs. So this is not ideal if you want to preserve the state of your VMs.
You need lots of RAM (typically your system should have at least 16GB).
Getting it done
1. Create the tmpfs
I create a tmpfs in /vbox. You can either use systemd or just edit your /etc/fstab. I will show the systemd way to mount the /vbox mount point in tmpfs.


echo "[Unit]
Description=VBox Machine Folder
Documentation=man:hier(7)
Documentation=http://www.freedesktop.org/wiki/Software/systemd/APIFileSystems
ConditionPathIsSymbolicLink=!/vbox
DefaultDependencies=no
Conflicts=umount.target
Before=local-fs.target umount.target
After=swap.target

[Mount]
What=tmpfs
Where=/vbox
Type=tmpfs
Options=mode=1777,noatime,nosuid,nodev,size=6G

[Install]
WantedBy=local-fs.target" | sudo tee /lib/systemd/system/vbox.mount


I set /vbox to 6GB on my 16GB laptop. Change this to suit your needs. 6GB can hold 3 vagrant boxes.

2. Mount the /vbox mount point in tmpfs
$ sudo systemctl enable vbox.mount
$ sudo systemctl start vbox.mount

3. Change the machine folder in your Virtualbox
$ VBoxManage setproperty machinefolder /vbox


Now create your vagrant boxes and watch them fly. On a sepatate terminal keep running watch -n1 "df -h /vbox" to keep track of the space.

Posted 25th March 2017 by Fazle Arefin
Labels: debian fedora linux ssd tmpfs ubuntu vagrant virtualbox
0 Add a comment
FEB
18
QOwnNotes - Another note taking app for Linux
I had previously posted about Tagspaces being an alternative to Zim Desktop Wiki. I was not entirely happy with Tagspaces and my hunt for note taking app continued. After testing various Linux note taking apps out there I think I have finally found one which I will be using for a long time. It's called QOwnNotes. It has markdown support with auto completion, is open source and saves files in plain text format in a single directory chosen by me. Additionally, it can sync your notes with your ownCloud server but you can choose to keep your files locally on your disk with no online sync.
Version Controlled Notes
This is how I keep my notes version controlled using QOwnNotes and keep it synced between my devices.
Codes are worth more than words. So here's some bash code:
mkdir ~/my-notes
cd ~/my-notes
git init
# open QOwnNotes and choose this ~/my-notes as Notes folder
# create some notes in QOwnNotes and go back to terminal
git add -A
git commit -m "My initial notes"
# add your git remotes to git push to GitHub, Bitbucket, etc
git push -u origin master

Now from another device, I can git clone the git repo and get all the notes.

Posted 18th February 2017 by Fazle Arefin
Labels: fedora linux markdown notes qownnotes rednotebook simplenote springseed tagspaces ubuntu zim

0 Add a comment
FEB
6
zeal - Offline Documentation Viewer for Linux
How many times have you been switching through your browser's tabs for official online documentation whether it's Java, Go, Python, Ansible or Puppet?
Meet zeal, the offline documentation viewer for Linux. With zeal, you can download the official documentation of 195 different technologies (at the time of writing this post) right into zeal. In just one application, zeal, you can browse and search for that many docs. It comes bundled in the official repos of Ubuntu. Or get the latest version from zeal.

$ sudo apt install zeal

After that you just need to download the docsets through zeal.

On Mac, you can use Dash.

Posted 6th February 2017 by Fazle Arefin

0 Add a comment
FEB
5
Pandoc | swiss-army knife of markup document conversion
I have started a new job and I am having to write a lot of documentation. The only format I like to write documentation is markdown(md) but the company wiki requires me to write in mediawiki format which I despise. The hero to my rescue is pandoc. To install on Ubuntu:

$ sudo apt install pandoc
Now, I write my documentation in markdown using Atom Editor (on Linux). Then use pandoc to convert to mediawiki format after which a copy-paste to the Mediawiki.

$ atom some_doc.md
$ pandoc -f markdown -t mediawiki /tmp/some_doc.md -o /tmp/some_doc.mediawiki
Other formats supported by pandoc:
Input formats:  commonmark, docbook, docx, epub, haddock, html, json*, latex,
                markdown, markdown_github, markdown_mmd, markdown_phpextra,
                markdown_strict, mediawiki, native, odt, opml, org, rst, t2t,
                textile, twiki
                [ *only Pandoc's JSON version of native AST]

Output formats: asciidoc, beamer, commonmark, context, docbook, docbook5, docx,
                dokuwiki, dzslides, epub, epub3, fb2, haddock, html, html5,
                icml, json*, latex, man, markdown, markdown_github,
                markdown_mmd, markdown_phpextra, markdown_strict, mediawiki,
                native, odt, opendocument, opml, org, pdf**, plain, revealjs,
                rst, rtf, s5, slideous, slidy, tei, texinfo, textile, zimwiki
                [**for pdf output, use latex or beamer and -o FILENAME.pdf]

Posted 5th February 2017 by Fazle Arefin

0 Add a comment
NOV
13
Tagspaces - An alternative to Zim Desktop Wiki
I have been a long time user of Zim Desktop Wiki. Although it is a great desktop wiki, I do not quite like it for a number of reasons:
no markdown support (you can, however, export to markdown)
user interface is not modern
does not work on macOS (out of the box)

I needed a desktop wiki which: 
supported markdown
did not store my notes in the cloud owned by the app (like Evernote, Simplenote)
worked on both Linux and macOS
could be easily version controlled using git

I keep my notes in git version control (using private git repo in Altassian's Bitbucket), so that I can easily keep my notes synced between the different devices I use (a MacBook, a Linux Desktop, 2 Linux Laptops). There are lot of note taking app supporting markdown in macOS such as Bear but I wasn't able to find a decent one for Linux until about half an hour ago. It's Tagspaces. Things that I like about Tagspaces:
markdown support
tagging files
open source
files stored locally and easily version controlled
modern user interface
Linux and macOS support

Tagspaces is actually meant for managing you local files. It doesn't advertise itself as being a note taking app. In the app, you can connect to folder(s) and tag files, rename files, etc. But since it has native markdown support it satisfies my requirement as a Desktop note taking app.

Goodbye Zim. It's been over six years.
Posted 13th November 2016 by Fazle Arefin
Labels: desktop evernote markdown notes simplenote tagspaces wiki zim

4 View comments
JUN
19
Dell XPS 13 or 15 | Set Keyboard Illumination Timeout
I got myself a Dell XPS 15 recently. One of the annoying issues was the keyboard illumination turning off after 10 seconds. I finally found an easy fix to set the timeout. This solution will probably work for other Dell models as well. I am using Ubuntu 16.04.


sudo apt install smbios-utils
sudo smbios-keyboard-ctl --get-status      # you will be able to see timeout is 10 seconds
sudo smbios-keyboard-ctl --set-timeout 1m  # set timeout to 1 minute, or set to 1h if you like

That's it.

The other annoying issue is the touchscreen (I have the touchscreen model) not working after a suspend and resume. I will keep looking for an easy fix but if you have a fix please let me know in the comments section.

Posted 19th June 2016 by Fazle Arefin
Labels: 13 15 16.04 backlight dell illumination inspiron keyboard latitude linux ubuntu xps
0 Add a comment
MAR
7
What init system am I running? (SysVinit/Upstart/Systemd)
The mainline init systems are either SysVinit or Upstart or Systemd. There are others, but I am not focusing on them in this post. Now, how do you find out which init system your distro is running? I have been looking for an easy solution and here is what I got.

On a rpm based distro, such as RHEL, CentOS, Fedora:

rpm -qf /sbin/init
On a deb based distro, such as Debian, Mint, Ubuntu:

dpkg -S /sbin/init
This should output what package the file belongs to. It will be one of the three mentioned above. And this is the init system you are running.

For other distributions you can inspect the /sbin/init file and try to find out what package the file belongs to, or where it symlinks to.

Feel free to comment if you have other ideas.

Posted 7th March 2016 by Fazle Arefin
Labels: init system systemd sysvinit upstart
0 Add a comment
NOV
19
Flashing Nexus 5/5X/6/6P/9 with Factory Images on Fedora 22/23
This guide is supposed to be a self-reference for me as well as others who got frustrated following flashing tutorials from other sites. I have tried to make this really easy to follow. You will need to have basic level of bash and Linux experience to follow this tutorial. This tutorial is meant only for Google Nexus phones and tablets. It might work on other Nexus devices but I have not been able to test as I don't own any other Nexus devices. The instructions are meant for flashing a laptop/desktop running Fedora 22/23 but will work on other distros (Ubuntu/Debian/openSUSE) wilth minor modifications.

# On Nexus Phone/Tablet 
- Turn off device
- Press Power - VolUp - VolDown buttons simultaneously to get to the boot menu (one that says Start at the top)
- Connect the Nexus Device to the laptop with a usb/microusb cable

# On laptop 
$ sudo dnf install -y android-tools usbutils 
$ NEXUSDEVICEIDS=$(lsusb | egrep -i 'Google|Nexus' | cut -d' ' -f6) 
$ idVendor=$(cut -d: -f1 <<< $NEXUSDEVICEIDS) 
$ idProduct=$(cut -d: -f2 <<< $NEXUSDEVICEIDS) 
$ echo "SUBSYSTEM==\"usb\", ATTR{idVendor}==\"${idVendor}\", ATTR{idProduct}==\"${idProduct}\", GROUP=\"androiddev\", MODE=\"0664\"" | sudo tee /etc/udev/rules.d/99-android-debug.rules 
$ sudo udevadm control --reload 
$ sudo systemctl restart systemd-udevd.service 

 # On laptop 
- Download Android image from https://developers.google.com/android/nexus/images?hl=en
(Make sure you download the image for your device. Check again!)

# On laptop 
$ tar xvfz downloaded-image.tgz 
$ cd downloaded-image-folder 
$ sudo fastboot devices 
$ sudo fastboot oem unlock 

 # On Nexus Phone/Tablet 
- Use the VolUp/VolDown keys to select Yes and press Power. LOCK STATE will now say unlocked

 # On laptop 
$ sudo ./flash-all.sh 
(The image will get transferred to your phone from the laptop and then the phone will reboot automatically)
(If you are stuck on the android animation screen too long, hold down the Power Button to turn off)

# On Nexus Phone/Tablet 
- Turn off device
- Press Power - VolUp - VolDown buttons simultaneously to get to the boot menu (one that says Start at the top)

# On laptop 
$ sudo fastboot oem lock 
(This command worked fine on my Nexus 5 but when I tried on my Nexus 6, it failed. I had to turn on developer mode and then turn on oem unlocking first to lock the bootloader again.)

# On Nexus Phone/Tablet 
(LOCK STATE will now say locked)
- Press Power to start the device
(If you are stuck on the android animation screen too long, hold down the Power Button to turn off. And then turn on the device again)

All done!

Posted 19th November 2015 by Fazle Arefin
Labels: android factory fedora image marshmallow nexus
0 Add a comment
OCT
23
Installing/Updating Atom Editor (Fedora 22 or later)
I have been using Atom Editor from GitHub alongside vim (+ vim-spf13) for the last one year. Atom still does not have a yum or apt repo (for Linux) to ease the process of installing or updating Atom. Installing or updating Atom requires checking their website manually. Here's a script to install or update to the latest release version of Atom Editor. I have added apm update --no-confirm so that Atom packages get updated as well.

#!/bin/bash

ATOM_INSTALLED_VERSION=$(atom --version 2>/dev/null | grep -i Atom | egrep -o '[0-9\.]+$' || echo "")
ATOM_LATEST_VERSION=$(curl -o /dev/null --silent --head --write-out '%{redirect_url}\n' https://github.com/atom/atom/releases/latest | egrep -o '[0-9\.]+$')

if [[ $ATOM_INSTALLED_VERSION != $ATOM_LATEST_VERSION ]]; then
  /bin/rm -f /tmp/{latest,atom.x86_64.rpm}
  wget -q https://github.com/atom/atom/releases/latest -O /tmp/latest
  wget --progress=bar -q 'https://github.com'$(cat /tmp/latest | grep -o -E 'href="([^"#]+)atom.x86_64.rpm"' | cut -d'"' -f2 | sort | uniq | tail -n1) -O /tmp/atom.x86_64.rpm -q --show-progress
  sudo dnf -y update /tmp/atom.x86_64.rpm || sudo dnf -y install /tmp/atom.x86_64.rpm
else
  echo "No update available. You are already running the latest version." >&2
fi

# now update atom packages
apm update --no-confirm

Instead of creating a script you can wrap this in a function in your .bashrc. With some modifications you can also make this script work on Ubuntu or Debian.

Posted 23rd October 2015 by Fazle Arefin
Labels: atom editor fedora github install update
0 Add a comment
JUL
12
Bash CLI editing shortcuts mega list
I am obsessed with the CLI and have looked everywhere to find the comprehensive list for Bash CLI editing. Unfortunately none of the cheatsheet links had the comprehensive link. I finally found the list right in the bash manpage! They are all in one section and you can actually get the full list for your Bash version by:

man bash | awk '/^   Commands for Moving$/{print_this=1} /^   Programmable Completion$/{print_this=0} print_this==1{sub(/^   /,""); print}'
I am running GNU bash, version 4.3.39 and here's what works for me. C-a means Ctrl and A. M-b means Alt and B

Commands for moving
C-a
Move to the start of the current line

C-e
Move to the end of the line

C-f or Right Arrow
Move forward a character

C-b or Left Arrow
Move back a character

M-f
Move forward to the end of the next word.  Words are composed of alphanumeric characters (letters and digits)

M-b
Move back to the start of the current or previous word.  Words are composed of  alphanumeric  characters (letters and digits)

C-l
Clear the screen leaving the current line at the top of the screen.  With an  argument,  refresh  the current line without clearing the screen
(More coming soon. I just don't have time at the moment for editing. Meanwhile, try the one-liner I posted above to get the full list from the Bash manpage)

Posted 12th July 2015 by Fazle Arefin

0 Add a comment
FEB
20
Transforming vim to an IDE using SPF13 vim distribution

I have moved to using SpaceVim. I encourage you to try out SpaceVim as SPF13 development seems to have stalled.
Why use vim when I can use a GUI IDE?
vim works on a CLI. And everyone knows CLI is much faster over a ssh connection compared to GUI. vim is however comes with basic functionalities not very helpful when coding in your facorite language. So some plugins need to be added. In this tutorial I will be showing you how to add SPF13 vim distribution which comes bundled with some really good vim plugins. Ofcourse you can choose to download and use the plugins on your own, but using the SPF13 just makes everything easier.

Before proceeding further, backup your ~/.vimrc  and other vim customization files from your ~

Step 0 Get the VIM package with support for clipboard (optional)
Check if you have clipboard support in vim:

vim --version | grep '+clipboard'

If you do not have clipboard support then

For Fedora:
yum install vim vim-X11
Then launch vimx

For Ubuntu:
apt-get install vim-gnome # or vim-gtk or vim-athena
The launch vim  as usual

Step 1 Get the mega VIM distro packaged by SPF13
This will install the vundle plugin manager and other vim plugins
curl http://j.mp/spf13-vim3 -L -o - | sh

On Fedora, -L  may not work as it should. If you get an error, try
curl https://raw.githubusercontent.com/spf13/spf13-vim/3.0/bootstrap.sh -o - | sh

Step 2 Remove unnecessary vim plugins bundled with spf13
The ~/.vimrc  file is best left untouched as spf13 makes modifications to this file.

Create and modify ~/.vimrc.local  for your customizations. Here's a sample copy of ~/.vimrc.local

Comments begin with ". To enable/disable any plugin, prepend/remove the " before the UnBundle keyword. Note that removing some plugins may break other plugins. For details, google the plugin name to check the relevant github site.

You may want to change the vim colorscheme and vim­airline theme

File: ~/.vimrc.local

" no backups "
set nobackup
set nowritebackup

" Vim colorscheme
colorscheme osx_like

" START Load vim-airline plugin "
set t_Co=256

" required for vim-airline status color
if !exists('g:airline_symbols')
let g:airline_symbols = {}
endif

" vim-airline theme
let g:airline_theme = 'luna'

" unicode symbols
let g:airline_left_sep = '»'
let g:airline_left_sep = '?'
let g:airline_right_sep = '«'
let g:airline_right_sep = '?'
let g:airline_symbols.linenr = '?'
let g:airline_symbols.linenr = '?'
let g:airline_symbols.linenr = '¶'
let g:airline_symbols.branch = '?'
let g:airline_symbols.paste = ' ρ'
let g:airline_symbols.paste = 'Þ'
let g:airline_symbols.paste = '?'
let g:airline_symbols.whitespace = ' Ξ '
" END "

" BEGIN Highlight current line "
set cursorline
hi CursorLine cterm=NONE ctermbg=lightyellow
hi CursorColumn cterm=NONE ctermbg=lightyellow
nnoremap c :set cursorline! cursorcolumn!
" END "

" BEGIN Get rid of unnecessary plugins "

" interpret a file by function and cache file automatically
"UnBundle 'MarcWeber/vim-addon-mw-utils'

" Some utility functions for VIM http://www.vim.org/scripts/script.php?script_id=1863
"UnBundle 'tomtom/tlib_vim'

" A tree explorer plugin for vim
"UnBundle 'scrooloose/nerdtree'

" precision colorscheme for the vim text editor
UnBundle 'altercation/vim-colors-solarized'

" Collection of color schemes for VIM
UnBundle 'spf13/vim-colors'

" quoting/parenthesizing made simple
"UnBundle 'tpope/vim-surround'

" enable repeating supported plugin maps with "."
"UnBundle 'tpope/vim-repeat'

" Inserts matching bracket, paren, brace or quote http://www.vim.org/scripts/script.php?script_id=1849
"UnBundle 'spf13/vim-autoclose'

" Fuzzy file, buffer, mru, tag, etc finder
"UnBundle 'kien/ctrlp.vim'

" Navigate and jump to function defs http://www.vim.org/scripts/script.php?script_id=4592
"UnBundle 'tacahiroy/ctrlp-funky'

" True Sublime Text style multiple selections for Vim
"UnBundle 'terryma/vim-multiple-cursors'

" Vim session manager http://www.vim.org/scripts/script.php?script_id=2010
UnBundle 'vim-scripts/sessionman.vim'

" extended % matching for HTML, LaTeX, and many other languages


This repository hosts the [Infra Standard](https://infra.spec.whatwg.org/).

## Code of conduct

We are committed to providing a friendly, safe, and welcoming environment for all. Please read and respect the [Code of Conduct](https://whatwg.org/code-of-conduct).

## Contribution opportunities

Folks notice minor and larger issues with the Infra Standard all the time and we'd love your help fixing those. Pull requests for typographical and grammar errors are also most welcome.

Issues labeled ["good first issue"](https://github.com/whatwg/infra/labels/good%20first%20issue) are a good place to get a taste for editing the Infra Standard. Note that we don't assign issues and there's no reason to ask for availability either, just provide a pull request.

If you are thinking of suggesting a new feature, read through the [FAQ](https://whatwg.org/faq) and [Working Mode](https://whatwg.org/working-mode) documents to get yourself familiarized with the process.

We'd be happy to help you with all of this [on Chat](https://whatwg.org/chat).

## Pull requests

In short, change `infra.bs` and submit your patch, with a [good commit message](https://github.com/whatwg/meta/blob/main/COMMITTING.md).

Please add your name to the Acknowledgments section in your first pull request, even for trivial fixes. The names are sorted lexicographically.

To ensure your patch meets all the necessary requirements, please also see the [Contributor Guidelines](https://github.com/whatwg/meta/blob/main/CONTRIBUTING.md). Editors of the Infra Standard are expected to follow the [Maintainer Guidelines](https://github.com/whatwg/meta/blob/main/MAINTAINERS.md).

## Tests

Tests are an essential part of the standardization process and will need to be created or adjusted as changes to the standard are made. Tests for the Infra Standard can be found in the `infra/` directory of [`web-platform-tests/wpt`](https://github.com/web-platform-tests/wpt).

A dashboard showing the tests running against browser engines can be seen at [wpt.fyi/results/infra](https://wpt.fyi/results/infra).

## Building "locally"

For quick local iteration, run `make`; this will use a web service to build the standard, so that you don't have to install anything. See more in the [Contributor Guidelines](https://github.com/whatwg/meta/blob/main/CONTRIBUTING.md#building).

## Formatting

Use a column width of 100 characters.

Do not use newlines inside "inline" elements, even if that means exceeding the column width requirement.
```html
<p>The
<dfn method for=DOMTokenList lt=remove(tokens)|remove()><code>remove(<var>tokens</var>&hellip;)</code></dfn>
method, when invoked, must run these steps:
```
is okay and
  ```html
<p>The <dfn method for=DOMTokenList
lt=remove(tokens)|remove()><code>remove(<var>tokens</var>&hellip;)</code></dfn> method, when
invoked, must run these steps:
```
is not.

Using newlines between "inline" element tag names and their content is also forbidden. (This actually alters the content, by adding spaces.) That is
```html
<a>token</a>
```
is fine and
```html
<a>token
</a>
```
is not.

An `<li>` element always has a `<p>` element inside it, unless it's a child of `<ul class=brief>`.

If a "block" element contains a single "block" element, do not put it on a newline.

Do not indent for anything except a new "block" element. For instance
```html
 <li><p>For each <var>token</var> in <var>tokens</var>, in given order, that is not in
 <a>tokens</a>, append <var>token</var> to <a>tokens</a>.
```
is not indented, but
```html
<ol>
 <li>
  <p>For each <var>token</var> in <var>tokens</var>, run these substeps:

  <ol>
   <li><p>If <var>token</var> is the empty string, <a>throw</a> a {{SyntaxError}} exception.
```
is.

End tags may be included (if done consistently) and attributes may be quoted (using double quotes), though the prevalent theme is to omit end tags and not quote attributes (unless they contain a space).

Place one newline between paragraphs (including list elements). Place three newlines before `<h2>`, and two newlines before other headings. This does not apply when a nested heading follows the parent heading.
```html
<ul>
 <li><p>Do not place a newline above.

 <li><p>Place a newline above.
</ul>

<p>Place a newline above.


<h3>Place two newlines above.</h3>

<h4>Placing one newline is OK here.</h4>


<h4>Place two newlines above.</h4>
```
Use camel-case for variable names and "spaced" names for definitions, algorithms, etc.
```html
<p>A <a for=/>request</a> has an associated
<dfn export for=request id=concept-request-redirect-mode>redirect mode</dfn>,...
```
```html
<p>Let <var>redirectMode</var> be <var>request</var>'s <a for=request>redirect mode</a>.
```
