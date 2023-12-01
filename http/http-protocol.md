A HTTPS request includes 5 stages:
- DNS resolving
- TCP handshaking
- SSL handShaking
- Server resolving
- Content transferring

So a HTTPS request have 5 RRT(Round-Trip Time)

We can use the command below to get these 5 time:
```bash
curl -w '\n dns_resolve_time=%{time_namelookup}\n tpc_connect_time=%{time_connect}\n ssl_time_appconnect=%{time_appconnect}\n time_redirect=%{time_redirect}\n time_pretransfer=%{time_pretransfer}\n time_starttransfer=%{time_starttransfer}\n time_total=%{time_total}\n' -o /dev/null -s 'https://www.thebyte.com.cn/'
```

其中, Server resolving time = time_starttransfer - time_pretransfer
TTFB = time_starttransfer - time_appconnect