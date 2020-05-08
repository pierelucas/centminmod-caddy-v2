# Caddy v2 Install On Centmin Mod LEMP Stack

Below outlined steps are for private Caddy v2 server testing on Centmin Mod LEMP stack for evaluation of Caddy server as a candidate for integration into Centmin Mod LEMP stack builds based on Caddy v2 server's performance and scalability. Centmin Mod LEMP stack uses Nginx mainline branch custom built binaries by default with long term plans to integrate other web servers including Litespeed/OpenLitespeed and Apache.

This guide is unsupported and subject to revisions over time and currently based on my first few hours of using newly released Caddy v2 server.

Previous Caddy benchmarks

* https://community.centminmod.com/threads/caddy-http-2-server-benchmarks.5170/
* https://community.centminmod.com/threads/caddy-http-2-server-benchmarks-part-2.12873/
* https://community.centminmod.com/threads/caddy-0-11-5-tls-1-3-http-2-https-benchmarks-part-1.16779/

## Installation requirements and configuration

### 1. Setup /etc/hosts local DNS override for test dummy domain names 

For `ngx.domain.com` and `caddy.domain.com`

Change `192.168.0.18` to your server's IP address

```
# /etc/hosts
192.168.0.18 ngx.domain.com
192.168.0.18 caddy.domain.com
```

### 2. Install Centmin Mod 123.09beta01 LEMP stack on CentOS 7 64bit server 

With default PHP 7.3 (php-fpm) and MariaDB 10.3 MySQL server

As per [official install page instructions](https://centminmod.com/install.html) or you can try advanced install guide at [https://servermanager.guide/117/centmin-mod-advanced-customised-installation-guide/](https://servermanager.guide/117/centmin-mod-advanced-customised-installation-guide/).

```
yum -y update; curl -O https://centminmod.com/betainstaller73.sh && chmod 0700 betainstaller73.sh && bash betainstaller73.sh
```

### 3. Create Nginx non-HTTPS (port 80) & HTTPS (port 443) vhost sites

Centmin Mod Nginx can create Nginx vhost sites via `centmin.sh` shell based menu or via `nv` command line. For this guide, I will use `nv` command line:

```
nv

Usage: /usr/bin/nv [-d yourdomain.com] [-s y|n|yd|le|led|lelive|lelived]

  -d  yourdomain.com or subdomain.yourdomain.com
  -s  ssl self-signed create = y or n or https only vhost = yd
  -s  le - letsencrypt test cert or led test cert with https default
  -s  lelive - letsencrypt live cert or lelived live cert with https default

  example:

  /usr/bin/nv -d yourdomain.com -s y
  /usr/bin/nv -d yourdomain.com -s n
  /usr/bin/nv -d yourdomain.com -s yd
  /usr/bin/nv -d yourdomain.com -s le
  /usr/bin/nv -d yourdomain.com -s led
  /usr/bin/nv -d yourdomain.com -s lelive
  /usr/bin/nv -d yourdomain.com -s lelived
```

For self-signed ECDSA 256bit SSL certificate instead of default self-signed RSA 2048bit SSL certificates, add `SELFSIGNEDSSL_ECDSA='y'` to Centmin Mod persistent config file at /etc/centminmod/custom_config.inc which can override Centmin Mod defaults globally. This command will create both non-HTTPS (port 80) & HTTPS (port 443) vhost sites for Nginx with pwgen randomly generated pure-ftpd virtual FTPS username/password.

```
echo "SELFSIGNEDSSL_ECDSA='y'" >> /etc/centminmod/custom_config.inc
nv -d ngx.domain.com -s y -u $(pwgen -1cnys 31)
```

If using real domains that have DNS resolving to server IP and want live Letsencrypt SSL certificates:

```
echo "LETSENCRYPT_DETECT='y'" >> /etc/centminmod/custom_config.inc
echo "DUALCERTS='y'" >> /etc/centminmod/custom_config.inc
nv -d ngx.domain.com -s y -u $(pwgen -1cnys 31)
```

* The first variable `LETSENCRYPT_DETECT='y'` enables regular RSA 2048bit SSL certificates via Letsencrypt.
* While second variable `DUALCERTS='y'` enables dual RSA 2048bit + ECDSA 256bit SSL certificate mode with a second Letsencrypt SSL certificated being obtained that is ECDSA 256bit based. Dual SSL certificates allow Centmin Mod Nginx to serve better performance based ECDSA 256bit SSL certificates to web browser and clients that support such certificates while falling back to traditional standard RSA 2048bit SSL certificates for older web browser and clients that do not support ECDSA 256bit. 

### 4. Install Caddy v2 via COPR EPEL 7 YUM repository

Install Caddy v2 via COPR EPEL 7 YUM repo ensuring to disable EPEL repo to prevent installing EPEL's Caddy v1.x version.

```
yum-config-manager --add-repo https://copr.fedorainfracloud.org/coprs/g/caddy/caddy/repo/epel-7/group_caddy-caddy-epel-7.repo
yum -y install caddy --disablerepo=epel
caddy trust
sed -i 's|:80|http://caddy.domain.com:81\n\nheader x-powered-by "caddy centminmod"\nheader vary Accept-Encoding\nencode gzip\n|' /etc/caddy/Caddyfile
mkdir -p /var/log/caddy
chown caddy:caddy /var/log/caddy
service caddy start
service caddy status
chkconfig caddy on
```

Copy Caddy default index.html to caddy-index.html file for both Caddy default public web root and Nginx site's default public web root and setup some command shortcuts for Caddy service commands like Centmin Mod Nginx's service commands have.

```
cp -a /usr/share/caddy/index.html /usr/share/caddy/caddy-index.html
cp -a /usr/share/caddy/index.html /home/nginx/domains/ngx.domain.com/public/caddy-index.html
echo "service caddy stop" >/usr/bin/caddystop ; chmod 700 /usr/bin/caddystop
echo "service caddy start" >/usr/bin/caddystart ; chmod 700 /usr/bin/caddystart
echo "service caddy status" >/usr/bin/caddystatus ; chmod 700 /usr/bin/caddystatus
echo "service caddy restart" >/usr/bin/caddyrestart ; chmod 700 /usr/bin/caddyrestart
echo "service caddy reload" >/usr/bin/caddyreload ; chmod 700 /usr/bin/caddyreload
```

Check Caddy logging and config

```
journalctl -u caddy --no-pager
curl -s localhost:2019/config/ | jq
```

### 5. Configure Caddy v2 Caddyfile non-HTTPS (port 81) & HTTPS (port 4444) local internal self-signed SSL certificates 

Backup default Caddyfile

```
cp -a /etc/caddy/Caddyfile /etc/caddy/Caddyfile-backup
```

Overwrite /etc/caddy/Caddyfile with below config using default Caddy public web root `/usr/share/caddy` just for testing purposes right now and configure non-HTTPS (port 81) & HTTPS (port 4444) local internal self-signed SSL certificates. For properly comparison with Nginx, try to keep equivalent response headers for both Caddy and Nginx as Caddy previously in 0.1x and 1.x releases have been known to have a performance drop as you add more HTTP headers to Caddy's responses.

```
{
    http_port   81
    https_port  4444
    #experimental_http3
}

http://caddy.domain.com:81 {

    header x-powered-by "caddy centminmod"
    header X-Content-Type-Options nosniff
    header X-XSS-Protection "1; mode=block"
    encode gzip
    
    # Set this path to your site's directory.
    root * /usr/share/caddy
    
    # Enable the static file server.
    file_server
    
    # Or serve a PHP site through php-fpm:
    # php_fastcgi localhost:9000
    log {
        output file /var/log/caddy/access_http.log {
            roll_size 100mb
            roll_keep 10
            roll_keep_for 720h
        }
        format single_field common_log
        level INFO
    }
}

https://caddy.domain.com:4444 {

    tls internal
    header x-powered-by "caddy centminmod"
    header X-Content-Type-Options nosniff
    header X-XSS-Protection "1; mode=block"
    encode gzip
    
    # Set this path to your site's directory.
    root * /usr/share/caddy
    
    # Enable the static file server.
    file_server
    
    # Or serve a PHP site through php-fpm:
    # php_fastcgi localhost:9000
    log {
        output file /var/log/caddy/access_https.log {
            roll_size 100mb
            roll_keep 10
            roll_keep_for 720h
        }
        format single_field common_log
        level INFO
    }
}
```

Restart Caddy server using previously created command line shortcuts

```
caddyrestart
caddystatus
```

### 6. CSF Firewall Configuration

Centmin Mod LEMP stack installs [CSF Firewall](https://centminmod.com/csf_firewall.html) as a wrapper to iptables so you need to configure the TCP/UDP ports for custom ports.

For TCP/UDP ports 80,443,81,4444 for IPv4 and IPv6 whitelisting. TCP inbound whitelisting already has port 80, 81, 443 added. UDP ports are if you want to test HTTP/3 over QUIC.

```
egrep '^TCP_|^TCP6_|^UDP_|^UDP6_' /etc/csf/csf.conf
sed -i "s/TCP_IN = \"/TCP_IN = \"4444,/g" /etc/csf/csf.conf
sed -i "s/TCP6_IN = \"/TCP6_IN = \"4444,/g" /etc/csf/csf.conf
sed -i "s/TCP_OUT = \"/TCP_OUT = \"4444,/g" /etc/csf/csf.conf
sed -i "s/TCP6_OUT = \"/TCP6_OUT = \"4444,/g" /etc/csf/csf.conf
sed -i "s/UDP_IN = \"/UDP_IN = \"80,443,81,4444,/g" /etc/csf/csf.conf
sed -i "s/UDP6_IN = \"/UDP6_IN = \"80,443,81,4444,/g" /etc/csf/csf.conf
sed -i "s/UDP_OUT = \"/UDP_OUT = \"80,443,81,4444,/g" /etc/csf/csf.conf
sed -i "s/UDP6_OUT = \"/UDP6_OUT = \"80,443,81,4444,/g" /etc/csf/csf.conf
egrep '^TCP_|^TCP6_|^UDP_|^UDP6_' /etc/csf/csf.conf
csf -ra
```

### 7. Verify Nginx & Caddy sites are working

Check Caddy and Nginx versions

```
caddy version
nginx -V
```

```
caddy version
v2.0.0 h1:pQSaIJGFluFvu8KDGDODV8u4/QRED/OPyIR+MWYYse8=
```

```
nginx -V
nginx version: nginx/1.17.10 (060520-051553-centos7-virtualbox-kvm-54698e1)
built by gcc 8.3.1 20190311 (Red Hat 8.3.1-3) (GCC) 
built with OpenSSL 1.1.1g  21 Apr 2020
TLS SNI support enabled
```
> configure arguments: --with-ld-opt='-Wl,-E -L/usr/local/zlib-cf/lib -L/usr/local/lib -ljemalloc -Wl,-z,relro -Wl,-rpath,/usr/local/zlib-cf/lib:/usr/local/lib -flto=2 -fuse-ld=gold' --with-cc-opt='-I/usr/local/zlib-cf/include -I/usr/local/include -m64 -march=native -DTCP_FASTOPEN=23 -g -O3 -fstack-protector-strong -flto=2 -fuse-ld=gold --param=ssp-buffer-size=4 -Wformat -Werror=format-security -Wimplicit-fallthrough=0 -fcode-hoisting -Wno-cast-function-type -Wno-format-extra-args -Wp,-D_FORTIFY_SOURCE=2' --sbin-path=/usr/local/sbin/nginx --conf-path=/usr/local/nginx/conf/nginx.conf --build=060520-051553-centos7-virtualbox-kvm-54698e1 --with-compat --with-http_stub_status_module --with-http_secure_link_module --with-libatomic --with-http_gzip_static_module --with-http_sub_module --with-http_addition_module --with-http_image_filter_module=dynamic --with-http_geoip_module --with-stream_geoip_module --with-stream_realip_module --with-stream_ssl_preread_module --with-threads --with-stream --with-stream_ssl_module --with-http_realip_module --add-dynamic-module=../ngx-fancyindex-0.4.2 --add-module=../ngx_cache_purge-2.5 --add-dynamic-module=../ngx_devel_kit-0.3.0 --add-dynamic-module=../set-misc-nginx-module-0.32 --add-dynamic-module=../echo-nginx-module-0.61 --add-module=../redis2-nginx-module-0.15 --add-module=../ngx_http_redis-0.3.7 --add-module=../memc-nginx-module-0.18 --add-module=../srcache-nginx-module-0.32rc1 --add-dynamic-module=../headers-more-nginx-module-0.33 --with-pcre-jit --with-zlib=../zlib-cloudflare-1.3.0 --with-http_ssl_module --with-http_v2_module --with-openssl=../openssl-1.1.1g --with-openssl-opt='enable-ec_nistp_64_gcc_128 enable-tls1_3 -fuse-ld=gold'

For Caddy HTTPS on port 4444 at `https://caddy.domain.com:4444/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" https://caddy.domain.com:4444/caddy-index.html -o /dev/null
```

For Nginx HTTPS on port 443 at `https://ngx.domain.com/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" https://ngx.domain.com/caddy-index.html -o /dev/null
```

For Caddy HTTPS on port 81 at `http://caddy.domain.com:81/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" http://caddy.domain.com:81/caddy-index.html -o /dev/null
```

For Nginx HTTPS on port 80 at `http://ngx.domain.com/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" http://ngx.domain.com/caddy-index.html -o /dev/null
```

Example output

For Caddy HTTPS on port 4444 at `https://caddy.domain.com:4444/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" https://caddy.domain.com:4444/caddy-index.html -o /dev/null
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Type: text/html; charset=utf-8
Etag: "q9xapl9fm"
Last-Modified: Wed, 06 May 2020 18:44:09 GMT
Server: Caddy
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
X-Powered-By: caddy centminmod
X-Xss-Protection: 1; mode=block
Date: Fri, 08 May 2020 19:09:02 GMT
Transfer-Encoding: chunked
```

For Nginx HTTPS on port 443 at `https://ngx.domain.com/caddy-index.html`

```
curl -skD - -H "Accept-Encoding: gzip" https://ngx.domain.com/caddy-index.html -o /dev/null                                              
HTTP/1.1 200 OK
Date: Fri, 08 May 2020 19:10:16 GMT
Content-Type: text/html; charset=utf-8
Last-Modified: Wed, 06 May 2020 18:44:09 GMT
Transfer-Encoding: chunked
Connection: keep-alive
Vary: Accept-Encoding
ETag: W/"5eb30579-2fc2"
Server: nginx centminmod
X-Powered-By: centminmod
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
Content-Encoding: gzip
```

## Caddy vs Centmin Mod Nginx HTTP/2 HTTPS Benchmarks

Centmin Mod LEMP stack already installs nghttp2 so has access to h2load HTTP/2 load testing tool.

On Virtualbox CentOS 7.8 guest OS on Windows 10 Pro laptop

```
lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                2
On-line CPU(s) list:   0,1
Thread(s) per core:    1
Core(s) per socket:    2
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 142
Model name:            Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz
Stepping:              9
CPU MHz:               2904.000
BogoMIPS:              5808.00
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              4096K
NUMA node0 CPU(s):     0,1
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc eagerfpu pni pclmulqdq ssse3 cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single fsgsbase avx2 invpcid rdseed clflushopt md_clear flush_l1d

free -mlt
              total        used        free      shared  buff/cache   available
Mem:           1837         434        1007          22         396        1184
Low:           1837         830        1007
High:             0           0           0
Swap:          2047           0        2047
Total:         3885         434        3055
```

h2load HTTP/2 HTTPS test for Caddy HTTPS on port 4444 at `https://caddy.domain.com:4444/caddy-index.html`

Caddy at 150 user concurrency

```
h2load -t1 -c150 -n1000 -m50 -H Accept-Encoding:gzip https://caddy.domain.com:4444/caddy-index.html
starting benchmark...
spawning thread #0: 150 total client(s). 1000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES256-GCM-SHA384
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done
progress: 70% done
progress: 80% done
progress: 90% done
progress: 100% done

finished in 1.04s, 959.57 req/s, 4.86MB/s
requests: 1000 total, 1000 started, 1000 done, 1000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 5.07MB (5311507) total, 35.41KB (36257) headers (space savings 86.67%), 5.00MB (5240000) data
                     min         max         mean         sd        +/- sd
time for request:   128.60ms    929.16ms    588.95ms    196.65ms    60.80%
time for connect:    84.69ms    580.50ms    282.73ms    125.63ms    78.67%
time to 1st byte:   213.30ms       1.03s    696.74ms    217.35ms    52.67%
req/s           :       5.83       32.80        8.22        4.19    92.67%
```

Caddy at 500 user concurrency

```
h2load -t1 -c500 -n2000 -m100 -H Accept-Encoding:gzip https://caddy.domain.com:4444/caddy-index.html
starting benchmark...
spawning thread #0: 500 total client(s). 2000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES256-GCM-SHA384
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done
progress: 70% done
progress: 80% done
progress: 90% done
progress: 100% done

finished in 2.02s, 990.03 req/s, 5.04MB/s
requests: 2000 total, 2000 started, 2000 done, 2000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 2000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 10.17MB (10667322) total, 103.34KB (105822) headers (space savings 80.55%), 9.99MB (10480000) data
                     min         max         mean         sd        +/- sd
time for request:   135.58ms       1.38s    718.29ms    366.74ms    53.25%
time for connect:   538.04ms    778.90ms    666.76ms     62.41ms    61.00%
time to 1st byte:   711.60ms       1.98s       1.36s    395.53ms    55.20%
req/s           :       2.00        5.61        3.15        1.01    64.60%
```

Caddy at 1000 user concurrency with 10k requests where only 68.89% of requests succeeded and 31.11% of requests failed which artifically inflate requests/second throughput numbers

```
h2load -t1 -c1000 -n10000 -m100 -H Accept-Encoding:gzip https://caddy.domain.com:4444/caddy-index.html 
starting benchmark...
spawning thread #0: 1000 total client(s). 10000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES256-GCM-SHA384
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done

finished in 6.57s, 1049.00 req/s, 5.30MB/s
requests: 10000 total, 10000 started, 6889 done, 6889 succeeded, 3111 failed, 3111 errored, 0 timeout
status codes: 6945 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 34.84MB (36530787) total, 189.12KB (193656) headers (space savings 89.75%), 34.43MB (36098360) data
                     min         max         mean         sd        +/- sd
time for request:   224.18ms       5.31s       2.17s       1.41s    51.46%
time for connect:   480.14ms       2.38s       1.15s    536.69ms    69.60%
time to 1st byte:   965.65ms       6.53s       3.34s       1.85s    52.83%
req/s           :       0.00       10.35        2.89        2.99    82.60%
```

h2load HTTP/2 HTTPS test for Nginx HTTPS on port 443 at `https://ngx.domain.com/caddy-index.html`

Nginx at 150 user concurrency

```
h2load -t1 -c150 -n1000 -m50 -H Accept-Encoding:gzip https://ngx.domain.com/caddy-index.html
starting benchmark...
spawning thread #0: 150 total client(s). 1000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES128-GCM-SHA256
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done
progress: 70% done
progress: 80% done
progress: 90% done
progress: 100% done

finished in 449.49ms, 2224.74 req/s, 11.32MB/s
requests: 1000 total, 1000 started, 1000 done, 1000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 1000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 5.09MB (5336350) total, 202.15KB (207000) headers (space savings 26.86%), 4.87MB (5104000) data
                     min         max         mean         sd        +/- sd
time for request:     9.33ms    276.56ms    115.17ms     71.07ms    61.90%
time for connect:    86.76ms    377.81ms    183.10ms     75.91ms    82.00%
time to 1st byte:   158.04ms    440.22ms    300.22ms     84.40ms    60.67%
req/s           :      13.71       44.23       24.21        8.72    72.67%
```

Nginx at 500 user concurrency

```
h2load -t1 -c500 -n2000 -m100 -H Accept-Encoding:gzip https://ngx.domain.com/caddy-index.html
starting benchmark...
spawning thread #0: 500 total client(s). 2000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES128-GCM-SHA256
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done
progress: 70% done
progress: 80% done
progress: 90% done
progress: 100% done

finished in 1.25s, 1600.52 req/s, 8.15MB/s
requests: 2000 total, 2000 started, 2000 done, 2000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 2000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 10.19MB (10682500) total, 404.30KB (414000) headers (space savings 26.86%), 9.74MB (10208000) data
                     min         max         mean         sd        +/- sd
time for request:    82.91ms    583.83ms    310.77ms    160.24ms    51.75%
time for connect:   459.89ms    664.50ms    552.15ms     36.49ms    89.00%
time to 1st byte:   583.80ms       1.23s    861.70ms    189.58ms    55.60%
req/s           :       3.25        6.85        4.86        1.07    56.00%
```

Nginx at 1000 user concurrency with 10k requests all succeeded

```
h2load -t1 -c1000 -n10000 -m100 -H Accept-Encoding:gzip https://ngx.domain.com/caddy-index.html
starting benchmark...
spawning thread #0: 1000 total client(s). 10000 total requests
TLS Protocol: TLSv1.2
Cipher: ECDHE-ECDSA-AES128-GCM-SHA256
Server Temp Key: ECDH P-256 256 bits
Application protocol: h2
progress: 10% done
progress: 20% done
progress: 30% done
progress: 40% done
progress: 50% done
progress: 60% done
progress: 70% done
progress: 80% done
progress: 90% done
progress: 100% done

finished in 5.23s, 1912.05 req/s, 9.73MB/s
requests: 10000 total, 10000 started, 10000 done, 10000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 10000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 50.87MB (53339000) total, 1.97MB (2070000) headers (space savings 26.86%), 48.68MB (51040000) data
                     min         max         mean         sd        +/- sd
time for request:     5.25ms       2.53s       1.24s    606.34ms    67.13%
time for connect:   603.67ms       3.59s       1.75s       1.19s    68.00%
time to 1st byte:   949.61ms       5.16s       2.98s       1.20s    56.40%
req/s           :       1.93       10.52        4.08        2.06    80.80%
```

# Nginx vhost config file

`nv` command line auto generated Nginx HTTP/2 HTTPS vhost config file at `/usr/local/nginx/conf/conf.d/ngx.domain.com.ssl.conf`

```
# Centmin Mod Getting Started Guide
# must read http://centminmod.com/getstarted.html
# For HTTP/2 SSL Setup
# read http://centminmod.com/nginx_configure_https_ssl_spdy.html

# redirect from www to non-www  forced SSL
# uncomment, save file and restart Nginx to enable
# if unsure use return 302 before using return 301
# server {
#       listen   80;
#       server_name ngx.domain.com www.ngx.domain.com;
#       return 302 https://$server_name$request_uri;
# }

server {
  listen 443 ssl http2 reuseport;
  server_name ngx.domain.com www.ngx.domain.com;

  ssl_dhparam /usr/local/nginx/conf/ssl/ngx.domain.com/dhparam.pem;
  ssl_certificate      /usr/local/nginx/conf/ssl/ngx.domain.com/ngx.domain.com.crt;
  ssl_certificate_key  /usr/local/nginx/conf/ssl/ngx.domain.com/ngx.domain.com.key;
  include /usr/local/nginx/conf/ssl_include.conf;

  # cloudflare authenticated origin pull cert community.centminmod.com/threads/13847/
  #ssl_client_certificate /usr/local/nginx/conf/ssl/cloudflare/ngx.domain.com/origin.crt;
  #ssl_verify_client on;
  http2_max_field_size 16k;
  http2_max_header_size 32k;
  http2_max_requests 5000;
  # mozilla recommended
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS;
  ssl_prefer_server_ciphers   on;
  #add_header Alternate-Protocol  443:npn-spdy/3;

  # before enabling HSTS line below read centminmod.com/nginx_domain_dns_setup.html#hsts
  #add_header Strict-Transport-Security "max-age=31536000; includeSubdomains;";
  #add_header X-Frame-Options SAMEORIGIN;
  add_header X-Xss-Protection "1; mode=block" always;
  add_header X-Content-Type-Options "nosniff" always;
  #add_header Referrer-Policy "strict-origin-when-cross-origin";
  #add_header Feature-Policy "accelerometer 'none'; camera 'none'; geolocation 'none'; gyroscope 'none'; magnetometer 'none'; microphone 'none'; payment 'none'; usb 'none'";
  #spdy_headers_comp 5;
  ssl_buffer_size 1369;
  ssl_session_tickets on;
  
  # enable ocsp stapling
  #resolver 8.8.8.8 8.8.4.4 1.1.1.1 1.0.0.1 valid=10m;
  #resolver_timeout 10s;
  #ssl_stapling on;
  #ssl_stapling_verify on;
  #ssl_trusted_certificate /usr/local/nginx/conf/ssl/ngx.domain.com/ngx.domain.com-trusted.crt;  

# ngx_pagespeed & ngx_pagespeed handler
#include /usr/local/nginx/conf/pagespeed.conf;
#include /usr/local/nginx/conf/pagespeedhandler.conf;
#include /usr/local/nginx/conf/pagespeedstatslog.conf;

  # limit_conn limit_per_ip 16;
  # ssi  on;

  access_log /home/nginx/domains/ngx.domain.com/log/access.log combined buffer=256k flush=5m;
  error_log /home/nginx/domains/ngx.domain.com/log/error.log;

  include /usr/local/nginx/conf/autoprotect/ngx.domain.com/autoprotect-ngx.domain.com.conf;
  root /home/nginx/domains/ngx.domain.com/public;
  # uncomment cloudflare.conf include if using cloudflare for
  # server and/or vhost site
  #include /usr/local/nginx/conf/cloudflare.conf;
  include /usr/local/nginx/conf/503include-main.conf;

  location / {
  include /usr/local/nginx/conf/503include-only.conf;

# block common exploits, sql injections etc
#include /usr/local/nginx/conf/block.conf;

  # Enables directory listings when index file not found
  #autoindex  on;

  # Shows file listing times as local time
  #autoindex_localtime on;

  # Wordpress Permalinks example
  #try_files $uri $uri/ /index.php?q=$uri&$args;

  }

  include /usr/local/nginx/conf/pre-staticfiles-local-ngx.domain.com.conf;
  include /usr/local/nginx/conf/pre-staticfiles-global.conf;
  include /usr/local/nginx/conf/staticfiles.conf;
  include /usr/local/nginx/conf/php.conf;
  
  include /usr/local/nginx/conf/drop.conf;
  #include /usr/local/nginx/conf/errorpage.conf;
  include /usr/local/nginx/conf/vts_server.conf;
}
```