# proxy configuration



```text
location / {
    # 配置反向代理到本机的8080端口
    proxy_pass http://127.0.0.1:8080;

    # 配置请求客户端真实的 Host 信息
    proxy_set_header Host $http_host;
    # 配置请求用户真实的IP信息
    proxy_set_header X-Real-IP $remote_addr;

    # 连接超时时间为30秒
    proxy_connect_timeout 30;
    # 读取响应超时时间为60秒
    proxy_send_timeout 60;
    # 发送请求超时时间为60秒
    proxy_read_timeout 60;

    # 开启代理缓冲区
    proxy_buffering on;
    # 响应头的缓冲区设为32k
    proxy_buffer_size 32k;
    # 网页内容缓冲区个数为4，单个大小为128k
    proxy_buffers 4 128k;
    proxy_busy_buffers_size 256k;
    # 缓冲区临时文件最大为 256k
    proxy_max_temp_file_size 256k;

```

