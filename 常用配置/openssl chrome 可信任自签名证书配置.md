## 一. 安装openssl
## 二. 创建CA根证书
1. 创建CA根证书key
```sh

openssl genrsa -out ca.key 2048

```

2. 创建CA证书请求（中间文件）
```sh

openssl req -new -key ca.key -out ca.csr

# Common Name 为域名

```

3. 生成CA证书
```sh

openssl x509 -req -in ca.csr -extensions v3_ca -signkey ca.key -out ca.crt

```

## 三. 创建服务器证书

1. 创建服务器证书key
```sh

openssl genrsa -out server.key 2048

```

2. 创建服务器证书请求
```sh

openssl req -new -key server.key -out server.csr

```

3. 向CA根证书申请创建证书
```sh

# 创建ext.ini文件，防止chrome报错NET::ERR_CERT_COMMON_NAME_INVALID
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 192.168.31.147 #设置为服务器的IP
# DNS.1 = www.mtiisl.cn 设置为服务器的域名

```

```sh

openssl x509 -days 365 -req -in server.csr -CAkey ca.key -CA ca.crt -CAcreateserial -extfile ./ext.ini -out server.crt

```

## 四. Nginx安装服务器证书
```sh
# Nginx配置文件添加ssl配置
# 注意：443配置放在80配置之前
server {
    listen 443 ssl;

    # 填写绑定证书的域名或IP
    server_name 192.168.31.147;

    # 证书文件名称
    ssl_certificate  /usr/share/nginx/html/server.crt;

    # 私钥文件名称
    ssl_certificate_key /usr/share/nginx/html/server.key;
    ssl_session_timeout 5m;

    # 请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    ...

}

```

## 五. chrome所在客户端安装CA根证书