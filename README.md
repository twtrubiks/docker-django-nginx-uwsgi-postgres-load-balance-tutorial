# docker-django-nginx-uwsgi-postgres-load-balance-tutorial

實戰 Docker + Django + Nginx + uWSGI + Postgres - **Load Balance**  📝

這篇文章主要接續上一篇

[Docker + Django + Nginx + uWSGI + Postgres 基本教學 - 從無到有 ( Docker + Django + Nginx + uWSGI + Postgres Tutorial )](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial)

繼續介紹，這次的重點會放在 Nginx 的 **Load Balance** 上 :blush:

* [Youtube Tutorial PART 1 - 正向代理器  VS 反向代理器 - 簡介](https://youtu.be/R2I8BBXnJJ8)
* [Youtube Tutorial PART 2 - Docker + Django + Nginx + Load Balance - 概念](https://youtu.be/W4EMOO-THGs)
* [Youtube Tutorial PART 3 - Docker + Django + Nginx + Load Balance - 實戰](https://youtu.be/ChK8MtQUDf0)
* [Youtube Tutorial PART 4 - Django + Nginx + Load Balance - Docker scale](https://youtu.be/w83_lV5tORI)

## 簡介

在開始介紹 Nginx 的 Load Balance 之前，我會先帶大家了解一些名詞。

還記得在 [上一篇](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial) 教學中，提到 Nginx 可以設定 反向代理器 嗎？

那時候沒有詳細解釋什麼是 反向代理器 ，這邊我將先帶大家了解

**正向代理器** 以及 **反向代理器** 的差別 :notebook:

### 正向代理器  VS 反向代理器

#### 正向代理器

正向代理器是一個位於使用者端以及目標 Server 之間的 **代理伺服器 ( Proxy Server )**，

透過代理伺服器收發 Request / Respond 資料，

![](https://i.imgur.com/zWIk8QW.png)

如上圖，X 無法直接和 Z 請求資料，但是 Y 可以和 Z 請求資料 ( 於是 X 透過 Y 向 Z 請求資料 )，

Y 得到資料後，再將資料回傳給 X，這樣 X 就能順利得到 Z 的資料了 。

通常正向代理器必須額外做一些設定才可以使用。

正向代理器，Client 和 Proxy 可視為同一個 LAN ，對 Server 是隱藏的。

![](https://i.imgur.com/MJ0Pr2q.png)

溫馨小提醒 :heart:

* LAN ( Local Area Network )，又稱 區域網路

* WAN ( Wide Area Network )，又稱 廣域網路

* WLAN ( Wireless LAN )，又稱 無線區域網路

正向代理器是代理使用者端（ Client 端 ），替使用者端（ Client 端 ）收發 Request / Respond，

讓真正的使用者對伺服器端（ Server 端 ）隱藏。

可能你自己也有使用過（ 或正在使用 :grin: ）正向代理器 ，只是你不知道而以，像最常見的就是 **翻牆 ( VPN )** :smirk:

今天主角是阿鬼，阿鬼無法瀏覽 FB，但可以瀏覽 Server A，而阿鬼發現 Server A 可以訪問 FB，

於是阿鬼就透過 Server A 去瀏覽 FB，整個流程是，阿鬼透過 Server A 去瀏覽 FB，Server A 收到

來自阿鬼的 Request 時，去瀏覽 FB，FB 再把 Response 回傳給 Server A，Server A 再把收到的

Response 回傳給阿鬼，這樣阿鬼就能順利得到 FB 的資料，這就是透過 Proxy ( 正向代理器 )。

#### 反向代理器

反向代理器則相反過來，對於使用者端來說，反向代理器就好像是目標 Server，

使用者端也不需要做額外的設定，

![](https://i.imgur.com/kUBGgKz.png)

如上圖，雖然整個流程還是 X->Y->Z，但因為不用特別設定，所以使用者端會感覺好像直接是 X->Z，

反向代理器，Proxy 和 Server 屬於一個 LAN，對 Client 隱藏

![](https://i.imgur.com/j6wnvnt.png)

反向代理器是代理伺服器端（ Server 端 ），替伺服器端（ Server 端 ）收發 Request / Respond，

讓真正的伺服器（ Server 端 ）對使用者隱藏。

本次的主角 **Load Balance** 就是反向代理器的一種應用 :open_mouth:

## 教學

解釋了那麼多，我們要開始實戰 Nginx 的 Load Balance，範例一樣是使用 [上一篇](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial) 做修改，

因為要做 Load Balance，所以增加一台主機（ service ），名稱命名為 [api2](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/tree/master/api2)，基本上內容

都和 [api](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/tree/master/api) 差不多，只是修改了一些名稱（ 方便區別而已 ），可參考 [docker-compose.yml](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/blob/master/docker-compose.yml)。

整體的 Load Balance 的概念可以參考下圖

![](https://i.imgur.com/RxUtPiz.png)

這樣有什麼好處呢 ？

最直接的好處就是，如果一台掛了，整個系統還是不會死，

因為其他的 Server 還是正常工作:relaxed:

再來就是我們的主角 [my_nginx.conf](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/blob/master/nginx/my_nginx.conf)，其實主要也是修改這邊的設定而已 :laughing:

```config
# the upstream component nginx needs to connect to
upstream uwsgi {
    # server api:8001; # use TCP
    server unix:/docker_api/app.sock weight=4 ; # for a file socket
    server unix:/docker_api2/app2.sock  weight=6; # for a file socket
}

# configuration of the server
server {
    # the port your site will be served on
    listen    80;
    # index  index.html;
    # the domain name it will serve for
    # substitute your machine's IP address or FQDN
    server_name  twtrubiks.com www.twtrubiks.com;
    charset     utf-8;

    client_max_body_size 75M;   # adjust to taste

    # Django media
    # location /media  {
    #     alias /docker_api/static/media;  # your Django project's media files - amend as required
    # }

    location /static {
        alias /docker_api/static; # your Django project's static files - amend as required
    }

    location / {
        uwsgi_pass  uwsgi;
        include     /etc/nginx/uwsgi_params; # the uwsgi_params file you installed
    }

}
```

其實設定並不難，只是在 `upstream` 裡面增加一台 Server 而已，後面的 `weight` 則代表權重，

Nginx 預設會使用 round-robin 演算法來實作 Load balance。

### Load balancing methods

Nginx 主要的 Load balance 有三種

****round-robin — requests to the application servers are distributed in a round-robin fashion****

使用輪詢的方式，也可以加上 `weight` 權重，讓比較強的 Server 承受比較大的壓力（ `weight` 設定高一點）。

****least-connected — next request is assigned to the server with the least number of active connections****

此種方式，Nginx 會將請求給負載 Loading 較輕的 Server，避免負載（A Server） Loading 已經很大了，

還將請求交給 A Server 處理。

****ip-hash — a hash-function is used to determine what server should be selected for the next request (based on the client's IP address)****

不管是 round-robin 還是 least-connected 的 Load balance ，每個 User 送出的請求都可能會被分到不同的

Server，並不能保證同一個 User 每次的請求都一定會連到同一台 Server，
因為這個原因，於是有了 ip-hash，

透過這個方法，會使用 ip 的方式將同一個 User 的請求都導到同一台
Server （除非這台 Server 掛了 :sob: ）

介紹完了 Nginx 主要的三種 Load balance 演算法，至於要使用哪一種，就看你自己的需求了 :grinning:

更多範例以及詳細的說明可參考官網

 ( 建議閱讀 ，雖然我有幫大家整理重點出來了 :grinning:  )

[http://nginx.org/en/docs/http/load_balancing.html](http://nginx.org/en/docs/http/load_balancing.html)

## 執行步驟

直接執行 `docker-compose up` 見證奇蹟

細部的圖就不再貼一次了，請參考 [上一篇](https://github.com/twtrubiks/docker-django-nginx-uswgi-postgres-tutorial)

會發現我們多了一台 `api2`

![](https://i.imgur.com/KhROEky.png)

接著開啟另一個 terminal，進入 `api` ( Django + uWSGI ) 的容器，

指令可參考之前的 [docker-tutorial-指令介紹](https://github.com/twtrubiks/docker-tutorial#指令介紹)，

```cmd
docker exec -it <Container ID> bash
```

執行 migrate

```cmd
python manage.py makemigrations musics
python manage.py migrate
```

![](https://i.imgur.com/nQDwP7e.png)

建立 user

```cmd
python manage.py createsuperuser
```

![](https://i.imgur.com/saaDD7R.png)

最後再執行

```cmd
python manage.py collectstatic
```

![](https://i.imgur.com/1EKNNOx.png)

將 Django 中的 static files 收集起來，變成 static folder，

我們不需要再進入 `api2` 的 container 執行剛剛的動作，因為我們都是連同一個 db，

到這裡基本上就是完工了:smile:

瀏覽 [http://localhost:8080/api/music/](http://localhost:8080/api/music/) 確認是否正常。

## 執行畫面

瀏覽 [http://localhost:8080/api/music/](http://localhost:8080/api/music/)

![](https://i.imgur.com/jl43jST.png)

![](https://i.imgur.com/Fw6LjbE.png)

那要怎麼確定真的有 Load balance 呢？

可以在頁面上瘋狂點擊 F5，然後我們觀察 terminal，你會發現

![](https://i.imgur.com/7xuFXw5.png)

有時候是 `api`，有時候是 `api2`，還記得前面設定的 `weight` 權重，

`api` 和 `api2` 是 4:6，4:6 是什麼意思呢:question::question:

假如現在有 10 個 Request 進來，4 個 Request 會交給 `api` 這個 Server 處理，

而剩下的 6 個 Request 則會交給 `api2` 這個 Server 處理 :+1:

`weight` 權重可以依照自己的需求設定。

接著再來模擬一台 Server 掛了的時候，如下圖，Server 因為某種原因掛了，

![](https://i.imgur.com/fRa1Q9t.png)

先透過 stop 指令將 `api` container 停止

( 模擬 `api` 這台 Server 掛了 )

```cmd
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

![](https://i.imgur.com/LkoQeDc.png)

![](https://i.imgur.com/lHmMPUu.png)

然後再繼續瀏覽 [http://localhost:8080/api/music/](http://localhost:8080/api/music/)，你會發現網站正常 work ( 確實沒有掛掉 )，

而且 terminal 現在都只會輸出 `api2`，

![](https://i.imgur.com/RTdzQqX.png)

因為我們已經將 `api` 停止了（模擬機器意外掛了），

有沒有很酷 :satisfied: 這就是最簡單的 Load balance :flushed:

範例是總共有兩台 server ( `api` 以及 `api2` )， 你也可以自己多新增幾台來玩玩看。

## 其他的負載平衡

除了 Nginx 的 Load banlance 之外，假設你的系統更龐大且有規模，

可以考慮使用更專業的負載平衡，像是 [HAProxy](http://www.haproxy.org/) ( High Availability Proxy ) ，

如果要專注在 Load banlance 上，選擇 [HAProxy](http://www.haproxy.org/) 應該會比 Nginx 的

Load banlance 還來的好。

如果對 HAProxy 有興趣，可參考 [Docker Swarm + HAProxy](https://github.com/twtrubiks/docker-swarm-tutorial#docker-swarm--haproxy) :sunglasses:

## 後記：

這次主要介紹了 Nginx 的 Load Banlance 給大家，建議大家可以動手玩玩看，整體會比較有感覺 :smiley:

這次嘗試參考網路上的圖自己簡單的畫一遍，希望對大家多多少少有幫助:flushed:

如果有在玩像是 AWS 的人，可以知道還有一種東西更狂，就是 異地同步備份，大家有興趣自行研究:sunglasses:

如果有任何講錯的地方，請麻煩大家和我說，我會再修改，感謝各位的閱讀:v:

下一步可以試試看用 Docker scale 的方法來完成( 更好的寫法 )，可參考 [better](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/tree/better) 分支。

如果意猶未盡，延伸閱讀 :satisfied:

* [Docker Swarm 基本教學 - 從無到有 Docker-Swarm-Beginners-Guide📝](https://github.com/twtrubiks/docker-swarm-tutorial)

## 執行環境

* Mac
* Python 3.6.2
* windows 10

## Reference

* [https://docs.docker.com/](https://docs.docker.com/)
* [nginx-load_balancing](http://nginx.org/en/docs/http/load_balancing.html)

## Donation

文章都是我自己研究內化後原創，如果有幫助到您，也想鼓勵我的話，歡迎請我喝一杯咖啡:laughing:

![alt tag](https://i.imgur.com/LRct9xa.png)

[贊助者付款](https://payment.opay.tw/Broadcaster/Donate/9E47FDEF85ABE383A0F5FC6A218606F8)

## License

MIT license
