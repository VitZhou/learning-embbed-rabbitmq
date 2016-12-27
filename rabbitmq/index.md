# Embbed RabbitMq
这个库允许使用各种RabbitMQ版本，它是一个可以在JVM中控制的嵌入式服务。
它的工作方式是通过从官方知识库下载给定版本和操作系统的正确工件，解压缩它并使用指定的配置启动RabbitMQ服务器。 然后可以通过使用对rabbitmqctl或rabbitmq-plugins的等效命令从JVM内管理代理。

## 环境依赖
- 需要java 7+
- RabbitMQ Broler需要安装Erlang

## 快速开始
### maven依赖:
```xml
    <dependency>
      <groupId>io.arivera.oss</groupId>
      <artifactId>embedded-rabbitmq</artifactId>
      <version>X.Y.Z</version>
    </dependency>
```
X.Y.Z是这个项目的最新发布版本。 有关更多信息，请访问[Maven Central repo](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22io.arivera.oss%22%20AND%20a%3A%22embedded-rabbitmq%22)或[访问releases](https://github.com/AlejandroRivera/embedded-rabbitmq/releases)页面。

对于SNAPSHOT版本，将SonaType存储库添加到构建系统：https://oss.sonatype.org/content/repositories/snapshots/

### 启动RabbitMQ broker
```java
    EmbeddedRabbitMqConfig config = new EmbeddedRabbitMqConfig.Builder().build();
    EmbeddedRabbitMq rabbitMq = new EmbeddedRabbitMq(config);
    rabbitMq.start();
```
当调用start（）时，Embedded-RabbitMQ库将从RabbitMQ.com下载与您的操作系统最匹配的最新版本。 该组件将被解压缩到一个临时文件夹，并且一个新的操作系统进程将启动RabbitMQ代理。

更多关于[如何自定义您的RabbitMQ代理]可以阅读定制章节。

### 验证RabbitMQ是否正常工作
```java
    ConnectionFactory connectionFactory = new ConnectionFactory();
    connectionFactory.setHost("localhost");
    connectionFactory.setVirtualHost("/");
    connectionFactory.setUsername("guest");
    connectionFactory.setPassword("guest");

    Connection connection = connectionFactory.newConnection();
    assertThat(connection.isOpen(), equalTo(true));
    Channel channel = connection.createChannel();
    assertThat(channel.isOpen(), equalTo(true));

    channel.close();
    connection.close();
```

### 停止RabbitMq Broker:
```java
	rabbitMq.top
```

## 定制
可以通过EmbeddedRabbitMqConfig和它的Builder类来完成定制。 下面的所有代码段都将引用：
```java
	EmbeddedRabbitMqConfig.Builder configBuilder = new EmbeddedRabbitMqConfig.Builder();
    ...
    EmbeddedRabbitMqConfig config = configBuilder.build();
    EmbeddedRabbitMq rabbitMq = new EmbeddedRabbitMq(config);
    rabbitMq.start();
```

## 定义要使用的RabbitMQ版本：
```java
	configBuilder.version(PredefinedVersion.LATEST)
```
或
```java
	configBuilder.version(PredefinedVersion.V3_6_5)
```
通过使用version（）方法，将为Unix / Mac / Windows操作系统预设下载URL，可执行文件路径等。 您可以使用downloadFrom（）方法从官方的rabbitmq.com服务器和Github更改下载源：
```java
	configBuilder.downloadFrom(OfficialArtifactRepository.GITHUB)
```
同样，如果您希望下载其他版本和/或使用其他服务器：
```java
    String url = "https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_6_milestone1/rabbitmq-server-mac-standalone-3.6.5.901.tar.xz";
    configBuilder.downloadFrom(new URL(url), "rabbitmq_server-3.6.5.901")
```

## 下载文件
默认情况下，EmbeddedRabbitMq将尝试将下载的文件保存到〜/ .embeddedrabbitmq。 您可以通过使用downloadTarget（）setter（接受目录或文件）来更改此设置：
```java
	configBuilder.downloadTarget(new File("/tmp/rabbitmq.tar.xz"))
    ...
	// configBuilder.downloadTarget(new File("/tmp"))
```
> 如果具有相同名称的文件已存在，它将被覆盖。

此库的默认行为是重复使用以前下载的文件。 如果您不想使用该行为，请禁用它：
```java
	configBuilder.useCachedDownload(false)
```
为确保损坏或部分下载的文件不被重复使用，默认行为是在检测到问题时将其删除。 这意味着下次下载新的副本。 要禁用此行为，请执行以下操作：
```java
	configBuilder.deleteDownloadedFileOnErrors(false)
```

## 提取路径
EmbeddedRabbitMq将解压缩下载的文件到一个临时文件夹。 您可以指定您自己的文件夹，如：
```java
	configBuilder.extractionFolder(new File("/rabbits/"))
```
> 此文件夹的内容将被每次新提取的文件/文件夹覆盖。

## 高级RabbitMQ管理
如果您希望进一步控制RabbitMQ代理，可以在/ bin目录中执行任何可用的命令，如下所示：
```java
	RabbitMqCommand command = new RabbitMqCommand(config, "command", "arg1", "arg2", ...);
    StartedProcess process = command.call();
    ProcessResult result = process.getFuture().get();
    boolean success = result.getExitValue() == 0;
    if (success) {
      doSomething(result.getOutput());
    }
```
注意:
- 命令是类似“rabbitmq-ctl”（在Windows中不需要.bat扩展名，因为它将被自动附加）。
- args是可变长度数组列表，其中每个元素是一个单词（例如“-n”，“nodeName”，“list_users”）

有关RabbitMqCommand和其他辅助类（如RabbitMqCtl，RabbitMq插件和RabbitMq服务器）的更多信息，请参阅JavaDocs，旨在使其更容易执行常用命令。

## 启用RabbitMQ插件：
要启用像rabbitmq_management这样的插件，可以使用RabbitMqPlugins类，如下所示：
```java
	RabbitMqPlugins rabbitMqPlugins = new RabbitMqPlugins(config);
    rabbitMqPlugins.enable("rabbitmq_management");
```
该调用将阻塞，直到命令完成。

您可以通过执行list（）方法来验证：
```java
	 Map<String, Plugin> plugins = rabbitMqPlugins.list();
    Plugin plugin = plugins.get("rabbitmq_management");
    assertThat(plugin, is(notNullValue()));
    assertThat(plugin.getState(), hasItem(Plugin.State.ENABLED_EXPLICITLY));
    assertThat(plugin.getState(), hasItem(Plugin.State.RUNNING));
```
您还可以通过调用groupedList（）来查看哪些其他插件被隐式启用：
```java
	 Map<Plugin.State, Set<Plugin>> groupedPlugins = rabbitMqPlugins.groupedList();
    Set<Plugin> plugins = groupedPlugins.get(Plugin.State.ENABLED_IMPLICITLY);
    assertThat(plugins.size(), is(not(equalTo(0))));
```