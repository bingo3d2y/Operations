# upstream（未完成）



```text
set_real_ip_from  192.168.9.91;
real_ip_header    X-Forwarded-For;


---
 upstream robot {
        server 192.168.4.104:8080 max_fails=0 fail_timeout=10s;
        server 192.168.4.105:8080 backup;
        check interval=3000 rise=5 fall=10 timeout=1000 type=http;
        check_http_send "HEAD /robot/version.xml HTTP/1.0\r\n\r\n";
        check_http_expect_alive http_2xx http_3xx http_4xx;
}

```

