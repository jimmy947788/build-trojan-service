# build-trojan-service

用 [torjan-go](https://github.com/p4gefau1t/trojan-go) 專案來建立翻牆節點

## Trojan Server

  > 這邊 Server 使用ubuntu，所以採用linux 指令

### Trojan 下載

1. 下載 [trojan-go-linux-amd64.zip](https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip)

    ```bash
    mkdir ~/trojan-go && cd ~/trojan-go
    wget https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip
    unzip trojan-go-linux-amd64.zip #解壓縮
    ```
    
    ```bash
    sudo cp trojan-go /usr/bin/      # 把程式丟到bin就不用在指定路徑了
    sudo mkdir -p  /etc/trojan-go  # 之後要放config和 cert用
    ```

  1. Server設定檔 [config.json](./trojan-server-config.json)

      ```json
      {
        "run_type": "server",
        "local_addr": "0.0.0.0",
        "local_port": 443,
        "remote_addr": "127.0.0.1",  /*就是讓人用瀏覽器打開可以看到網頁騙過去我們是web server*/
        "remote_port": 5000, /*這裡適用docker 跑一個簡單的flask  http://127.0.0.1:5000/ */
        "password": [
          "PASSWORD"  /* client 連線驗證用 */
        ],
        "ssl": {
          "cert": "/etc/trojan-go/server.cer",
          "key": "/etc/trojan-go/server.key"
        }
      }
      ```

2. 啟動 trojan

    ```bash
    trojan-go -config /etc/trojan-go/config.json
    ```

3. 建立[trojan-go.service](./trojan-go.service)

    ```ini
    # trojan-go.service 定義內容
    [Unit]
    Description=Trojan-Go - An unidentifiable mechanism that helps you bypass GFW
    Documentation=https://github.com/p4gefau1t/trojan-go
    After=network.target nss-lookup.target
    Wants=network-online.target

    [Service]
    Type=simple
    User=root
    ExecStart=/usr/bin/trojan-go -config /etc/trojan-go/config.json #這邊和啟動指令參數一樣
    Restart=on-failure
    RestartSec=10
    RestartPreventExitStatus=23

    [Install]
    WantedBy=multi-user.target​
    ```

    > 檔案路徑: /etc/systemd/system/trojan-go.service
    >

4. 啟動 trojan-go.service
   
    ```bash
    $ sudo systemctl daemon-reload # 如果第一次新增 trojan-go.service 就要執行 reload
    $ sudo systemctl enable trojan-go
    $ sudo systemctl start trojan-go
    $ sudo systemctl status trojan-go  # 檢查啓動狀態是否成功
    ```

### 申請憑證

1. 安裝 `acme.sh`

```bash
wget -qO- get.acme.sh | bash 
source ~/.bashrc
```

2. 设置 domin DNS API KEY

```bash
# Cloudflare 
$ export CF_Key="xxxxxx"
$ export CF_Email="xxxxxx"

# GoldDady 
$ export GD_Key="xxxxxx"
$ export GD_Secret="xxxxxx"
```

3. 申请 ECC 证书

先註冊acme帳號

```bash
acme.sh  --register-account  -m jimmy.wu@daedalus.cc --server zerossl
```

> `--server` 可以不用指定，這邊是註冊順便指定 *zerossl*
> acme也有提供切換 ca 機構的指令參數

acme.sh 实现了 acme 协议支持的所有验证协议，一般有两种方式验证，http 和 dns 验证。由于 Wildcard 证书验证只支持 dns 验证，不支持 http 验证，所以要使用 dns api 模式。推荐申请 ECC 证书。

```bash
$ acme.sh --issue --dns dns_gd -d <domain> -k ec-256
# dns:
# golddady -> dns_gd 
# Cloudflare -> dns_cf 
```

如果預設CA無法頒發，可以切換下列CA

```bash
acme.sh --set-default-ca --server letsencrypt #切换 Let’s Encrypt
acme.sh --set-default-ca --server buypass    #切换 Buypass
acme.sh --set-default-ca --server zerossl #切换 ZeroSSL
```

4. 安裝憑證

```bash
acme.sh --installcert -d <domain>  --fullchain-file /etc/trojan-go/server.cer --key-file /etc/trojan-go/server.key --ecc
acme.sh --remove -d <domain> --ecc #移除憑證
```
