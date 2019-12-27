![1_rat3S9caTDTLKnbAUhBuZg.png](resources/31D0195ACF5FC781F6628B807A425DC2.png =1500x1139)

# 目標

你可以學習到：
* 使用 docker 建立常見的 MongoDB 架構
* 如何存取 MongoDB

# 為什麼我們要用 Docker？

MongoDB 可以安裝在  上[Linux/macOS/Windows](https://docs.mongodb.com/manual/installation/)，它提供水平擴展 ([Sharding](https://docs.mongodb.com/manual/sharding/))，垂直擴展 ([Replication](https://docs.mongodb.com/manual/replication/)) 的強大機制。 然而，要建置 MongoDB 叢集會有很多工作要做，像是啟動很多台 MongoDB 的服務、組態設定，工作十份繁複。

再者，你也可能遭遇實體主幾不夠而不能啟動太多 MongoDB(若一台只啟動一個 MongoDB)，或者一台機器太強大，有高核心數高記憶本，想要榨乾它的效能。

基於以上理由， Docker 容器化技術為構築 MongoDB 叢集提供很好的解決方案。 Docker Compose File 提供述描執行服務的組態設定，以達到 *code as infrastructure* 的目地，你可以容易在任何 docker 環境下重現你的 MongoDB 叢集。

> 若你想要快速的體驗使用 docker 建立 MongoDB 可以見我在 iT 邦的文章 [用js成為老闆心中的全端工程師：Day 15 - 二周目 - 用 Docker 玩轉 MongoDB](https://ithelp.ithome.com.tw/articles/10201657)

> 因為本篇的目地是「用 Docker 建立 MongoDB 叢集」，所以不會涉汲太多 Docker 說明。若需要幫助，可以見以下的 *用js成為老闆心中的全端工程師* 文章：
[Day 26 - 三周目 - Docker 基本使用：看完就會架 docker 化的服務](https://ithelp.ithome.com.tw/articles/10205481)
[Day 29- 三周目 - Docker Compose：一次管理多個容器](https://ithelp.ithome.com.tw/articles/10206437)
[Day 30- 三周目 - Docker network 暨完賽回顧](https://ithelp.ithome.com.tw/articles/10206725)

# MongoDB 架構(Architecture)

MongoDB 可能有以下架構：
1. Standalone：
    * 建立難易度：低
    * 特色：單一台 MongoDB 服務，只有一個存取資料庫
    * 用途：獨立使用或用於維護 Replica Set/Sharded Cluster 的單一 node
2. [Replica Set](https://docs.mongodb.com/manual/replication/)
    * 建立難易度：中
    * 特色：資料在 Replica Set 中會有副本資料
    * 用途：
        * 需要 High availability (HA) 的環境
        * 依硬體效能調整副本策略，如：你有較快/較大的硬碟
        * 不影響 Primary，使用 [Read Preference](https://docs.mongodb.com/manual/core/read-preference/) 讀取較近或快的副本
![IMAGE](resources/8177C8A69E31D60050C8E0CBC13FE119.jpg =457x437)

3. [Sharded Cluster](https://docs.mongodb.com/manual/sharding/)
    * 建立難易度：高
    * 特色：資料遍佈在不同的 shard，每個 shard/config servers 都是 `Replica Set`，所以同時有 HA 的能力
    * 用途：
        * 依照 [Shard Keys](https://docs.mongodb.com/manual/core/sharding-shard-key/) 分散式地存取資料
        * 依硬體效能調整分散策略，如：你有較強/多核的CPU，常用資料可以放在讀寫快的主機
![IMAGE](resources/B88E712DED055E3627E1D1B56BC1C39C.jpg =645x519)

當 MongoDB 變成叢集就會衍生出以下問題：
1. 資料怎麼分散，分散的略策如何影響存取效能？ -> [Shard Keys](https://docs.mongodb.com/manual/core/sharding-shard-key/) 資料如何畫分 [chunk](https://docs.mongodb.com/manual/core/sharding-data-partitioning/index.html)、[Zones](https://docs.mongodb.com/manual/core/zone-sharding/) 指定資料存放位置、[Balancer](https://docs.mongodb.com/manual/core/sharding-balancer-administration/) 平衡 Shard 中的 chunk 數量
2. 邏輯上怎樣才算寫入資料庫? -> [Write Concern](https://docs.mongodb.com/manual/reference/write-concern/index.html) 資料寫入的策略
3. 邏輯上怎樣才算讀出真正的資料? -> [Read Concern](https://docs.mongodb.com/manual/reference/read-concern/index.html) 資料讀取的策略
4. 怎麼決定誰是 Primary ?  -> [Replica Set Elections](https://docs.mongodb.com/manual/core/replica-set-elections/#replica-set-elections) 推選投票誰當 Primary
5. 叢集如何管理？  -> [Replica Set Maintenance](https://docs.mongodb.com/manual/administration/replica-set-maintenance/)、[Sharded Cluster Administration](https://docs.mongodb.com/manual/administration/sharded-cluster-administration/)
    ...等。 

因為本系列只專注建立叢集，不會涉及其它議題，有興趣的人可以自行學習。

# 試用 MongoDB Atlas

若你不想自己架設 MongoDB，可以考慮試用看看 MongoDB Atlas。它提供 DBaaS，幫你架設在 AWS, Azure 或 GCP。目前有 512 MB 免費空間且現送 $200 優惠券，代碼為 *MAXIME200*。見：[Quick Start: Getting Your Free MongoDB Atlas Cluster](https://www.mongodb.com/blog/post/quick-start-getting-your-free-mongodb-atlas-cluster?fbclid=IwAR0IGM60a1mlkZ89MekcJsjcqCskl8PYjfnKC8Nt8fVGnNzf59NkhemdUtw)

你可以在以下地方輸入優惠券代碼
![mongoalt1.jpg](resources/A52EE48F2277B236028D1FFE0110859F.jpg =2234x836)
![mongoalt2.jpg](resources/3CE447D6A2A5D00130949E8F914EEE09.jpg =2048x1417)