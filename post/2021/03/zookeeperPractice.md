# zookeeper 를 이용한 분산 클러스터 구조 개발
JAVA 환경에서 여러 서버의 상태를 관리해야 하는 일이 생겼다.

 현재 스펙
- 각 서버는 각자의 처리량을 가지고 있다.
- 기존 방식은 처리량을 redis에 저장하고 각자의 서버들이 redis의 특정 영역에 자신의 처리량을 업데이트 하고, 다른 서버의 처리량을 읽어서 자신이 현재 처리할 max 처리량을 계산하는 방식으로 구현되어 있다.

문제점 도출
- polling 방식으로 구현되어 있어서 주기적으로 redis에 읽고 / 쓰기가 발생하며 이는 서버 증설시마다 계속 증가하며, 불필요한 읽기 쓰기가 발생한다.
- 모든 서버가 처리량에 대한 계산을 하고 있다.

개선을 위한 요구사항 도출
- 처리량 계산을 하는 리더 서버를 만들어서 계산 로직을 한 서버에 전담 시키자.
- 리더 서버가 각 서버의 상태(처리량 값)을 변경할수 있어야 한다.

요구사항 도출까지 끝냈을 시점에 가장 먼전 든 생각은 zookeeper 였다.
정확히 어떠한 기능을 가지고 서비스인지는 몰랐지만 kafka의 topic, replica, leader 관련 기능에 사용된다는 것을 알고 있는 상태였다.

그래서 일단 몇몇 문서를 읽고 공부겸 바로 구현을 해 봤다.
- https://zookeeper.apache.org/
- https://brunch.co.kr/@timevoyage/77

결론부터 말하자면 생각했던 요구사항을 생각보다 쉽고 빠르게 구현 할수 있어서 놀랐다. (밑 바닥부터 한 6시간 정도 투자하였다)


## 구현

### 요구사항
1. 여러 클라이언트는 각자의 처리량을 가지고 있다.
2. 리더 클라이언트는 다른 클라이언트의 처리량을 관리한다.
3. 리더는 전체 처리량이 변경될시 전체 클라이언트의 처리량에 반영한다.
4. 리더 클라이언트가 죽을시 다른 클라이언트 중 하나가 리더가 된다.

### znode 설정과 연결 로직
/global-config/max-throughput (persistent) - 모든 클러스터가 처리해야할 총 처리량
/leader (ephemeral) - leader client의 client ID를 가지고 있는 노드
/client (persistent) - 서비스를 할수 있는 클라이언트들의 정보를 가지고 있는 노드
/client/client-{sequential} (ephemeral sequential) - 서비스 가능한 client ID 노드

#### client 등록
1. client 는 서비스 가능 상태일시 zookeeper의 /client 하위에 zooKeeper의 CreateMode.EPHEMERAL_SEQUENTIAL 기능을 이용해서 자신을 등록한다. 
2. 등록이 성공되면 'client-{sequential}' 로 ID가 생성되며 이를 client ID로 사용한다.
3. /leader 노드가 존재하는지 확인하고 없다면 자기 자신을 leader로 등록한다.

#### leader 의 선출과 변경
1. client 등록시 /leader 노드의 존재 여부를 통해서 최초 client가 leader가 된다.
2. leader client가 종료 되면 zookeeper 의 기능(ephemeral)을 통해 /leader 노드가 제거 된다.
3. leader 노드의 삭제는 zookeeper의 watch 기능을 통해서 각 client 들에게 알려진다.
4. 신규 leader 선출 로직이 동작한다.

#### 처리량 변경시 동작
1. leader client 는 /global-config/max-throughput 노드의 변경을 watch하고 있는다.
2. 노드가 값이 변경되면 leader 는 전체 client의 처리량을 조정한다.

#### client 변경시 동작
1. leader client 는 /leader 노드의 변경을 watch하고 있는다.
2. /leader 하위에 신규 client가 생성되거나, 기존 client가 제거되면 leader는 전체 client의 처리량을 조정한다.

### 요구사항1 - 여러 클라이언트는 각자의 처리량을 가지고 있다.
```
// client로 등록한다, 정보에 status를 json으로 넣어 놓는다.
this.clientPath = zk.create(ZookeeperConfig.clientRootNodePath + ZookeeperConfig.clientNodePrefixPath,
        objectMapper.writeValueAsBytes(status), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
// 외부에서 status정보가 변경되면 자신의 status 값을 수정하기 위해 watcher를 등록한다
zk.addWatch(this.clientPath, clientStatusWatcher, AddWatchMode.PERSISTENT);

...

private Watcher clientStatusWatcher = event -> {
    log.info("Status Change Watcher : {}", event);

    // status 값 변경이 되었으면 로컬 데이터를 업데이트 한다.
    if (Watcher.Event.EventType.NodeDataChanged == event.getType()) {
        try {
            log.info("Prev status : {}", status);
            byte[] bStatus = zk.getData(clientPath, false, null);
            status = objectMapper.readValue(bStatus, Status.class);
            log.info("Update status : {}", status);
        } catch (Exception e) {
            log.error("Status Change Watcher Exception : {}", e.getMessage(), e);
        }
    }
};
```

### 요구사항2 - 리더 클라이언트는 다른 클라이언트의 처리량을 관리한다.
```
private Watcher clientWatchWatcher = new Watcher() {
    @Override
    public void process(WatchedEvent event) {
        log.info("Client State Event : {}", event);
        Event.EventType eventType = event.getType();

        switch (eventType) {
            case NodeCreated:
                log.info("Join Client");
                rebalanceClientThroughput();
                break;
            case NodeDeleted:
                log.info("Detach Client");
                rebalanceClientThroughput();
                break;
            default:
                break;
        }
    }
};

if (KeeperException.Code.OK == returnCode) {
    try {
        // 전체 throughput 데이터를 가져옴
        this.clusterMaxThroughput = getClusterMaxThroughput();

        // 자식 프로세스 상태 확인
        // 자식들 throughput 처리
        zk.addWatch(ZookeeperConfig.clientRootNodePath, clientWatchWatcher, AddWatchMode.PERSISTENT_RECURSIVE);

        // 다른 client 들의 throughput 제
        rebalanceClientThroughput();
    } catch (Exception e) {
        log.error("add watch exception {}", e.getMessage(), e);
    }
}

...

private void rebalanceClientThroughput() {
    try {
        List<String> clientList = zk.getChildren(ZookeeperConfig.clientRootNodePath, false);

        // 일단 간단한 로직으로 ...
        long eachThroughput = (long) Math.floor(clusterMaxThroughput / clientList.size());

        for (String client : clientList) {
            String clientPath = ZookeeperConfig.clientRootNodePath + "/" + client;
            byte[] bStatus = zk.getData(clientPath, false, null);
            log.info("{} status is {}", client, new String(bStatus));
            Status status = objectMapper.readValue(bStatus, Status.class);
            status.setThroughput(eachThroughput);
            zk.setData(clientPath, objectMapper.writeValueAsBytes(status), -1);
        }
    } catch (Exception e) {
        log.error("rebalance Exception : {}", e.getMessage(), e);
    }
}
```

### 요구사항3 - 리더는 전체 처리량이 변경될시 전체 클라이언트의 처리량에 반영한다.
```
// cluster의 max throughput 변경 감지
zk.addWatch(ZookeeperConfig.clusterMaxThroughputPath, event -> {
    if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
        clusterMaxThroughput = getClusterMaxThroughput();
        rebalanceClientThroughput();
    }
}, AddWatchMode.PERSISTENT);
```

### 요구사항4 - 리더 클라이언트가 죽을시 다른 클라이언트 중 하나가 리더가 된다.
```
private Watcher leaderStateWatcher = event -> {
    log.info("Leader Client State Change : {}", event);

    try {
        switch (event.getType()) {
            case NodeCreated:
                log.info("Prev leader is {}", leaderClientId);
                leaderClientId = new String(zk.getData(ZookeeperConfig.leaderNodePath, false, null));
                log.info("New leader is {}", leaderClientId);
                break;
            case NodeDeleted:
                log.info("{} is dead", leaderClientId);
                electNewLeader();
                break;
            default:
                break;
        }
    } catch (Exception e) {
        log.error("Leader Event Error : {}", e.getMessage(), e);
    }

};
```


## 구현 결과
Client1 로그 (샘플)
- 첫번쨰 Client는 leader 로 선출 됨
- 신규 Client 를 감지하고 throughput 조정 작업 진행
```
[main] INFO jiho.zookeeper.Client - client-0000000000 Client is Ready, Leader Client is client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000000 status is {"throughput":10}
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=10)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=1000)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeCreated path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Join Client
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":10}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000000 status is {"throughput":1000}
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=1000)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=500)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeCreated path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - Join Client
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":500}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000000 status is {"throughput":500}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":10}
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=500)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeCreated path:/client/client-0000000003
[main-EventThread] INFO jiho.zookeeper.Client - Join Client
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":333}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000000 status is {"throughput":333}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000003 status is {"throughput":10}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":333}
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=250)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000003
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
```

Client 2 로그(샘플)
```
[main] INFO jiho.zookeeper.Client - client-0000000001 Client is Ready, Leader Client is client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=10)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=250)
[main-EventThread] INFO jiho.zookeeper.Client - Leader Client State Change : WatchedEvent state:SyncConnected type:NodeDeleted path:/leader
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000000 is dead
[main-EventThread] INFO jiho.zookeeper.Client - Elect a new leader
[main-EventThread] INFO jiho.zookeeper.Client - Candidate list : [client-0000000001, client-0000000003, client-0000000002]
[main-EventThread] INFO jiho.zookeeper.Client - Leader Client State Change : WatchedEvent state:SyncConnected type:NodeCreated path:/leader
[main-EventThread] INFO jiho.zookeeper.Client - Prev leader is client-0000000000
[main-EventThread] INFO jiho.zookeeper.Client - New leader is client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":250}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000003 status is {"throughput":250}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":250}
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=250)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000003
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDeleted path:/client/client-0000000003
[main-EventThread] INFO jiho.zookeeper.Client - Detach Client
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":333}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":333}
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=333)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=500)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":500}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":500}
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=500)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=250)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeCreated path:/client/client-0000000004
[main-EventThread] INFO jiho.zookeeper.Client - Join Client
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000001 status is {"throughput":250}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000002 status is {"throughput":250}
[main-EventThread] INFO jiho.zookeeper.Client - client-0000000004 status is {"throughput":10}
[main-EventThread] INFO jiho.zookeeper.Client - Status Change Watcher : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Prev status : Status(throughput=250)
[main-EventThread] INFO jiho.zookeeper.Client - Update status : Status(throughput=166)
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000001
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000002
[main-EventThread] INFO jiho.zookeeper.Client - Client State Event : WatchedEvent state:SyncConnected type:NodeDataChanged path:/client/client-0000000004
```

### 구현 결과 (표)

|                                       | client1              | client2             | client3     | client4     | client5     |
|:--------------------------------------|:---------------------|:--------------------|:------------|:------------|:------------|
| client1 up                            | leader, client, 1000 |                     |             |             |             |
| client2 up                            | leader, client, 500  | client, 500         |             |             |             |
| client3 up                            | leader, client, 333  | client, 333         | client, 333 |             |             |
| client4 up                            | leader, client, 250  | client, 250         | client, 250 | client, 250 |             |
| client1 down                          | -                    | leader, client, 333 | client, 333 | client, 333 |             |
| client4 down                          | -                    | leader, client, 500 | client, 500 | -           |             |
| zookeeper max-throughput  1000 -> 500 | -                    | leader, client, 250 | client, 250 | -           |             |
| client5 up                            | -                    | leader, client, 166 | client, 166 |             | client, 166 |

## 총평
에러 핸들링이나, zookeeper 서버가 구성등에 대해서는 크게 고민하지 않고 기능과 구현에만 집중하였다.
zookeeper는 생각보다 더 쉽고, 기능적인 면에서 정말 유용하다.