# APIGateway (load-balancer) and Service Discovery Project

NOTE: Renamed from an older project to this wonder well thought out generic name :). Also, this project only supports Linux.

---

In an environment where there are many API endpoints it becomes difficult for developers to keep track and if a change is made to an endpoint there can be very bad outcomes.

Phase I (load-balancer):

The first phase of this project is an API Gateway that accepts on ports 80 and 443. Port 80 is actually redirected to 443 since all endpoints should be secure. I know, you're thinking but doesn't TLS add almost a 20% overhead on the initial series of handshaking? The answer of course is yes, with HTTP/1.X. With HTTP/2 the amount of overhead is almost all gone so everything should use TLS no matter what. Besides, it just good practice. However, this version can but does not support HTTP/2 out of the box so to speak since almost all backends are still HTTP/1.X.

>`tokio-http2` project in this org's repo is being developed now and it fully supports HTTP/2. This is very important since HTTP/2 requires HPACK (header compression) and Multiplexing (multiple requests in/out on same TCP connection). The additional throughput is huge.

The great news about the api-gateway service is that it's using HAProxy to do the bulk of the work in this phase. BTW, HAProxy must be version >= 1.6 because starting in version 1.6 you can specify a FQDN in the `backend` server instead of just an IP address.

For example:

```
server someservername api.of.some.service.somewhere.com:443 check ssl verify none
vs
server someservername 10.10.10.100:80 check port 80
```

The above example assumes that the `api.of.some.service.somewhere.com` has a valid cert so that you can relay TLS traffic but with the naked IP you can't (unless of course you set the CN up that way and IIRC I don't believe you can for a valid cert).

## Linux Distributions

As of this writing RHEL/CentOS 7.2 only support 3.X kernels but Ubuntu 16.04 supports 4.X kernel (only tried with RHEL and Ubuntu). This is important since 4.X kernels support larger TCP Window and Fast Open (along with other TCP enhancements) that provide you with closer to `line-speed` throughput. In addition, the default version of HAProxy on RHEL/CentOS was 1.5.8 while Ubuntu 16.04 the default version was 1.6.3. The latest version of HAProxy at the time of this writing is 1.7.0.

Instead of building HAProxy and OpenSSL from source I decided to use Ubuntu (my goto is usually RHEL) to keep things simple.

## Performance

This is always the first thing everyone thinks of or at least it should be. I don't have specific numbers because that depends on your environment (server, switches, NICs, etc). However, I can tell you that I tested against a very well known hardware load-balancer (also very expensive) and I was surprised to see that HAProxy outperformed it in every single test (on bare-metal)!

## Architecture

.. coming soon ..

## Instructions

The API Gateway can be used on bare-metal machines or VMs depending on your environment and needs. In an enterprise market the most important thing is reliability (actually everywhere I suppose) with performance a close second. No one wants to get called in the middle of the night because the gateway went down for some reason.

Of course there are enhancements that can be made to the HAProxy config and I will leave that to you. However, to get up and running quickly simply do the following:

1. Get a bare-metal node or VM with a base OS of Ubuntu 16.04
2. Install HAProxy: `sudo apt install haproxy -y`
3. If your server can access the outside world then `fork this repo` (oh, star it too :)) and then `git clone <your forked repo>` to the server or to your desktop/laptop
4. Edit the `haproxy.cfg` file to replace the `<backend server info>` place holders. Of course, feel free to customize it to fit your needs beyond what it is. BTW, if you have good suggestions on improving then issue a PR (pull request) from your forked github repo
5. If your server is on a bare-metal machine then make sure `bind 0.0.0.0` is set to the IP of the server like `bind 10.10.10.100`. If you're on a VM then most likely the `bind 0.0.0.0` will be fine but double check with your cloud admin
6. Make sure your server can see backend servers and any firewalls are open to allow your new API Gateway
7. If you are using a VM say in OpenStack then make sure you have changed your default rules (or whatever rule you're using) to allow for port 80 and 443. As simple as this is, it's what gets everyone because it's so easy to forget!
8. IMPORTANT: If not using TLS then comment out the `bind 0.0.0.0:443 ssl...` and change it to `bind 0.0.0.0:80` (see #5 above)
9. IMPORTANT: If using TLS then make sure to create a CSR via openssl and get a valid cert. BTW, you can get up to 10 free certs at [StartSSL](https://www.startssl.com/). I have not tried but I heard they work well. Anyway, you should get back a CRT (cert) and maybe an intermediate CRT. HAProxy uses a PEM file which has the format as: `primary cert + intermediate cert + private key`. The private key would have been generated during the openssl CSR phase. Put the PEM in `/etc/ssl/private`. If you want to put it somewhere else then make sure you update the path in `haproxy.cfg`.
10. Restart HAProxy: `sudo systemctl restart haproxy`. If no error shows then your good. If an error pops up it is most likely due to a syntax error so double check `haproxy.cfg`
11. Enable HAProxy to start on boot: `sudo systemctl enable haproxy`
12. You're all set. Of course, make sure that your final endpoints are already working before doing this

## It's running, now what?

See Architecture above on how to setup a load-balanced environment.

## Automation

You can easily setup Chef, Ansible or whatever to automate the process. That exercise is up to you. Take `haproxy.cfg` and add template place holders where you want to the data to be. You will need a way to feed in the data to the template for the sections with multiple listings.
