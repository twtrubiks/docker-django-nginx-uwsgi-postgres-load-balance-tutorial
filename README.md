# docker-django-nginx-uwsgi-postgres-load-balance-tutorial

實戰 Docker + Django + Nginx + uWSGI + Postgres - **Load Balance**  📝

這篇文章主要是用 Docker 中的 scale 指令來完成 **Load Balance** （ 更好的寫法 ）

這邊要感謝 [cropse](https://github.com/cropse) 的 PR :+1:，可參考 [issues-1](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/issues/1) 以及 [pull-2](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/pull/2)

* [Youtube Tutorial PART 4 - Django + Nginx + Load Balance - Docker scale](https://youtu.be/w83_lV5tORI)

## 教學

請先將根目錄資料夾改名，在這裡我改名為 `demo` ( 後面會說明原因 )，

要修改三個地方，分別是 [uwsgi.ini](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/blob/better/api/uwsgi.ini) , [my_nginx.conf](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/blob/better/nginx/my_nginx.conf) , [docker-compose.yml](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial/blob/better/docker-compose.yml)

***uwsgi.ini***

使用 TCP socket

```ini
# use protocol uwsgi, port 8001 ,  TCP port socket
socket=:8001
```

如果不了解 TCP socket，請自行 google ，要解釋感覺麻煩:sweat:

***my_nginx.conf***

修改 `upstream`

```config
upstream uwsgi {
    # server api:8001; # use TCP
    # server unix:/docker_api/app.sock weight=4; # for a file socket
    # server unix:/docker_api2/app2.sock  weight=6; # for a file socket
     server demo_api_1:8001 weight=4; # use TCP
     server demo_api_2:8001 weight=6;  # use TCP
}
```

注意到了嘛！！ `demo_api_1` 前面的 `demo` 就是你的資料夾名稱，

避免資料夾名稱太長，所以我一開始命名為簡單的 `demo`。

使用 scale 方式沒辦法為 container 指定命名。

***docker-compose.yml***

先將 `api2` service 拿掉，因為我們要使用 Docker scale 的方式來完成。

```yml
version: '3'
services:

    db:
    #   container_name: postgres
      image: postgres
      environment:
        POSTGRES_PASSWORD: password123
      ports:
        - "5432:5432"
        # (HOST:CONTAINER)
      volumes:
        - pgdata:/var/lib/postgresql/data/

    nginx:
        # container_name: nginx-container
        build: ./nginx
        restart: always
        ports:
        - "8080:80"
        volumes:
        - api_data:/docker_api
        - ./log:/var/log/nginx
        depends_on:
        - api

    api:
        build: ./api
        restart: always
        command:  bash -c "python manage.py collectstatic --noinput &&
                                        uwsgi --ini uwsgi.ini"
        ports:
        - "8002"
        volumes:
        - api_data:/docker_api
        depends_on:
        - db

volumes:
    api_data:
    pgdata:
```

在 command 中，多加上 `collectstatic` 指令，

這樣就不用每次都要另外進入 container 內執行 :relaxed:

```cmd
python manage.py collectstatic --noinput
```

後面加上的 `--noinput` 也可以改成 `--no-input`，

主要目的是不會再跳出東西要你輸入。

## 執行步驟

主要是透過 docker 中的 scale 指令，

直接執行 `docker-compose up --scale api=2` 見證奇蹟

scale 詳細用法可參考 [https://docs.docker.com/compose/reference/scale/](https://docs.docker.com/compose/reference/scale/)

使用這個方法，一樣會啟動兩個 container，分別是 `api1` 以及 `api2`，

其餘的步驟和之前的都一樣，這邊就沿用 [master 分支](https://github.com/twtrubiks/docker-django-nginx-uwsgi-postgres-load-balance-tutorial) 的圖 :blush:

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

## 執行環境

* Mac
* Python 3.6.2
* windows 10

## Reference

* [https://docs.docker.com/](https://docs.docker.com/)
* [nginx-load_balancing](http://nginx.org/en/docs/http/load_balancing.html)

## License

MIT license
