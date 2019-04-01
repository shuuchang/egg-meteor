# egg-meteor的仓库
https://github.com/shuuchang/egg-meteor
# egg-meteor的设计背景
在国网中干了几年统一权限，吸取了其优秀的部分设计，也吸取了其有些过时的部分设计，结合以往经验，渐渐的萌生了一个新的类似于统一权限的程序设计方案，充分利用postgresql及其fdw插件、mariadb及其集群的功能实现数据的流转，业务逻辑与页面由egg.js实现，如出现数据流转稳定性不足，估计还需要使用kettle实现数据的辅助流转。

## 先说说统一权限
因为涉及版权，在此只能描述一下与本项目有关的部分。现在很多使用软件支撑服务的业主（运营商、社保、国网及其他大型企业），基本会同时存在很多不同架构、不同语言、不同数据库的很多系统，虽然可以解决很多风险（毕竟每个架构、语言、数据库都有不能支撑的业务）、但是也带来的许多问题，数据碎片化、各系统之间兼容等。

统一权限这个系统解决的是一个账号在各个系统中都能用的情况。国网正式员工只有一个工作账号，但需要在多个系统中进行操作，所以设计一套系统专门对账号、组织结构等进行管理，与统一权限功能极为相近的开源项目是著名的论坛Discuz的辅助项目UCenter（用户中心，似乎已停止开发）。

在此我描述一下UCenter（实在是不能过多的说统一权限，人家毕竟不是开源的），UCenter是用PHP+mysql的架构开发的，当时是利用UCenter支撑，实现Discuz+的设计，比如Discuz+ecshop是当时很流行的，估计现在还有一些小网站还会用这个架构支撑业务。UCenter可以将Discuz的账号及其登录状态传递给ECShop等接入的系统，实现只要登录Discuz就可以无缝登录其他接入系统，对用户来说似乎是同一个系统，据我使用UCenter的经验要是接口支撑完整，还可以将业务系统的代办通知通过UCenter传递给Discuz来显示。

也就是说UCenter用户中心主要的功能就是获取所接入系统的账号及其登录保持信息，传递给其他接入系统，实现一个系统登录、所有系统保持的基本功能，更进一步的功能是所有接入系统都有统一的入口，比如Discuz一样的门户入口。在这样的一个系统支撑下，可以统一各个不同语言、架构设计的系统，在用户使用时感觉是一个庞大的系统。

## 开发UCenter相似的系统需要解决的技术问题
基于经验，设计一个UCenter技术问题主要在于接口设计与如何适配多种多样架构的业务系统。

在与业务系统进行账号及其登录状态信息的传递时如何设计接口，最省事的方法是提供接口服务，业务系统基于我方提供的一系列接口标准调取数据，用户要访问业务系统就调接口直接获取数据。当然，并不是所有系统都会按照接口标准获取数据，他们总会提一些超出标准的要求，我们要不要响应呢。

还是因为要接入的系统存在多种多样，虽然可以用接口解决许多，任然有接口无法支撑的情况，我们设计的UCenter如何支撑呢。

所以在设计egg-meteor时想了许多方案，最后确定这这样的一个方案，egg-meteor使用postgresql来作为配置数据库与归档数据库，用egg.js实现页面与相关配置逻辑。配置完成后使用mysql_fdw插件将数据同步至mariadb集群中，再基于mariadb集群对外提供服务。这是一种服务端与客户端分离，使用postgresql与mariadb集群各自的优势，由postgresql支撑服务端，实现数据的配置、同步，由mariadb集群支撑客户端对外服务。干了很长时间的实施运维，代码对我来说还是存在难度的，辛亏对各个数据库有较为深刻的了解，反而可以充分利用数据库及其插件的优势，让我尽量在代码上少花点时间。

## postgresql及其插件
postgresql是一个和Oracle功能一样强大的开源数据库，可能就是因为开源，所以他的各种插件是所有数据库中最为成熟的一个。在这里我们主要使用的fdw（foreign data wrapper，外部数据装饰器），比较成熟的是面向[mysql（mariadb）](https://github.com/EnterpriseDB/mysql_fdw)、[MongoDB](https://github.com/EnterpriseDB/mongo_fdw)、[Hadoop（HDFS）](https://github.com/EnterpriseDB/hdfs_fdw)，通过映射，可以在postgresql中像操作内部表一样，对映射后的mysql表进行CURD操作。

postgresql的fdw插件详细描述在postgresql官网有较为清晰的描述，请参照[Foreign Data Wrappers](http://wiki.postgresql.org/wiki/Foreign_data_wrappers)

## MriaDB及其集群
MriaDB是mysql的分支版本，已经开始拥有了自己的版本体系，使用与mysql基本一致，MriaDB程序中兼容了集群模块，安装后可直接进行集群设置，mysql不带集群模块，需要另外安装。

MriaDB的集群使用全冗余集群，即每台单机数据库中都储存着整个集群数据。这样可以每台单机MriaDB部署一个客户端接口程序，负载均衡不管分发至任何一个MriaDB集群单节点，都会是完整的客户端。如果其中某个节点异常将不会影响其他节点。

## egg.js
egg.js是由阿里对node.js进一步封装形成的框架，继承并优化了node.js的模块。具体资料请查看[egg.js](https://eggjs.org/zh-cn/)

通过对我所掌握的语言的了解程度，最终确定了egg.js为egg-meteor的页面实现、逻辑与接口实现的编程语言。

## egg-meteor-server设计的部分补充
因为长期从事实施运维，在开发上功力尚浅，所以在设计过程中尽量使用各数据库自身诸多优势，所用框架也是尽可能简单的实现所需功能的egg.js。

这个设计中也有一个不稳定因素，就是fdw的稳定性尚未在其他生产环境测试，如稳定性不足，则使用kettle支撑。

#相关环境部署
### [postgresql安装部署](https://www.postgresql.org/download/linux/redhat/)
### [mysql_fdw安装部署](https://www.cnblogs.com/ctypyb2002/p/9793125.html)
### [MariaDB集群安装部署](https://www.cnblogs.com/oneapm/p/4617637.html)
### node.js安装

```
$ curl -sL https://rpm.nodesource.com/setup_10.x | bash -
$ yum install -y nodejs
$ node -v
$ npm -v
```


### egg.js安装（[eggjs](https://eggjs.org/zh-cn/intro/quickstart.html)）

```
$ npm i egg-init -g
$ egg-init egg-example --type=simple
$ cd egg-example
$ npm i
```
