此份 Repo 是台科 104 第二學年度，CS5124701 巨量資料與分析的課堂作業一。作業一的要求就是寫一隻應用到 Twitter 或是 Github 的爬蟲，並把爬下的資料格式化成 json。在 3/29 的這週，教了 ELK Stack 設定方法，所以 Crawler 程式並沒有多做特別設定，直接套用課堂簡報上的 Logstash Twitter 樣板。

以下記錄一下 ELK 安裝筆記，與課堂上略有不同的部分。說明一下，因為主力機 MBA 的儲存空間快炸了，所以我在 AWS 上開了一台 t2.medium 的機器當做跑 Spark 以及 ELK Stack 的平臺。在上週的課程簡報中 ELK 幾乎都是以下載 binary zip 包的方式設定，因為習慣用 apt 之流等套件管理程式，裝 ELK 相關設定檔也跟 binary zip 不太一樣。

安裝過程主要參考[數位海的這篇][1]，還有上課簡報。數位海原本這篇超詳細的，可以交互對照一下。

[1]: https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-elk-stack-on-ubuntu-14-04

## 安裝 Java 8

```
sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update
sudo apt-get -y install oracle-java8-installer
```

## 安裝 ElasticSearch

```
wget -qO - https://packages.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb http://packages.elastic.co/elasticsearch/2.x/debian stable main" | sudo tee -a /etc/apt/sources.list.d/elasticsearch-2.x.list
sudo apt-get update
sudo apt-get -y install elasticsearch
```

到這邊裝完，然後編輯 ElasticSearch 設定檔：

```
sudo vi /etc/elasticsearch/elasticsearch.yml
```

把 network.host 改成 0.0.0.0，讓外網也可以存取。
![](http://i.imgur.com/dlKuGDm.png)

重啟 elasticsearch 的服務：

```
sudo service elasticsearch restart
```

把它加入開機自啟動（Optional）

```
sudo update-rc.d elasticsearch defaults 95 10
```

ElasticSearch binary 檔的位置在 `/usr/share/elasticserach/bin`。

安裝 head plugin：

```
sudo /usr/share/elasticserach/bin/plugin install mobz/elasticsearch-head
```

把 iptable 打開

```
sudo iptables -A INPUT -m tcp -p tcp --dport 9200 -j ACCEPT
sudo iptables -A INPUT -m udp -p udp --dport 9200 -j ACCEPT
```

記得在 EC2 打開 9200 port，這時候開啟瀏覽器，進到

http://YOUR_IP_ADDRESS:9200/_plugin/head

就有一個簡單的圖形界面啦。

## 安裝 Kibana

```
echo "deb http://packages.elastic.co/kibana/4.4/debian stable main" | sudo tee -a /etc/apt/sources.list.d/kibana-4.4.x.list
sudo apt-get update
sudo apt-get -y install kibana
```

到這裡裝完，然後編輯 kibana 的設定檔：

```
sudo vi /opt/kibana/config/kibana.yml
```

把 server.host 從 0.0.0.0 改成 localhost，因為稍後會裝 nginx 來做我們的反向代理。就像這樣：

```
server.host: "localhost"
```

加入開機啟動，然後啟動服務：

```
sudo update-rc.d kibana defaults 96 9
sudo service kibana start
```

安裝 nginx 啥的

```
sudo apt-get install nginx apache2-utils
```

建立一個 kibanaadmin 的認證使用者（名稱可換）

```
sudo htpasswd -c /etc/nginx/htpasswd.users kibanaadmin
```

編輯 nginx 設定檔

```
sudo vi /etc/nginx/sites-available/default
```

換成下面這個
```nginx
server {
    listen 80;

    server_name example.com;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

`example.com` 的地方，看你要換成 Instance 的 IP 或是 domain name 都可以。EC2 可以換成 public dns name。

重啟 nginx

```
sudo service nginx restart
```

## 安裝 Logstash

```
echo 'deb http://packages.elastic.co/logstash/2.2/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash-2.2.x.list
sudo apt-get update
sudo apt-get install logstash
```

中間數位海插了一段設定 SSL 的，因為沒有要用就跳過這部分。

Logstash 安裝完的 binary 會在 `/opt/logstash/bin` 。

## 設定 Logstash

```
sudo vim /etc/logstash/conf.d/02-twitter.conf
```

填入以下內容

```logstash
input {
    twitter {
        consumer_key => ""
        consumer_secret => ""
        oauth_token => ""
        oauth_token_secret => ""
        keywords => ["beauty"]
        languages => ["en"]
        full_tweet => true
    }

}
output {
    elasticsearch {
        index => "twitter"
    }
}
```

oauth 那些 key 就填自己申請的，keywords 自由改。
存檔離開

然後跑

```
sudo service logstash configtest
```

看一下設定檔有沒有寫錯， 然後重啟 logstash 服務，並加入自啟動(Optional)

```
sudo service logstash restart
sudo update-rc.d logstash defaults 96 9
```

原則上這時候登入 kibana 或是在 head 儀表板就可以看到記錄數一直增多了。太神啦！

## 匯出資料

我是用 elasticsearch-dump 套件，記得先裝好 node v1.0 以上版本
https://github.com/taskrabbit/elasticsearch-dump

然後無腦一行匯出：

```
elasticdump --input=http://localhost:9200/twitter --output=twitter.json
```

## 上傳

之前 Github 推了 LFS(Large File Storage)，json 雖然純文字檔很好壓縮，但無論怎麽壓還是有點難推進到 100 MB ，也就是 Github 支援的最大容量，以內。勢必就要來用下 git-lfs 了。

![](http://i.imgur.com/ypwyyN4.png)

（完）
