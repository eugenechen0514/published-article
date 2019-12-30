![1_rat3S9caTDTLKnbAUhBuZg.png](resources/31D0195ACF5FC781F6628B807A425DC2.png =1500x1139)

# 前言
在[使用 Docker 構築不同 MongoDB 架構 (一)](https://medium.com/@yujiechen0514/%E4%BD%BF%E7%94%A8-docker-%E6%A7%8B%E7%AF%89%E4%B8%8D%E5%90%8C-mongodb-%E6%9E%B6%E6%A7%8B-%E4%B8%80-dde6ae2aa4fb) 提到 MongoDB 可能有以下架構：
1. Standalone
2. Replica Set
3. Sharded Cluster

因為 Standalone 是最簡單的 MongoDB 架構，所以我們直接用 Docker 架設 Standalone MongoDB，來介紹「當使用 Docker 架設 MongoDB 要注意的事項」。

開始之前先列舉幾個要點，請先放在心中：
1. Network: Port 設定、存取 Docker 容器內的 MongoDB
2. OS (Operating System): Memory, Open file limitation (系統開檔限制)
3. Data Persistence(資料持久化): volume mounting (卷掛載)

當看完本篇可以試著找出它們的出現在哪？

> 本系列文的專案放在：[https://github.com/eugenechen0514/demo_mongo_cluster](https://github.com/eugenechen0514/demo_mongo_cluster)

> 本系列文使用只 [Docker Compose File](https://docs.docker.com/compose/) 來設定 MongoDB 叢集。我們假設已安裝 Docker，還沒安裝請到 [Docker Desktop](https://www.docker.com/products/docker-desktop) 下載安裝。

> 為了專注建立 MongoDB 叢集，我們容器的 Network Mode 都是假設為 `host`，使用其它模式(見： [Day 30- 三周目 - Docker network 暨完賽回顧](https://ithelp.ithome.com.tw/articles/10206725)) 會需要有更多的設定，所以我們不使用。

# 用 Docker Compose 建立 Standalone MongoDB

Standalone 架構只有一個  MongoDB 的實體，存取資料的資料庫只有一個， Mongo Client 只要直接連到它就可以使用。

我們的資料夾目錄如下，並假設工作目錄在 *demo_mongo_cluster* (也就是 */Users/eugenechen/Documents/Projects/demo_mongo_cluster*)

![IMAGE](resources/B71EA51FB50820FF9831E0CEE40317DB.jpg =490x157)

我們可以在 *demo_mongo_cluster* 工作目錄下執行以下操作
1. 建立 MongoDB 容器並且在背景執行容器
    ``` shell
    docker-compose -f docker-compose-standalone.yml up -d
    ```
    就會在背景執行 (`-d`) 一個 Standalone MongoDB。
  
    第一次執行應該會像這樣  
    ![IMAGE](resources/23CBAF5CB8D016F55AB908F28A21FC55.jpg =575x362) 
    紅線上相當於執行指令 `docker pull mongo:4.2`，下載 `mongo:4.2` 的映像檔到本機；紅線下是建立並執行容器。

2. 停止執行容器
    ``` shell
    docker-compose -f docker-compose-standalone.yml stop
    ```
    請注意，此操作是停止容器運作導致 MongoDB process(行程) 停止，類似 kill process 非正常關閉 MongoDB，在有讀寫的情況下會有無法預期的情況發生。*安全的* 關閉 MongoDB 要連入資料庫
    ```
    use admin
    db.shutdownServer()
    ```
    先關 MongoDB ，再停止容器會比較安全。
3. 重新執行容器(在背景運作)
    ``` shell
    docker-compose -f docker-compose-standalone.yml start
    ```
4. 停止並刪除容器
    ``` shell
    docker-compose -f docker-compose-standalone.yml down
    ```

# 在 Docker 中使用 Mongo Shell

因為我們使用 MongoDB 4.2 (`mongo:4.2`) ，所以 Mongo Shell 建議也要使用同樣的版本號，否則可能會有不可預期的事。

官方的 `mongo:4.2` 映像檔本身就放了所有的工具，像是 `mongo`, `mongoimport`, ... 等 (見：[MongoDB Package Components](https://docs.mongodb.com/manual/reference/program/)。因此我們只需要直接使用就可以了。在終端機輸入：
``` shell
docker run --rm -it --network host mongo:4.2 bash
```
就會建立一個 *暫時的* 容器且進入(或稱執行)一個具有 MongoDB shell 終端機，接下來就有 `mongo` 可以用了。
![IMAGE](resources/8A9A7AE0984580CF6C8AD767D7F68470.jpg =769x56)


## Docker 指令說明

不要被長長的指令嚇到了，分開看就一目瞭然

![Snip20191013_2.png](resources/897EE1A15EDC328B56D0277CF1F778D0.png =706x108)

A (`docker run`) ：讀入某個映像檔，建立容器並執行。
B (`--rm`) ：容器停止後自動刪除，也就是不能再用 `docker start` 重新執行容器。
C (`-it`)：這是 `-i/--interactive`, `-t/--tty` 的合併，簡單來說就是開啟*可互動式終端機*。
D (`--network host`)：容器的網路模式用 `host`，可以想像容器「寄生」在本機上，所以 `127.0.0.1` 就是指*本機*。此模式下，若容器內部開啟任何 *PORT* 就如同在本機開啟。
E (`mongo:4.2`)：映像檔名是 `mongo`，tag(標籤)是 `4.2`，就是用 MongoDB 4.2 版本。
F (`bash`)： 當 `docker run` 執行容器時要執行的 *command* (指令)。 `bash` 就是指執行容器內 `/bin/bash` 的程式。

全部合起來看就像是：在 *本機端* 開啟一個 *具有 MongoDB tools* 且 *暫時的* *終端機*。


接下來，我們詳細說明 Standalone 組態。有二個地方需要關注：
1. Docker Compose File / Data Folder
2. MongoDB Config File


# 詳解 Standalone 組態 - Docker Compose File / Data Folder

``` yml
version: '3'
services:
  database:
    image: mongo:4.2
    container_name: mongo4
    network_mode: host
    command: mongod --dbpath /data/db --port 27041 --config /resource/mongod.yml
    volumes:
      - ./standalone/config/mongod.yml:/resource/mongod.yml
      - /Users/eugenechen/Documents/Projects/demo_mongo_cluster/standalone/data:/data/db
```

## `network_mode: host`

`database` 的網路模式是 `host`，當容器內部開啟任何 *PORT* 就如同在本機開啟。

## `command: mongod --dbpath /data/db --port 27041 --config /resource/mongod.yml`

`command` 是容器執行時要執行的指令。 

它後面接的 ` mongod` 有三個常用的參數：
1. `--dbpath` 是指定 MongoDB 存放資料的資料夾。
2. `--port` 是 MongoDB 要開啟(監聽)的 PORT。 
3. `--config` 指定 MongoDB Config file。

我們注意到：路徑應該要是在容器內的路徑，不是指本機端的路徑。

## `volumes:` 
這是把本機端的檔案/資料夾掛載到容器內的 File System。這方法在 docker 中叫 [bind mounts](https://docs.docker.com/v17.09/engine/admin/volumes/bind-mounts/)，可以讓本機端和容器共享資料，我們透過此方法就可以把 MongoDB 的資料留在本機端，不會因為刪除容器而資料同時刪除。

`- /Users/eugenechen/Documents/Projects/demo_mongo_cluster/standalone/data:/data/db` 的格式是 `A:B`， *A* 是本機端的路徑，*B*  是容器內的路徑。當容器建立完成掛載，這兩個路徑就可以共享資料。如果在容器中寫資料到 `/data/db` 中，從本機端的 `/Users/eugenechen/Documents/Projects/demo_mongo_cluster/standalone/data` 也會看的到。

MongoDB 雖然執行在容器中，但我們有指定資料庫位置到 `/data/db` 中，且它被 bind (綁定) 在  `/Users/eugenechen/Documents/Projects/demo_mongo_cluster/standalone/data` 中，所以當容器刪除 (`docker-compose down`) 資料依然會留在本機端。下次重新建立 MongoDB 容器時 (`docker-compose up`)，可以指向同樣的資料庫位置，就可以繼續使用以前的資料。

`bind mounts` 也可以用在檔案上，我們指定本機端的 `./standalone/config/mongod.yml` (相對目錄) 掛載到容器中的 `/resource/mongod.yml`， `mongod` 就可以用 `--config /resource/mongod.yml` 讀入 MongoDB 組態檔。

# 詳解 Standalone 組態 - MongoDB Config File
``` yml
net:
  bindIpAll: true
storage:
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.2

```
這裡完整的 MongoDB 組態見 [Configuration File Options](https://docs.mongodb.com/manual/reference/configuration-options/)，我們只用幾個重要的設定

## net Options 的 `bindIp` / `bindIpAll`
此設定哪些 Client IP 可以連入 MongoDB 中，預設是 `localhost` 也就是本機而已。我們假設 Client IP 可能來自不同的主機，所以用 `net.bindIpAll: true` 指定任意 Client IP 都可以連入。

## storage Options 的 `engine`
`storage.engine` 有 `wiredTiger` / `inMemory` 可以選擇，不論選哪個都要小心設定 「*可以使用的 `cacheSizeGB` / `inMemorySizeGB`* 大小」。因為 Mongo 沒有假設它會執行在容器中，它會依照硬體配置[計算預設值](https://docs.mongodb.com/manual/reference/configuration-options/#storage.wiredTiger.engineConfig.cacheSizeGB)，所以當一台機器用 Docker 架兩台以上 Mongo 就要小心過度配置記憶體大小。因此，當使用 Docker 時，我們就 *一定* 要設定大小。

> 若使用 [Vagrant](https://www.vagrantup.com/) 這類的虛擬機器，也最好設定記憶體大小。

> 筆者有切身經驗，因為沒有設定大小導致狂斷線或有時會連不進去。

# 詳解 Standalone 組態 - Open file limitation

一般 OS 建立連線(socket) 時會開啟檔案，若是 OS 所設定的開檔數不夠大，就會出現
```
Too many open files
```
之類的錯誤訊息，這設定會影響到 MongoDB 可以開啟的連線數。

> 順帶一提，除了 OS 層級的連線限制，MongoDB 也提供 [net.maxIncomingConnections](https://docs.mongodb.com/manual/reference/configuration-options/#net.maxIncomingConnections) 限制最大連線數。

根據 Docker 文件 [Set ulimits in container](https://docs.docker.com/engine/reference/commandline/run/#set-ulimits-in-container---ulimit)，它說預設的值是由 Docker Daemon 決定。我們可以用
``` shell
docker run --rm mongo:4.2 sh -c "ulimit -n"
# 1048576
```
查到 Docker 建立容器時預設的開檔限制。

## 修改開檔上限
我們可以在建立容器時修改開檔上限，使用 `docker run --ulimit` 
``` shell
docker run --ulimit nofile=1024:1024 --rm mongo:4.2 sh -c "ulimit -n"
# 1024
```

若是使用 Docker Compose File，加入 [`ulimits`](https://docs.docker.com/compose/compose-file/#ulimits)
``` yml
version: '3'
services:
  database:
    image: mongo:4.2
    container_name: mongo4
    ...略
    ulimits:
      nofile:
        soft: 20000
        hard: 40000
```

檢驗是否有修改成功

``` shell
docker exec  mongo4  sh -c "ulimit -n"
# 20000
```

> 筆者發現 Docker Daemon 預設的開檔數本來就很大，目前還沒遇到需要修改的必要，但請讀者留心有這限制。

# 總結
我們透過建立 Standalone MongoDB 來引入 Docker 的組態應該如何設定和使用。

讀者可以回想以下要點，是否找到答案呢？
1. Network: Port 設定、存取 Docker 容器內的 MongoDB
2. OS: Memory, Open file limitation (系統開檔限制)
3. Data Persistence(資料持久化): volume mounting (卷掛載)

# 參考連結
* [Compose file version 3 reference](https://docs.docker.com/compose/compose-file/)
* [MongoDB Configuration File Options](https://docs.mongodb.com/manual/reference/configuration-options/#configuration-file-options)
* [深入理解Docker ulimit](http://dockone.io/article/522)