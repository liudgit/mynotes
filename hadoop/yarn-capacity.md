
Capacity Scheduler
===================================
Capacity Scheduler是YARN中默认的资源调度器，但是在默认情况下只有root.default 一个queue。而当不同用户提交任务时，任务都会在这个队里里面按优先级先进先出，大大影响了多用户的资源使用率。

Capacity 配置介绍
----------------------------------
#### 1.首先在yarn-site.xml 配置调度器为Capacity

> <property>
	<name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
  </property>
  
#### 2. 配置

- yarn.scheduler.capacity.root.default.maximum-capacity,队列最大可使用的资源率。
- capacity：队列的资源容量（百分比）。 当系统非常繁忙时，应保证每个队列的容量得到满足，而如果每个队列应用程序较少，可将剩余资源共享给其他队列。注意,所有队列的容量之和应小于100。
- maximum-capacity：队列的资源使用上限（百分比）。由于存在资源共享，因此一个队列使用的资源量可能超过其容量,而最多使用资源量可通过该参数限制。
- user-limit-factor：单个用户最多可以使用的资源因子，默认情况为1，表示单个用户最多可以使用队列的容量不管集群有空闲，如果该值设为5，
表示这个用户最多可以使用5capacity的容量。实际上单个用户的使用资源为
- min(user-limit-factorcapacity，maximum-capacity)。这里需要注意的是，如果队列中有多个用户的任务，那么每个用户的使用量将稀释。
- minimum-user-limit-percent：每个用户最低资源保障（百分比）。任何时刻，一个队列中每个用户可使用的资源量均有一定的限制。
当一个队列中同时运行多个用户的应用程序时中,每个用户的使用资源量在一个最小值和最大值之间浮动，其中，最小值取决于正在运行的应用程序数目，
而最大值则由minimum-user-limit-percent决定。比如，假设minimum-user-limit-percent为25。当两个用户向该队列提交应用程序时，
每个用户可使用资源量不能超过50%，如果三个用户提交应用程序，则每个用户可使用资源量不能超多33%，如果四个或者更多用户提交应用程序,则每个用户可用资源量不能超过25%。
- maximum-applications ：集群或者队列中同时处于等待和运行状态的应用程序数目上限，这是一个强限制，一旦集群中应用程序数目超过该上限，后续提交的应用程序将被拒绝，默认值为10000。
所有队列的数目上限可通过参数yarn.scheduler.capacity.maximum-applications设置（可看做默认值）,而单个队列可通过参数
yarn.scheduler.capacity..maximum-applications设置适合自己的值。
- maximum-am-resource-percent：集群中用于运行应用程序ApplicationMaster的资源比例上限,该参数通常用于限制处于活动状态的应用程序数目。
该参数类型为浮点型，默认是0.1,表示10%。所有队列的ApplicationMaster资源比例上限可通过参数yarn.scheduler.capacity. maximum-am-resource-percent设置（可看做默认值），而单个队列可通过参数yarn.scheduler.capacity.. maximum-am-resource-percent设置适合自己的值。
- state ：队列状态可以为STOPPED或者RUNNING，如果一个队列处于STOPPED状态，用户不可以将应用程序提交到该队列或者它的子队列中，类似的，如果ROOT队列处于STOPPED状态，
用户不可以向集群中提交应用程序，但正在运行的应用程序仍可以正常运行结束，以便队列可以优雅地退出。
- acl_submit_applications：限定哪些Linux用户/用户组可向给定队列中提交应用程序。需要注意的是，该属性具有继承性,即如果一个用户可以向某个队列中提交应用程序，
则它可以向它的所有子队列中提交应用程序。配置该属性时，用户之间或用户组之间用“，”分割，用户和用户组之间用空格分割，比如“user1, user2 group1,group2”。
- acl_administer_queue：为队列指定一个管理员,该管理员可控制该队列的所有应用程序,比如杀死任意一个应用程序等。同样该属性具有继承性，
如果一个用户可以向某个队列中提交应用程序,则它可以向它的所有子队列中提交应用程序。

#### 3.使用队列

mapreduce:在Job的代码中，设置Job属于的队列,例如hive：

conf.setQueueName("hive");

hive:在执行hive任务时，设置hive属于的队列,例如dailyTask:

set mapred.job.queue.name=dailyTask;

#### 4.刷新配置

生产环境中，队列及其容量的修改在现实中是不可避免的，而每次修改，需要重启集群，这个代价很高，如果修改队列及其容量的配置不重启呢: 1.在主节点上根据具体需求，修改好mapred-site.xml和capacity-scheduler.xml 2.把配置同步到所有节点上yarn rmadmin -refreshQueues 这样就可以动态修改集群的队列及其容量配置，不需要重启了，刷新mapreduce的web管理控制台可以看到结果。 注意:如果配置没有同步到所有的节点，一些队列会无法启用

