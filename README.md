# Project: StaticBackend

A simple go backend structure for universial purpose.

[Legacy offical README](./README_legacy.md)

## DDD-based Design Thought
- DDD-Domain Driven Design 被非常广泛地应用到了很多项目当中。可能它不是被非常规范地设计，但是几乎所有的成熟 Go 项目都应用了这种思想因为 Go 本身的语言特性非常适合实现 DDD 的设计思想
- 本质上包含了：模型 + 分层包 + 抽象接口
- 比如下面提到的比较重要的缓存+pub/sub就用到了DDD的设计思路，将接口根据业务抽象出来，并且和具体的实现分离。
- [Domain-Driven Design DDD - GeeksforGeeks](https://www.geeksforgeeks.org/domain-driven-design-ddd/)
- [Domain-Driven Design DDD - Fundamentals - Redis](https://redis.io/glossary/domain-driven-design-ddd/)

- ## Usage of Middleware Chain
- 本质上也是函数式编程的一种典型实现 #functional-programming
	- 代码复用 + 关注点分离
	- 模块化自由组合
	- 请求处理管道化
	- 具体体现出的一些思路：
		- **将横切关注点与核心业务逻辑分离**: 识别出在多个路由处理函数中都需要执行的通用逻辑，例如认证、授权、日志记录、错误处理、请求预处理等。将这些逻辑抽象成独立的中间件。
		  logseq.order-list-type:: number
		- 构建可重用的组件：将这些抽象出来的逻辑封装成独立的、可复用的中间件函数。
		  logseq.order-list-type:: number
		- **使用函数式编程的概念**: middleware.Chain 本身可能就是一个高阶函数，它接受多个函数（中间件）并返回一个新的函数（处理链）。这体现了函数式编程中组合和管道的概念。
		  logseq.order-list-type:: number
		- **约定优于配置**: 通过约定中间件的执行顺序和参数传递方式，简化配置，提高开发效率。
		  logseq.order-list-type:: number
		- **关注请求的生命周期**: 中间件的核心思想是拦截和处理 HTTP 请求的生命周期中的不同阶段。
		  logseq.order-list-type:: number
	- ```go
	  // Chain creates a request pipeline from which the Middleware are chained
	  // together and h is the last Handler executed.
	  func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
	  	for i := len(middlewares) - 1; i >= 0; i-- {
	  		h = middlewares[i](h)
	  	}
	  	return h
	  }
	  
	  pubWithDB := []middleware.Middleware{
	  		middleware.Cors(),
	  		middleware.WithDB(backend.DB, backend.Cache, getStripePortalURL),
	  	}
	  
	  	stdAuth := []middleware.Middleware{
	  		middleware.Cors(),
	  		middleware.WithDB(backend.DB, backend.Cache, getStripePortalURL),
	  		middleware.RequireAuth(backend.DB, backend.Cache),
	  	}
	  
	  	stdRoot := []middleware.Middleware{
	  		middleware.WithDB(backend.DB, backend.Cache, getStripePortalURL),
	  		middleware.RequireRoot(backend.DB, backend.Cache),
	  	}
	  
	  http.Handle("/me", middleware.Chain(http.HandlerFunc(m.me), stdAuth...))
	  http.Handle("/password/resetcode", middleware.Chain(http.HandlerFunc(m.setResetCode), stdRoot...))
	  http.Handle("/password/reset", middleware.Chain(http.HandlerFunc(m.resetPassword), pubWithDB...))
	  ```

- ## SSE vs Socket
- 核心在于对于 server-client 的双向连接是如何确定和实现的
	- [轮询、SSE 和 webSocket](https://juejin.cn/post/7086104218259423268)
	- [[WebSocket, SSE and TCP]]
	- **WebSockets:** 提供**全双工 (full-duplex)** 的通信通道。这意味着一旦客户端和服务器之间建立连接，双方可以**同时**向对方发送数据。
		- 本质是使用 websocket 协议
		- 有着相对较高的开销，要处理消息头以及维护双向连接的状态
	- **SSE (Server-Sent Events):** 提供**单向 (unidirectional)** 的通信通道，从服务器到客户端。客户端发起连接后，服务器可以**单方面**向客户端推送数据。
		- 本质是使用 HTTP 协议
		- 有相对较低的开销，但更主要是实现容易，依赖的库比较少，且和具体的业务场景高度相关
	- SSE Broker 实现：
		- 主要用于连接管理和消息分发，本质上是用了 go 的 channel 机制作为消息分发的中枢
	- ```go
	  
	  // NewBroker returns a ready to use Broker for accepting web socket connections
	  func NewBroker(v Validator, pubsub cache.Volatilizer, log *logger.Logger) *Broker {
	  	b := &Broker{
	  		Broadcast:          make(chan model.Command, 1),
	  		newConnections:     make(chan ConnectionData),
	  		closingConnections: make(chan chan model.Command),
	  		clients:            make(map[chan model.Command]string),
	  		ids:                make(map[string]chan model.Command),
	  		conf:               make(map[string]context.Context),
	  		subscriptions:      make(map[string][]chan bool),
	  		validateAuth:       v,
	  		pubsub:             pubsub,
	  		log:                log,
	  	}
	  
	  	go b.start()
	  
	  	return b
	  }
	  
	  func (b *Broker) start() {
	  	for {
	  		select {
	  		case data := <-b.newConnections:
	  			id, err := uuid.NewUUID()
	  			if err != nil {
	  				b.log.Error().Err(err)
	  			}
	  
	  			b.clients[data.messages] = id.String()
	  			b.ids[id.String()] = data.messages
	  			b.conf[id.String()] = data.ctx
	  
	  			msg := model.Command{
	  				Type: model.MsgTypeInit,
	  				Data: id.String(),
	  			}
	  
	  			data.messages <- msg
	  		case c := <-b.closingConnections:
	  			b.unsub(c)
	  		case msg := <-b.Broadcast:
	  			clients, payload := b.getTargets(msg)
	  			for _, c := range clients {
	  				c <- payload
	  			}
	  		}
	  	}
	  }
	  ```

- ## Event Based PubSub: A simple production and consumption model
- ### Interface
	- 本质上这个项目中是通过一个大的 interface 来涵盖了所有 cache 相关+pub/sub 的函数定义
		- Pros
			- 用例和需求的数据都紧密相关
			- 简化底层基础设施的实现，增强统一性，更倾向于整体使用 DDD，将这部分看作一个整体 Domain
		- Cons
			- 把两个核心组件的函数耦合到了一个结构体实现中，能够统一替换同时也带来了过于耦合的构造
			- 但同时也设计了能够替换 pub/sub 组件的能力，这部分的真实定义和构造在实现中其实是一个嵌套关系：
				- `CacheDev(Observer)`, 而`Observer`就是仅处理全部 pub/sub 部分底层实现的结构
				- 到此我感觉还很难去说这个架构到底是不是一个非常合理的架构逻辑，其看似耦合，但内部又通过嵌套实现了解耦 #TODO
	- ```go
	  // Volatilizer is the cache and pub/sub interface
	  type Volatilizer interface {
	  	// Get returns a string value from a key
	  	Get(key string) (string, error)
	  	// Set sets a string value
	  	Set(key string, value string) error
	  	// GetTyped returns a typed struct by its key
	  	GetTyped(key string, v any) error
	  	// SetTyped sets a typed struct for a key
	  	SetTyped(key string, v any) error
	  	// Inc increments a numeric value for a key
	  	Inc(key string, by int64) (int64, error)
	  	// Dec decrements a value for a key
	  	Dec(key string, by int64) (int64, error)
	  	// Subscribe subscribes to a pub/sub channel
	  	Subscribe(send chan model.Command, token, channel string, close chan bool)
	  	// Publish publishes a message to a channel
	  	Publish(msg model.Command) error
	  	// PublishDocument publish a database message to a channel
	  	PublishDocument(auth model.Auth, dbname, channel, typ string, v any)
	  	// QueueWork add a work queue item
	  	QueueWork(key, value string) error
	  	// DequeueWork dequeue work item (if available)
	  	DequeueWork(key string) (string, error)
	  }
	  ```
	- ```go
	  // CacheDev used in local dev mode and is memory-based
	  type CacheDev struct {
	  	data     map[string]string
	  	log      *logger.Logger
	  	observer observer.Observer
	  	m        *sync.RWMutex
	  }
	  
	  // NewDevCache returns a memory-based Volatilizer
	  func NewDevCache(log *logger.Logger) *CacheDev {
	  	return &CacheDev{
	  		data:     make(map[string]string),
	  		observer: observer.NewObserver(log),
	  		log:      log,
	  		m:        &sync.RWMutex{},
	  	}
	  }
	  
	  type Observer interface {
	  	Subscribe(channel string) Subscriber
	  	Publish(channel string, msg interface{}) error
	  	Unsubscribe(channel string, subscriber Subscriber) error
	  	PubNumSub(channel string) map[string]int
	  }
	  
	  type memObserver struct {
	  	Subscriptions map[string][]*memSubscriber
	  	mx            sync.Mutex
	  	log           *logger.Logger
	  }
	  
	  func NewObserver(log *logger.Logger) Observer {
	  	subs := make(map[string][]*memSubscriber)
	  	return &memObserver{Subscriptions: subs, mx: sync.Mutex{}, log: log}
	  }
	  ```
- ### Workflow
	- 1. 每个消息通道都有独立的 key，可以看作一个 dict
	- 2. Subscribe 时会创建一个和消息通道同域的 subscriber，将其存入字典（生产者）的同时返回给消费者
	- 3. 消费者只需从 subscriber 里的 ch 拿东西即可，生产者只需往同样的 subscriber 的 ch 存东西即可
	- 4. 在此基础上顶层需要封装一层使用函数，用于动态确定订阅的消息通道是哪个，以及取出来的消息转存到哪里消费（另外一个 ch）
	- hierarchy
		- Top level package:[Volatilizer](./cache/volatilizer.go)
		- Middle level impl for different service usage:[injection_impl](./cache/dev.go)
		- Base level memory-based pub/sub impl:[mem-impl](./cache/observer/observer.go)
	- #### Base Producer
		- 在 observer 中实现对应的 publish 方法，一旦有新的消息过来，就会 publish 给对应 channel 的全部订阅者
		- ```go
		  func (o *memObserver) Publish(channel string, msg interface{}) error {
		  	o.mx.Lock()
		  	defer o.mx.Unlock()
		  	if len(o.Subscriptions[channel]) == 0 {
		  		return errors.New("not subscribers for chan: " + channel)
		  	}
		  	for _, sub := range o.Subscriptions[channel] {
		  		//s := sub
		  
		  		go func(msub *memSubscriber, msg any) {
		  			timer := time.NewTimer(15 * time.Second)
		  			select {
		  			case msub.msgCh <- msg:
		  				if !timer.Stop() {
		  					<-timer.C
		  				}
		  			case <-timer.C:
		  				o.log.Error().Msg("the previous message is not read; dropping this message")
		  				timer.Stop()
		  			}
		  		}(sub, msg)
		  
		  	}
		  	return nil
		  }
		  ```
	- #### Base Conusmer
		- 在订阅之后，通过 goroutine 时刻监控对应的 channel
		- ```go
		  func (d *CacheDev) Subscribe(send chan model.Command, token, channel string, close chan bool) {
		  	pubsub := d.observer.Subscribe(channel)
		  
		  	ch := pubsub.Channel()
		  
		  	for {
		  		select {
		  		case m := <-ch:
		  			var msg model.Command
		  			if err := json.Unmarshal([]byte(m.(string)), &msg); err != nil {
		  				d.log.Error().Err(err).Msg("error parsing JSON message")
		  				_ = pubsub.Close()
		  				_ = d.observer.Unsubscribe(channel, pubsub)
		  				return
		  			}
		  
		  			// TODO: this will need more thinking
		  			if msg.Type == model.MsgTypeChanIn {
		  				msg.Type = model.MsgTypeChanOut
		  			} else if msg.IsSystemEvent {
		  
		  			} else if msg.IsDBEvent() && !d.HasPermission(token, channel, msg.Data) {
		  				continue
		  			}
		  			send <- msg
		  		case <-close:
		  			_ = pubsub.Close()
		  			_ = d.observer.Unsubscribe(channel, pubsub)
		  			return
		  		}
		  	}
		  }
		  ```
	- ### Broker
		- Broker 一定是跟外部进行对接的。且内部对接完全不应该使用 broker，既会带来性能的损耗，又会使内部接口与流程混乱 [Broker Code](./realtime/broker.go)
		- 所以此 project 内的 broker 是用来跟 client 对接的
			- 1. NewConnections
			- 2. CloseConnections
			- 3. Broadcast
		- 这个项目中的 broker 的核心功能是将消息发送给全部的 clients，且输入和输出完全解耦，我不用管是什么协议连接进来的，所有 broker 接到和发出的内容都通过内部的管道化 channel 实现
			- ```go
			  func (b *Broker) start() {
			  	for {
			  		select {
			  		case data := <-b.newConnections:
			  			id, err := uuid.NewUUID()
			  			if err != nil {
			  				b.log.Error().Err(err)
			  			}
			  
			  			b.clients[data.messages] = id.String()
			  			b.ids[id.String()] = data.messages
			  			b.conf[id.String()] = data.ctx
			  
			  			msg := model.Command{
			  				Type: model.MsgTypeInit,
			  				Data: id.String(),
			  			}
			  
			  			data.messages <- msg
			  		case c := <-b.closingConnections:
			  			b.unsub(c)
			  		case msg := <-b.Broadcast:
			  			clients, payload := b.getTargets(msg)
			  			for _, c := range clients {
			  				c <- payload
			  			}
			  		}
			  	}
			  }
			  ```

- ## Why Cache Is So Important?
	- All in one: 能够显著提升应用程序的性能、可伸缩性、降低成本并提高用户体验。
	- 本质上是为了更好地降低在几乎任何条件下的性能瓶颈
	- 从简单缓存 ---> 复杂高效缓存
		- In-memory
		- 本地缓存
		- 多级缓存
	- 从单机缓存 ---> 云原生情景下的缓存
		- 分布式缓存
		- Service Mesh
		- 数据一致性
		- 监控和告警（命中率，延迟，错误率等）