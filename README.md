一、产生背景
单体应用
网站流量很小时，将所用功能部署在一起，减少部署的成本。此时，可采用 ORM 框架来简化增删改查工作量。
问题：
不同业务纠缠在一起，这种开发模型和架构不利于快速发展的业务。
• 架构分化，不同业务团队对于开发细节存在差异
• 开发效率，不同团队的业务发展的快慢会导致发布的频度不一致
• 可用性，一个团队的问题可能导致整个应用挂掉，难以提升稳定性
垂直应用
随着访问量增大，单一应用通过增加机器带来的性能提升越来越小。此时，将应用拆分成不互相干的应用，提升效率。为加速前端页面的开发，MVC 设计模式是关键。
分布式服务架构
当垂直应用越来越多，应用交互不可避免，将核心业务抽取出来的作为独立的服务，逐渐成为稳定的服务中心。应用之间的通信方式不能依赖本地，必须走远程，因此需要一个高效、稳定的 RPC 框架（跨越了运输层和应用层）
SOA 框架
当服务越来越多，容量的评估，小服务资源的浪费问题逐渐显现。此时需要增加一个调度中心，进行资源调度和服务治理：
• 透明化的远程方法调用，像本地调用般进行远程调用
• 软负载均衡及容错机制
• 服务注册中心自动注册及配置管理
• 服务接口监控与治理
HSF 应用场景
HSF 本身就是就是对应用服务化进行支撑的框架。
解决业务快速发展带来的技术问题：单一应用 -> 多应用，本地调用 -> 远程调用
二、优秀特性
高性能服务调用
低入侵
基于Java接口完成透明的PRC调用，用户对服务是否在本地不感知，不入侵用户代码
高性能
提供基于非阻塞 I/O 的高性能调用
多语言
提供了C++以及nodejs客户端，支持HTTP REST调用
大流量的应对场景
client端负载均衡
HSF在客户端基于服务地址列表做负载均衡，不需要借助其他负载均衡设备，高效完成负载均衡任务
多种地址选择
以server重启这个场景为例，HSF提供了基于可用地址的选址策略
• 当发现地址的链接不可用时，会暂时将改地址移出地址列表并尝试恢复链接
• 这样既保证了平滑调用，又能使 server在重启完成后对应的地址被加回列表
上下线策略
HSF 提供优雅的上下线的能力，保证服务在重启时对client影响最小
• client调用在server重启时表现平滑
全方位的服务治理
服务管理功能
HSF OPS 运维平台提供了：服务查询、测试、mock功能，支持用户通过服务名（一般时接口名）查询，通过输入参数调用
规则管理功能
HSF OPS 运维平台支持使用归组、路由以及同机房等规则对client发起的调用进行干预
三、基本结构
注册中心
起到协调的作用（中介），接收 Provider 发布的地址，并将地址信息推送给服务消费方
Provider
本地运行项目（启动服务），会向地址注册中心发布地址
Consumer
根据服务名向地址注册中心订阅服务地址，Consumer 可以从地址列表选取一个地址发起 RPC 调用
HSFOPS
可以指定服务进行测试，在平台中可以通过输入项目地址，测试对应的功能。通过 ip 地址可指定 Provider 调用
服务方在启动后，会向地址注册中心发布地址
Consumer 根据服务名向 ConfigServer 订阅服务地址，当服务地址推送到服务消费方后，服务消费方就可以从地址列表中选择一个地址，发起RPC调用；
Provider在发布地址的同时会将服务元信息发布到元数据存储中心 redis，HSF OPS 通过访问 redis 向使用者展示服务的详情，同时还可以通过 Diamond 和 ConfigServer 查询服务信息和规则信息
四、一次调用过程
HSF 一次调用会从 Consumer 发起，经过网络抵达 Provider，再将结果通过网络携带回来，最终返回给用户。这个过程会涉及：
• 多个线程交互
• HSF 中不同的领域对象
Consumer：
• 请求通信对象对应的是HSF协议，包含诸如请求id等多个与请求对象无关的内容
• I/O 线程中会完成编码，最终通过网络发送给 Provider
• 此时 client 线程会等待结果返回，处于 waiting 状态
Provider：
• 通过反射机制去调用服务
五、客户端调用方式
HSF 的 I/O 操作都是异步的。
同步调用
客户端默认是采取同步的方式调用，本质是通过 future.get(timeout) 操作，等待服务端的结果返回
对于客户端来说，并不是所有的 HSF 服务都需要同步等待 server 的返回结果，HSF 提供异步调用的形式，让用不不必同步阻塞在 HSF 操作上。
• 异步调用时，HSF 服务的调用返回值都是默认值（int 0，Object null）
• 真正的结果，HSFResponseFuture 或者回调函数中获取
@Configuration
public class HsfConfiguration {
    @HSFConsumer
    private NetworkCalcSimulationService networkCalcSimulationService;
    @HSFConsumer
    private NetWorkCalcFacade netWorkCalcFacade;
}
//在需要使用的地方 @Autowired 即可
@Autowired
NetworkCalcSimulationService networkCalcSimulationService;
Future异步调用
HSF 发起调用后，用户可以在上下文中获取跟返回结果关联的 HSFFuture 对象
• 用户可在任意时候调用，HSFFuture.getResponse(timeout) 获取服务端的返回结果
@Configuration
public class HsfConfig {
    @HSFConsumer(serviceVersion = "1.0.0", serviceGroup = "HSF", futureMethods = "sayHelloInFuture")
    OrderService orderService;
}
//使用时注入即可
@Autowired
OrderService orderService;
Callback 异步调用
Client 配置为 callback 方式调用时，需要配置一个实现了HSFResponseCallback 接口的 Listener，结果返回后，HSF 会调用 callback 里面的方法
使用注解 @AsyncOn 指明作用到的 interface 和 method
@AsyncOn(interfaceName = OrderService.class, methodName = "queryOrder")
public class CallbackHandler implements HSFResponseCallback {
    //业务异常时会触发
    @Override
    public void onAppException(Throwable t) {
        t.printStackTrace();
    }
    //业务返回结果
    @Override
    public void onAppResponse(Object result) {
        // 取callback调用时设置的上下文
        Object context = CallbackInvocationContext.getContext();
        System.out.println(result.toString() + context);
    }
    //HSF 异常    
    @Override
    public void onHSFException(HSFException e) {
        e.printStackTrace();
    }
}
六、编码及发布
服务端代码
1）将 interface 定义在一个工程中，会打包成一个 jar 包，发布到 maven 仓库中
2）服务端根据业务逻辑实现 interface，发布对应的服务，Consumer 通过 (pom 坐标) 依赖 jar 包，通过 HSF 远程调用服务端
服务发布
1）在项目的 pom 文件中添加 HSF 依赖 starter
2）通过加入 @HSFProvider  配置到实现类
3) 运行项目，将服务通过 HSF 发布出去
调用端代码
1）在项目的 pom 文件中添加 HSF 依赖 starter
2）在需要调用的地方，使用@HSFConsumer 注解，由于 Consumer 可能多个地方使用，可以写一个 Config 类，在需要的地方使用 @Autowired 注入即可
http://mw.alibaba-inc.com/products/hsf/_book/mw-docs/quick-start.html?spm=a1zco.hsf.0.0.36c019b5oztvj8
七、OPS 查询、测试服务
HSF 服务发布后，可通过 HSF 运维系统查询
• 服务名、IP、应用名
日常环境：http://hsf.alibaba.net/hsfops/serviceSearch.htm?envType=daily
com.cainiao.global.lnp.web.facade.res.PickupCoverageFacade:1.0.0.daily（serviceGroup.serviceInterface:version）
HSF 服务测试、这里如果landlord没有设置默认租户，测试的时候要传入 RPC Context，不然会没有租户
