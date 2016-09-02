# neutron中的QOS功能

对于QOS来说，主要的功能包括限速、入口队列映射、流量监管、流量整形和出口队列调度，在neutron中已经开始支持QOS的功能，[具体参见](https://wiki.openstack.org/wiki/Neutron/Metering/Bandwidth),
但是仅局限于QoS meter等基本概念，主要目的在于router和每条traffic的级别上通过设置iptables的设置来对流量进行限速。


neutron中QOS的使用场景：

- 于公网带宽、端口带宽、广播域中的风暴抑制
- 报文转发延迟保证
- 基于业务流的队列调度
- VXLAN 的BUM 问题



公网带宽可以通过虚拟化技术中的端口带宽限制或开发Neutron 中的相应功能来提供，但是通过VM 虚拟化技术的端口限速来限制带宽的使用做法太过彻底，在限速南北流量的
同时将VM 之间的东西流量也进行了限速，从而影响了某些东西方向业务流量的正常传输。虽然所需QoS 功能在传统物理网络中大都比较基础且有成熟技术方案，但是在云计算网络里，这些功能却是比较费周折才能实现。如何保证Neutron 根据业务提供适合的QoS 功能是将来OpenStack 产品化的一个重点，
这个点的需求和SDN 有非常好的切合之处。可以结合OpenDayLight 等SDN 控制器，通过SDN 架构对Neutron 的底层网络流量进行控制，来实现流量均衡和流级别的限速。





