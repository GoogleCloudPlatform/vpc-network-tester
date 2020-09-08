# Serverles VPC Access Diagnostic Tools

This repository contains Google App Engine and Google Cloud Run services
that are deployable in a customer's project to diagnose and debug the
configuration of serverless networking for the serverless services
including the VPC Access connectors in the customer's
project.

Google Cloud serverless solutions including App Engine, Cloud Functions, 
and Cloud Run are able to be optionally connected to a customer's
VPC network through use of [Serverless VPC Access connectors](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access).

## Installation in your Google Cloud Project

This repository includes the same Python application packaged for use with either
App Engine or Cloud Run.  Choose one of the two technologies and deploy
the service into your Google Cloud project.  Once deployed, you will be able
to run the same diagnostics with either solution.

### Install Cloud Run service

The Cloud Run version builds a container image with Google Cloud Build that is
pushed to your project's Google Cloud Container Registry.  It is possible to
build this container image with the provided Dockerfile on your local machine
but the instructions below are for Cloud Build:

1. Clone this repository from GitHub.
    * There is no need to fork it.
1. Build container image (using Cloud Build):
    * ```cd cloudrun; gcloud builds submit --tag gcr.io/[PROJECT]/vpc-access-diagnostics```
1. Deploy Cloud Run service to your project
    * ```cd cloudrun; gcloud run --platform=managed --region=[REGION] deploy [SERVICE_NAME] --image gcr.io/[PROJECT]/vpc-access-diagnostics --vpc-connector=projects/[PROJECT]/locations/[REGION]/connectors/[CONNECTOR_NAME]```

### Install App Engine service

The App Engine version assumes you are already using Google App Engine and that
you already have a default service deployed.  The instructions below include
optionally choosing a different service name, and it is possible to use this as
the default service in your project.

1. Clone this repository from GitHub.
    * There is no need to fork it.
1. Modify appengine/app.yaml to:
    * Optionally: change the App Engine service name
    * Optionally: increase the instance class
        * Instance class in App Engine is the ammount of a CPU core and memory
          provided to your code.  Latency (especially at the 95-99 percentile)
          and single-instance throughput
          can be impacted by the CPU availability, but connectivity testing
          won't be affected by increasing instance class.
    * Update vpc_access_connector name to reference your connector 
      ([see instructions for where to find it](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access))
1. Optionally: Build iperf3 binary:
    * You can skip this step if you don't want to run iperf3 throughput tests
    * ```cd ipref3_build; ./build_iperf3.sh```
1. Deploy App Engine service to your project
    * ```cd appengine; gcloud app deploy```

## Usage

This application presents a simple HTML UI on it's '/' path.  Point a browser
at this Cloud Run or App Engine SERVICE_HOSTNAME reported from either
`gcloud run deploy` or `gcloud app deploy` command above to interact with the
user interface.  This currently presents the ability to diagnose HTTP(s) GET
an arbitrary URL, ICMP ping an arbitrary host or IP address, or run iperf3
client against an arbirary host.

In order to use iperf3, you must have an iperf3 server running on the host specified.
This can be a VM instance started in your VPC or anywhere reachable via the VPC
connector (or public internet).

### Validating Serverless VPC Access

The examples below reference public hostnames, but private IP addresses are just as possible.
If the customer wishes to validate access to their services on their VPC or over a VPN to
their on-premise services via an IP address (or GCE DNS custom domain) they can do so.  An
example of NGINX running on a VM in a VPC Network attached by VPC Access:

https://[SERVICE_HOSTNAME]/http?url=http%3A%2F%2F10.128.0.2

```
curl --silent --verbose http://10.128.0.2

* Rebuilt URL to: http://10.128.0.2/
*   Trying 10.128.0.2...
* TCP_NODELAY set
* Connected to 10.128.0.2 (10.128.0.2) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.128.0.2
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.10.3
< Date: Fri, 31 Jul 2020 21:39:24 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Fri, 27 Mar 2020 21:14:20 GMT
< Connection: keep-alive
< ETag: "5e7e6cac-264"
< Accept-Ranges: bytes
<
{ [612 bytes data]
* Connection #0 to host 10.128.0.2 left intact
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### Example HTTP Probe

The following is not left alive when I am not testing, but an example
URL of https://[SERVICE_HOSTNAME]/ping?host=yahoo.com
outputs:

```
curl --silent --verbose http://yahoo.com

* Rebuilt URL to: http://yahoo.com/
*   Trying 2001:4998:58:1836::10...
* TCP_NODELAY set
* Connected to yahoo.com (2001:4998:58:1836::10) port 80 (#0)
> GET / HTTP/1.1
> Host: yahoo.com
> User-Agent: curl/7.58.0
> Accept: */*
>
< HTTP/1.1 301 Moved Permanently
< Date: Fri, 31 Jul 2020 21:36:03 GMT
< Connection: keep-alive
< Server: ATS
< Cache-Control: no-store, no-cache
< Content-Type: text/html
< Content-Language: en
< X-Frame-Options: SAMEORIGIN
< Location: https://yahoo.com/
< Content-Length: 8
<
{ [8 bytes data]
* Connection #0 to host yahoo.com left intact
redirect
```

### Example Ping

The following is not left alive when I am not testing, but an example
URL of https://[SERVICE_HOSTNAME]/ping?host=yahoo.com
outputs:

```
PING yahoo.com(media-router-fp2.prod1.media.vip.gq1.yahoo.com (2001:4998:c:1023::5)) 56 data bytes
64 bytes from media-router-fp2.prod1.media.vip.gq1.yahoo.com (2001:4998:c:1023::5): icmp_seq=1 time=52.1 ms
64 bytes from media-router-fp2.prod1.media.vip.gq1.yahoo.com (2001:4998:c:1023::5): icmp_seq=2 time=98.5 ms
64 bytes from media-router-fp2.prod1.media.vip.gq1.yahoo.com (2001:4998:c:1023::5): icmp_seq=3 time=50.7 ms
64 bytes from media-router-fp2.prod1.media.vip.gq1.yahoo.com (2001:4998:c:1023::5): icmp_seq=4 time=50.6 ms

--- yahoo.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 50.695/63.049/98.581/20.524 ms
```

### Example IPerf3

This example requires having an IPerf3 server running for the client to connect to.  The most relevant
use case is via VPC Access to a VM on a VPC in the same region.  This specific example runs a single
instance communicating to the default iperf3 port on a single server.  To perform more exhaustive and
representative performance testing, multiple instances should be run in parallel via an HTTP automation
trigger that can trigger multiple HTTP requests in parallel (e.g. Apache Bench)

https://[SERVICE_HOSTNAME]/iperf?host=10.128.0.2

```
Connecting to host 10.128.0.2, port 5201
[  4] local 169.254.8.1 port 30011 connected to 10.128.0.2 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  73.6 MBytes   616 Mbits/sec    0   0.00 Bytes
[  4]   1.00-2.00   sec  78.6 MBytes   660 Mbits/sec    0   0.00 Bytes
[  4]   2.00-3.00   sec  49.0 MBytes   411 Mbits/sec    0   0.00 Bytes
[  4]   3.00-4.00   sec  69.6 MBytes   584 Mbits/sec    0   0.00 Bytes
[  4]   4.00-5.00   sec  71.0 MBytes   594 Mbits/sec    0   0.00 Bytes
[  4]   5.00-6.00   sec  71.1 MBytes   598 Mbits/sec    0   0.00 Bytes
[  4]   6.00-7.00   sec  67.4 MBytes   565 Mbits/sec    0   0.00 Bytes
[  4]   7.00-8.01   sec  83.6 MBytes   697 Mbits/sec    0   0.00 Bytes
[  4]   8.01-9.00   sec  82.1 MBytes   693 Mbits/sec    0   0.00 Bytes
[  4]   9.00-10.00  sec  77.4 MBytes   648 Mbits/sec    0   0.00 Bytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec   723 MBytes   607 Mbits/sec    0             sender
[  4]   0.00-10.00  sec   723 MBytes   606 Mbits/sec                  receiver

iperf Done.
```

IPERF3 URL above supports additional query string parameters that are not exposed in the UI:

| Query String Parameter | Description                                             | Default   |
|------------------------|---------------------------------------------------------|-----------|
| udp                    | Run bandwidth generator with UDP packets instead of TCP | off       |
| bandwidth              | Attempt to meet specified bandwidth (mbps)              | max (tcp) |
| plen                   | Packet size to send (bytes)                             | MTU       |
| npackets               | Finish after sending specified number of packets        | use time  |
| streams                | Number of parallel TCP sessions to use                  | 1         |
| port                   | Port number of the iperf3 server                        | 5201      |
| time                   | Length of type to send bandwidth traffic for (sec)      | 10        |


Example of running 20 simultaneous instances each running 20 simultaneous TCP streams for 30
seconds each and repeating the invocations to generate load for an hour:

```
for i in 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20; do
    ab -s 600 -n 1000 https://[SERVICE_HOSTNAME]/iperf?host=10.128.0.2\&streams=20\&time=30\&port=$((i+5200)) >& log_$((i)) < /dev/null &
done
```
