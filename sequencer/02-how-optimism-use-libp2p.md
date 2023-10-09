# Application of libp2p in Optimism

In this section, we will mainly discuss how Optimism utilizes libp2p to establish the p2p network in op-node. The p2p network primarily serves the purpose of passing information between different nodes. For instance, after a sequencer completes the construction of an unsafe block, it is disseminated through the p2p's gossiphub via pub/sub. libp2p also addresses other basics in the p2p network, such as networking, addressing, etc.

## Understanding libp2p

libp2p (shortened from "library peer-to-peer") is a framework geared towards peer-to-peer (P2P) networks that aids in the development of P2P applications. It encompasses a set of protocols, standards, and libraries, making P2P communication between network participants (also known as "peers") more straightforward ([source](https://docs.libp2p.io/introduction/what-is-libp2p)). Initially, libp2p was introduced as a part of the IPFS (InterPlanetary File System) project. However, over time, it evolved into an independent project, becoming a modular network stack for distributed systems ([source](https://proto.school/introduction-to-libp2p)).

libp2p is an open-source initiative by the IPFS community, inviting extensive community contributions, including assisting in drafting specifications, coding implementations, and curating examples and tutorials ([source](https://libp2p.io/)). Comprised of various building blocks, each module of libp2p boasts clear, well-documented, and tested interfaces. This design allows them to be combinable, interchangeable, and hence upgradeable ([source](https://medium.com/@mycoralhealth/understanding-ipfs-in-depth-5-6-what-is-libp2p-fd42ed27e656)). The modularity of libp2p enables developers to pick and utilize only those components that are essential for their application, promoting flexibility and efficiency during the construction of P2P network applications.

### Relevant Resources
- [Official documentation of libp2p](https://docs.libp2p.io/)
- [GitHub repository of libp2p](https://github.com/libp2p)
- [Introduction to libp2p on ProtoSchool](https://proto.school/introduction-to-libp2p)

The modular architecture and the open-source nature of libp2p offer a conducive environment for developing robust, scalable, and versatile P2P applications, marking its significance in the realm of distributed network and web application development.

### Implementation of libp2p

When deploying libp2p, you would need to implement and configure certain core components to establish your P2P network. Here are some primary aspects of libp2p application:

#### 1. **Creation and Configuration of Nodes**:
   - The foundational step is the creation and configuration of a libp2p node. This encompasses setting up the node's network address, identity, and other basic parameters.
   Essential usage code:
```go
   libp2p.New()
```

#### 2. **Transport Protocols**:
   - Choose and configure your transport protocols (e.g., TCP, WebSockets, etc.) to ensure communication between nodes.
   Key usage code:
```go
   tcpTransport := tcp.NewTCPTransport()
```

#### 3. **Multiplexing and Flow Control**:
   - Implement multiplexing to allow handling multiple concurrent data streams on a single connection.
   - Implement flow control to manage the transmission and processing rate of data.
   Key usage code:
```go
   yamuxTransport := yamux.New()
```

#### 4. **Security and Encryption**:
   - Configure a secure transport layer to ensure the security and privacy of communications.
   - Implement encryption and authentication mechanisms to protect data and verify communicators.
   Key usage code:
```go
   tlsTransport := tls.New()
```

#### 5. **Protocols and Message Handling**:
   - Define and implement custom protocols to handle specific network operations and message exchanges.
   - Handle received messages and send responses as needed.
   Key usage code:
```go
   host.SetStreamHandler("/my-protocol/1.0.0", myProtocolHandler)
```

#### 6. **Discovery and Routing**:
   - Implement node discovery mechanisms to find other nodes in the network.
   - Implement routing logic to determine how to route messages to the correct node in the network.
   Key usage code:
```go
   dht := kaddht.NewDHT(ctx, host, datastore.NewMapDatastore())
```

#### 7. **Network Behavior and Policies**:
   - Define and implement behaviors and policies for the network, such as connection management, error handling, and retry logic.
   Key usage code:
```go
   connManager := connmgr.NewConnManager(lowWater, highWater, gracePeriod)
```

#### 8. **State Management and Storage**:
   - Manage the state of the node and the network, including connection states, node lists, and data storage.
   Key usage code:
```go
   peerstore := pstoremem.NewPeerstore()
```

#### 9. **Testing and Debugging**:
   - Write tests for your libp2p application to ensure its correctness and reliability.
   - Use debugging tools and logs to diagnose and solve network issues.
   Key usage code:
```go
   logging.SetLogLevel("libp2p", "DEBUG")
```

#### 10. **Documentation and Community Support**:
   - Consult the documentation of libp2p to understand its various components and APIs.
   - Engage with the libp2p community for support and problem resolution.

The above are some primary aspects to consider and implement when using libp2p. The specific implementation might vary for each project, but these foundational elements are essential for building and running libp2p applications. When implementing these features, you can refer to the [official documentation of libp2p](https://docs.libp2p.io/) and example codes and tutorials in the [GitHub repository](https://github.com/libp2p/go-libp2p/tree/master/examples).


## Use of libp2p in OP-node

To understand the relationship between op-node and libp2p, we need to clarify a few questions:
    - Why choose libp2p? Why not devp2p (as used by geth)?
    - What data or processes in OP-node are closely related to the p2p network?
    - How are these functionalities implemented at the code level?

### Reasons op-node needs the libp2p network

**First, we need to understand why optimism requires a p2p network.**
libp2p is a modular network protocol, allowing developers to build decentralized peer-to-peer applications suitable for various use cases ([source](https://ethereum.stackexchange.com/questions/12290/what-is-the-distinction-between-libp2p-devp2p-and-rlpx))([source](https://www.geeksforgeeks.org/what-is-the-difference-between-libp2p-devp2p-and-rlpx/)). On the other hand, devp2p is mainly for the Ethereum ecosystem, tailored specifically for Ethereum applications ([source](https://docs.libp2p.io/concepts/similar-projects/devp2p/)). The flexibility and broad applicability of libp2p might make it a preferred choice for developers.

### Main functionalities of op-node using libp2p

    - To transmit unsafe blocks produced by the sequencer to other non-sequencer nodes.
    - For fast synchronization (reverse chain synchronization) in other nodes under non-sequencer mode when gaps appear.
    - To employ an integral reputation system to maintain a conducive environment among all nodes.

### Code Implementation

#### Custom Initialization of the host

The host can be understood as the p2p node. When starting this node, some specific initialization configurations are required according to the project.

Now let's take a look at the `Host` method in the `op-node/p2p/host.go` file.

This function is primarily used to set up the libp2p host and various configurations. Below are the key parts of the function along with a brief description in English:

1. **Check if P2P is Disabled**  
   If P2P is disabled, the function will return immediately.

2. **Obtain Peer ID from the Public Key**  
   Generate the Peer ID using the public key from the configuration.

3. **Initialize the Basic Peerstore**  
   Create a basic Peerstore storage.

4. **Initialize the Extended Peerstore**  
   Build an extended Peerstore on top of the basic one.

5. **Add Private and Public Keys to Peerstore**  
   Store the Peer's private and public keys in the Peerstore.

6. **Initialize the Connection Gater**  
   For controlling network connections.

7. **Initialize the Connection Manager**  
   For managing network connections.

8. **Set up Transport and Listening Addresses**  
   Set up the network transport protocols and the listening addresses for the host.

9. **Create the libp2p Host**  
   Use all the preceding settings to create a new libp2p host.

10. **Initialize Static Peers**  
    Initialize them if there are configured static peers.

11. **Return the Host**  
    Finally, the function returns the configured libp2p host.

These key parts are responsible for the initialization and setup of the libp2p host, with each part catering to a specific aspect of the host's configuration.



```go
    func (conf *Config) Host(log log.Logger, reporter metrics.Reporter, metrics HostMetrics) (host.Host, error) {
        if conf.DisableP2P {
            return nil, nil
        }
        pub := conf.Priv.GetPublic()
        pid, err := peer.IDFromPublicKey(pub)
        if err != nil {
            return nil, fmt.Errorf("failed to derive pubkey from network priv key: %w", err)
        }

        basePs, err := pstoreds.NewPeerstore(context.Background(), conf.Store, pstoreds.DefaultOpts())
        if err != nil {
            return nil, fmt.Errorf("failed to open peerstore: %w", err)
        }

        peerScoreParams := conf.PeerScoringParams()
        var scoreRetention time.Duration
        if peerScoreParams != nil {
            // Use the same retention period as gossip will if available
            scoreRetention = peerScoreParams.PeerScoring.RetainScore
        } else {
            // Disable score GC if peer scoring is disabled
            scoreRetention = 0
        }
        ps, err := store.NewExtendedPeerstore(context.Background(), log, clock.SystemClock, basePs, conf.Store, scoreRetention)
        if err != nil {
            return nil, fmt.Errorf("failed to open extended peerstore: %w", err)
        }

        if err := ps.AddPrivKey(pid, conf.Priv); err != nil {
            return nil, fmt.Errorf("failed to set up peerstore with priv key: %w", err)
        }
        if err := ps.AddPubKey(pid, pub); err != nil {
            return nil, fmt.Errorf("failed to set up peerstore with pub key: %w", err)
        }

        var connGtr gating.BlockingConnectionGater
        connGtr, err = gating.NewBlockingConnectionGater(conf.Store)
        if err != nil {
            return nil, fmt.Errorf("failed to open connection gater: %w", err)
        }
        connGtr = gating.AddBanExpiry(connGtr, ps, log, clock.SystemClock, metrics)
        connGtr = gating.AddMetering(connGtr, metrics)

        connMngr, err := DefaultConnManager(conf)
        if err != nil {
            return nil, fmt.Errorf("failed to open connection manager: %w", err)
        }

        listenAddr, err := addrFromIPAndPort(conf.ListenIP, conf.ListenTCPPort)
        if err != nil {
            return nil, fmt.Errorf("failed to make listen addr: %w", err)
        }
        tcpTransport := libp2p.Transport(
            tcp.NewTCPTransport,
            tcp.WithConnectionTimeout(time.Minute*60)) // break unused connections
        // TODO: technically we can also run the node on websocket and QUIC transports. Maybe in the future?

        var nat lconf.NATManagerC // disabled if nil
        if conf.NAT {
            nat = basichost.NewNATManager
        }

        opts := []libp2p.Option{
            libp2p.Identity(conf.Priv),
            // Explicitly set the user-agent, so we can differentiate from other Go libp2p users.
            libp2p.UserAgent(conf.UserAgent),
            tcpTransport,
            libp2p.WithDialTimeout(conf.TimeoutDial),
            // No relay services, direct connections between peers only.
            libp2p.DisableRelay(),
            // host will start and listen to network directly after construction from config.
            libp2p.ListenAddrs(listenAddr),
            libp2p.ConnectionGater(connGtr),
            libp2p.ConnectionManager(connMngr),
            //libp2p.ResourceManager(nil), // TODO use resource manager interface to manage resources per peer better.
            libp2p.NATManager(nat),
            libp2p.Peerstore(ps),
            libp2p.BandwidthReporter(reporter), // may be nil if disabled
            libp2p.MultiaddrResolver(madns.DefaultResolver),
            // Ping is a small built-in libp2p protocol that helps us check/debug latency between peers.
            libp2p.Ping(true),
            // Help peers with their NAT reachability status, but throttle to avoid too much work.
            libp2p.EnableNATService(),
            libp2p.AutoNATServiceRateLimit(10, 5, time.Second*60),
        }
        opts = append(opts, conf.HostMux...)
        if conf.NoTransportSecurity {
            opts = append(opts, libp2p.Security(insecure.ID, insecure.NewWithIdentity))
        } else {
            opts = append(opts, conf.HostSecurity...)
        }
        h, err := libp2p.New(opts...)
        if err != nil {
            return nil, err
        }

        staticPeers := make([]*peer.AddrInfo, len(conf.StaticPeers))
        for i, peerAddr := range conf.StaticPeers {
            addr, err := peer.AddrInfoFromP2pAddr(peerAddr)
            if err != nil {
                return nil, fmt.Errorf("bad peer address: %w", err)
            }
            staticPeers[i] = addr
        }

        out := &extraHost{
            Host:        h,
            connMgr:     connMngr,
            log:         log,
            staticPeers: staticPeers,
            quitC:       make(chan struct{}),
        }
        out.initStaticPeers()
        if len(conf.StaticPeers) > 0 {
            go out.monitorStaticPeers()
        }

        out.gater = connGtr
        return out, nil
    }
```

#### Block Propagation under Gossip

Gossip is used in distributed systems to ensure data consistency and address issues caused by multicasting. It's a communication protocol where information from one or more nodes is sent to another set of nodes in the network. This is useful when a group of clients in the network simultaneously requires the same data. When the sequencer produces blocks in an unsafe state, they are propagated to other nodes via the gossip network.

Let's first see where the node joins the gossip network. In `op-node/p2p/node.go`, during node initialization, the `init` method is called, which then invokes the JoinGossip method to join the gossip network.

```go
    func (n *NodeP2P) init(resourcesCtx context.Context, rollupCfg *rollup.Config, log log.Logger, setup SetupP2P, gossipIn GossipIn, l2Chain L2Chain, runCfg GossipRuntimeConfig, metrics metrics.Metricer) error {
        …
        // note: the IDDelta functionality was removed from libP2P, and no longer needs to be explicitly disabled.
        n.gs, err = NewGossipSub(resourcesCtx, n.host, rollupCfg, setup, n.scorer, metrics, log)
        if err != nil {
            return fmt.Errorf("failed to start gossipsub router: %w", err)
        }
        n.gsOut, err = JoinGossip(resourcesCtx, n.host.ID(), n.gs, log, rollupCfg, runCfg, gossipIn)
        …
    }
```

Next, let's move to `op-node/p2p/gossip.go`.

Below is a simple overview of the main operations executed within the `JoinGossip` function:

1. **Validator Creation**:
   - `val` is assigned the result of the `guardGossipValidator` function call. This is intended to create a validator for gossip messages, checking the validity of blocks propagated in the network.

2. **Block Topic Name Generation**:
   - The `blocksTopicName` is generated using the `blocksTopicV1` function, which formats a string based on the `L2ChainID` in the configuration (`cfg`). The formatted string follows a specific structure: `/optimism/{L2ChainID}/0/blocks`.

3. **Topic Validator Registration**:
   - The `RegisterTopicValidator` method of `ps` is invoked to register `val` as the block topic's validator. Some configuration options for the validator are also specified, such as a 3-second timeout and a concurrency level of 4.

4. **Joining the Topic**:
   - The function attempts to join the block gossip topic by calling `ps.Join(blocksTopicName)`. If there's an error, it returns an error message indicating the inability to join the topic.

5. **Event Handler Creation**:
   - It tries to create an event handler for the block topic by calling `blocksTopic.EventHandler()`. If there's an error, it returns an error message indicating the failure to create the handler.

6. **Logging Topic Events**:
   - A new goroutine is spawned to log topic events using the `LogTopicEvents` function.

7. **Topic Subscription**:
   - The function attempts to subscribe to the block gossip topic by calling `blocksTopic.Subscribe()`. If there's an error, it returns an error message indicating the inability to subscribe.

8. **Subscriber Creation**:
   - A `subscriber` is created using the `MakeSubscriber` function, which encapsulates a `BlocksHandler` that handles the `OnUnsafeL2Payload` event from `gossipIn`. A new goroutine is spawned to run the provided `subscription`.

9. **Creation and Return of Publisher**:
   - An instance of `publisher` is created and returned, configured to use the provided configuration and block topic.

```go
    func JoinGossip(p2pCtx context.Context, self peer.ID, ps *pubsub.PubSub, log log.Logger, cfg *rollup.Config, runCfg GossipRuntimeConfig, gossipIn GossipIn) (GossipOut, error) {
        val := guardGossipValidator(log, logValidationResult(self, "validated block", log, BuildBlocksValidator(log, cfg, runCfg)))
        blocksTopicName := blocksTopicV1(cfg) // return fmt.Sprintf("/optimism/%s/0/blocks", cfg.L2ChainID.String())
        err := ps.RegisterTopicValidator(blocksTopicName,
            val,
            pubsub.WithValidatorTimeout(3*time.Second),
            pubsub.WithValidatorConcurrency(4))
        if err != nil {	
            return nil, fmt.Errorf("failed to register blocks gossip topic: %w", err)
        }
        blocksTopic, err := ps.Join(blocksTopicName)
        if err != nil {
            return nil, fmt.Errorf("failed to join blocks gossip topic: %w", err)
        }
        blocksTopicEvents, err := blocksTopic.EventHandler()
        if err != nil {
            return nil, fmt.Errorf("failed to create blocks gossip topic handler: %w", err)
        }
        go LogTopicEvents(p2pCtx, log.New("topic", "blocks"), blocksTopicEvents)

        subscription, err := blocksTopic.Subscribe()
        if err != nil {
            return nil, fmt.Errorf("failed to subscribe to blocks gossip topic: %w", err)
        }

        subscriber := MakeSubscriber(log, BlocksHandler(gossipIn.OnUnsafeL2Payload))
        go subscriber(p2pCtx, subscription)

        return &publisher{log: log, cfg: cfg, blocksTopic: blocksTopic, runCfg: runCfg}, nil
    }
```

Thus, a non-sequencer node's subscription is established. Next, let's shift our focus to the nodes in sequencer mode and see how they broadcast the blocks.

`op-node/rollup/driver/state.go`

Within the event loop, it waits for the generation of new payloads in sequencer mode (unsafe blocks) through a loop. Subsequently, this payload is propagated to the gossip network via `PublishL2Payload`.


```go
    func (s *Driver) eventLoop() {
        …
        for(){
            …
            select {
            case <-sequencerCh:
                payload, err := s.sequencer.RunNextSequencerAction(ctx)
                if err != nil {
                    s.log.Error("Sequencer critical error", "err", err)
                    return
                }
                if s.network != nil && payload != nil {
                    // Publishing of unsafe data via p2p is optional.
                    // Errors are not severe enough to change/halt sequencing but should be logged and metered.
                    if err := s.network.PublishL2Payload(ctx, payload); err != nil {
                        s.log.Warn("failed to publish newly created block", "id", payload.ID(), "err", err)
                        s.metrics.RecordPublishingError()
                    }
                }
                planSequencerAction() // schedule the next sequencer action to keep the sequencing looping
                …
                }
        …
        }
        …
    }
```
Thus, a new payload has entered the gossip network.

Within libp2p's pubsub system, nodes first receive messages from other nodes and then check the validity of those messages. If the message is valid and aligns with the node's subscription criteria, the node considers forwarding it to other nodes. Based on certain strategies, such as network topology and the subscriptions of nodes, a node decides whether or not to forward a message. If it decides to forward, the node sends the message to all nodes it's connected to that have subscribed to the same topic. During forwarding, to prevent the message from looping infinitely within the network, there are typically mechanisms in place to track already forwarded messages, ensuring a message isn't forwarded multiple times. Additionally, a message might have a "Time To Live" (TTL) attribute, which defines the number or time a message can be forwarded in the network. Every time a message is forwarded, its TTL value decreases until it's no longer forwarded. Regarding validation, messages typically go through some validation processes, such as checking the message's signature and format, to ensure its integrity and authenticity. In libp2p's pubsub model, this process ensures widespread propagation of messages to many nodes in the network while preventing infinite loops and network congestion, achieving efficient message delivery and processing.

#### Quick Sync via p2p when Blocks are Missing

When a node, due to special circumstances like going down and reconnecting, might end up with some unsynchronized blocks (gaps), it can quickly sync using the p2p network's reverse chain method.

Let's look at the `checkForGapInUnsafeQueue` function in `op-node/rollup/driver/state.go`.

This code segment defines a method named `checkForGapInUnsafeQueue` which belongs to the `Driver` struct. Its purpose is to check if there's a gap in a queue named "unsafe queue" and attempts to retrieve the missing payloads through an alternate sync method named `altSync`. Here's the key point: the method ensures data continuity and tries to fetch missing data from other sync methods when a data gap is detected. The main steps of the function are:

1. The function first gets `UnsafeL2Head` and `UnsafeL2SyncTarget` from `s.derivation` to define the range's starting and ending points for the check.
2. The function checks if there are missing data blocks between `start` and `end` by comparing the `Number` values of `end` and `start`.
3. If a data gap is detected, the function tries to request the missing data range by calling `s.altSync.RequestL2Range(ctx, start, end)`. If `end` is a null reference (i.e., `eth.L2BlockRef{}`), the function requests a sync with an open-ended range starting from `start`.
4. While making the data request, the function logs a debug message detailing the data range it's requesting.
5. The function finally returns an error value. It will return `nil` if no error is present.

```go
    // checkForGapInUnsafeQueue checks if there's a gap in the unsafe queue and tries to retrieve the missing payloads using an alt-sync method.
    // WARNING: This is just an outgoing signal; there's no guarantee that blocks will be retrieved.
    // Results are received through OnUnsafeL2Payload.
    func (s *Driver) checkForGapInUnsafeQueue(ctx context.Context) error {
        start := s.derivation.UnsafeL2Head()
        end := s.derivation.UnsafeL2SyncTarget()
        // Check for missing blocks between start and end and request them if found.
        if end == (eth.L2BlockRef{}) {
            s.log.Debug("requesting sync with open-ended range", "start", start)
            return s.altSync.RequestL2Range(ctx, start, eth.L2BlockRef{})
        } else if end.Number > start.Number+1 {
            s.log.Debug("requesting missing unsafe L2 block range", "start", start, "end", end, "size", end.Number-start.Number)
            return s.altSync.RequestL2Range(ctx, start, end)
        }
        return nil
    }
```

The `RequestL2Range` function passes the beginning and ending signals of the requested blocks to the `requests` channel.

It then distributes the request to the `peerRequests` channel through the `onRangeRequest` method. The `peerRequests` channel is awaited by loops opened by multiple peers, meaning each dispatch is handled by a single peer.

```go
    func (s *SyncClient) onRangeRequest(ctx context.Context, req rangeRequest) {
            …
            for i := uint64(0); ; i++ {
            num := req.end.Number - 1 - i
            if num <= req.start {
                return
            }
            // check if we have something in quarantine already
            if h, ok := s.quarantineByNum[num]; ok {
                if s.trusted.Contains(h) { // if we trust it, try to promote it.
                    s.tryPromote(h)
                }
                // Don't fetch things that we have a candidate for already.
                // We'll evict it from quarantine by finding a conflict, or if we sync enough other blocks
                continue
            }

            if _, ok := s.inFlight[num]; ok {
                log.Debug("request still in-flight, not rescheduling sync request", "num", num)
                continue // request still in flight
            }
            pr := peerRequest{num: num, complete: new(atomic.Bool)}

            log.Debug("Scheduling P2P block request", "num", num)
            // schedule number
            select {
            case s.peerRequests <- pr:
                s.inFlight[num] = pr.complete
            case <-ctx.Done():
                log.Info("did not schedule full P2P sync range", "current", num, "err", ctx.Err())
                return
            default: // peers may all be busy processing requests already
                log.Info("no peers ready to handle block requests for more P2P requests for L2 block history", "current", num)
                return
            }
        }
    }
```

Next, let's look at how a peer handles the request when received.

First and foremost, it's essential to understand that the connection or message passage between the peer and the requesting node is carried out through libp2p's stream. The receiving peer node implements the stream's handling method, while the sending node initiates the stream creation.

From the previous `init` function, we see code snippets like the following. Here, `MakeStreamHandler` returns a handling function. `SetStreamHandler` binds the protocol id with this handling function. Thus, every time the sending node creates and utilizes this stream, the returned handling function is triggered.

```go
    n.syncSrv = NewReqRespServer(rollupCfg, l2Chain, metrics)
    // register the sync protocol with libp2p host
    payloadByNumber := MakeStreamHandler(resourcesCtx, log.New("serve", "payloads_by_number"), n.syncSrv.HandleSyncRequest)
    n.host.SetStreamHandler(PayloadByNumberProtocolID(rollupCfg.L2ChainID), payloadByNumber)
```

Now, let's delve into how the handler function processes the request. 

The function first performs global and personal rate-limiting checks to control the speed of handling requests. It then reads and verifies the block number of the request, ensuring it falls within a reasonable range. Subsequently, the function retrieves the requested block payload from the L2 layer and writes it into the response stream. While writing the response data, it sets a write deadline to prevent being blocked by slow peer connections during the write process. Ultimately, the function returns the requested block number and any potential errors.


```go
    func (srv *ReqRespServer) handleSyncRequest(ctx context.Context, stream network.Stream) (uint64, error) {
        peerId := stream.Conn().RemotePeer()

        // take a token from the global rate-limiter,
        // to make sure there's not too much concurrent server work between different peers.
        if err := srv.globalRequestsRL.Wait(ctx); err != nil {
            return 0, fmt.Errorf("timed out waiting for global sync rate limit: %w", err)
        }

        // find rate limiting data of peer, or add otherwise
        srv.peerStatsLock.Lock()
        ps, _ := srv.peerRateLimits.Get(peerId)
        if ps == nil {
            ps = &peerStat{
                Requests: rate.NewLimiter(peerServerBlocksRateLimit, peerServerBlocksBurst),
            }
            srv.peerRateLimits.Add(peerId, ps)
            ps.Requests.Reserve() // count the hit, but make it delay the next request rather than immediately waiting
        } else {
            // Only wait if it's an existing peer, otherwise the instant rate-limit Wait call always errors.

            // If the requester thinks we're taking too long, then it's their problem and they can disconnect.
            // We'll disconnect ourselves only when failing to read/write,
            // if the work is invalid (range validation), or when individual sub tasks timeout.
            if err := ps.Requests.Wait(ctx); err != nil {
                return 0, fmt.Errorf("timed out waiting for global sync rate limit: %w", err)
            }
        }
        srv.peerStatsLock.Unlock()

        // Set read deadline, if available
        _ = stream.SetReadDeadline(time.Now().Add(serverReadRequestTimeout))

        // Read the request
        var req uint64
        if err := binary.Read(stream, binary.LittleEndian, &req); err != nil {
            return 0, fmt.Errorf("failed to read requested block number: %w", err)
        }
        if err := stream.CloseRead(); err != nil {
            return req, fmt.Errorf("failed to close reading-side of a P2P sync request call: %w", err)
        }

        // Check the request is within the expected range of blocks
        if req < srv.cfg.Genesis.L2.Number {
            return req, fmt.Errorf("cannot serve request for L2 block %d before genesis %d: %w", req, srv.cfg.Genesis.L2.Number, invalidRequestErr)
        }
        max, err := srv.cfg.TargetBlockNumber(uint64(time.Now().Unix()))
        if err != nil {
            return req, fmt.Errorf("cannot determine max target block number to verify request: %w", invalidRequestErr)
        }
        if req > max {
            return req, fmt.Errorf("cannot serve request for L2 block %d after max expected block (%v): %w", req, max, invalidRequestErr)
        }

        payload, err := srv.l2.PayloadByNumber(ctx, req)
        if err != nil {
            if errors.Is(err, ethereum.NotFound) {
                return req, fmt.Errorf("peer requested unknown block by number: %w", err)
            } else {
                return req, fmt.Errorf("failed to retrieve payload to serve to peer: %w", err)
            }
        }

        // We set write deadline, if available, to safely write without blocking on a throttling peer connection
        _ = stream.SetWriteDeadline(time.Now().Add(serverWriteChunkTimeout))

        // 0 - resultCode: success = 0
        // 1:5 - version: 0
        var tmp [5]byte
        if _, err := stream.Write(tmp[:]); err != nil {
            return req, fmt.Errorf("failed to write response header data: %w", err)
        }
        w := snappy.NewBufferedWriter(stream)
        if _, err := payload.MarshalSSZ(w); err != nil {
            return req, fmt.Errorf("failed to write payload to sync response: %w", err)
        }
        if err := w.Close(); err != nil {
            return req, fmt.Errorf("failed to finishing writing payload to sync response: %w", err)
        }
        return req, nil
    }
```

Up to this point, the overall process of reverse chain sync requests and handling has been explained.

#### Scoring Reputation System in p2p Nodes

To prevent certain nodes from making malicious requests and responses that could compromise the security of the entire network, Optimism has also employed a scoring system.

For instance, in the file `op-node/p2p/app_scores.go`, there is a series of functions set up to score peers.

```go
    func (s *peerApplicationScorer) onValidResponse(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementValidResponses{Cap: s.params.ValidResponseCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }

    func (s *peerApplicationScorer) onResponseError(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementErrorResponses{Cap: s.params.ErrorResponseCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }

    func (s *peerApplicationScorer) onRejectedPayload(id peer.ID) {
        _, err := s.scorebook.SetScore(id, store.IncrementRejectedPayloads{Cap: s.params.RejectedPayloadCap})
        if err != nil {
            s.log.Error("Unable to update peer score", "peer", id, "err", err)
            return
        }
    }
```

Then before adding a new node, its points will be checked

```go
    func AddScoring(gater BlockingConnectionGater, scores Scores, minScore float64) *ScoringConnectionGater {
        return &ScoringConnectionGater{BlockingConnectionGater: gater, scores: scores, minScore: minScore}
    }

    func (g *ScoringConnectionGater) checkScore(p peer.ID) (allow bool) {
        score, err := g.scores.GetPeerScore(p)
        if err != nil {
            return false
        }
        return score >= g.minScore
    }

    func (g *ScoringConnectionGater) InterceptPeerDial(p peer.ID) (allow bool) {
        return g.BlockingConnectionGater.InterceptPeerDial(p) && g.checkScore(p)
    }

    func (g *ScoringConnectionGater) InterceptAddrDial(id peer.ID, ma multiaddr.Multiaddr) (allow bool) {
        return g.BlockingConnectionGater.InterceptAddrDial(id, ma) && g.checkScore(id)
    }

    func (g *ScoringConnectionGater) InterceptSecured(dir network.Direction, id peer.ID, mas network.ConnMultiaddrs) (allow bool) {
        return g.BlockingConnectionGater.InterceptSecured(dir, id, mas) && g.checkScore(id)
    }
```

### Conclusion
The high configurability of libp2p allows for a high degree of customization and modularity in the project's p2p system. The above illustrates the primary logic behind Optimism's personalized implementation of libp2p. For further details, one can delve deeper into the p2p directory by examining the source code.
