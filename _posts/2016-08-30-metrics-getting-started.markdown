---
layout: post
title: Metrics Getting Started
categories: Metrics
---
# Getting Started #
Getting Started 将会是一个引导你将Metrics添加到已有的应用程序的过程。我们将会介绍Metrics提供的不同工具，如何使用它们，什么时候它们会派上用场。

# Setting Up Maven #
你需要将`metrics-core`作为依赖包：
{% highlight xml linenos %}
<dependencies>
    <dependency>
        <groupId>io.dropwizard.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>${metrics.version}</version>
    </dependency>
</dependencies>
{% endhighlight %}

{% highlight markdown %}
### 注意
确保你的`metrics.version`为最新的版本：3.1.0
{% endhighlight %}

现在开始在你的程序中添加metrics吧！

# Meters
Meter测量的是一段时间内事件发生的速率(rate)，例如：每秒钟请求的次数。除了全部时间的速率(rate)，meters还会统计最近1分钟，5分钟，15分钟的平均速率。
{%highlight java linenos%}
private final Meter requests = metrics.meter("requests");

public void handleRequest(Request request, Response response) {
    requests.mark();
    // etc
}
{%endhighlight%}
以上的meter将会测量每秒请求的次数

# Console Reporter
Console Reporter 就像看到的那样——在console中记录。以下reporter每秒输出：
{%highlight java linenos%}
ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
       .convertRatesTo(TimeUnit.SECONDS)
       .convertDurationsTo(TimeUnit.MILLISECONDS)
       .build();
   reporter.start(1, TimeUnit.SECONDS);
{%endhighlight%}

# Complete getting started
所以完整的Getting Started为：
{%highlight java linenos%}
package sample;
  import com.codahale.metrics.*;
  import java.util.concurrent.TimeUnit;

  public class GetStarted {
    static final MetricRegistry metrics = new MetricRegistry();
    public static void main(String args[]) {
      startReport();
      Meter requests = metrics.meter("requests");
      requests.mark();
      wait5Seconds();
    }

  static void startReport() {
      ConsoleReporter reporter = ConsoleReporter.forRegistry(metrics)
          .convertRatesTo(TimeUnit.SECONDS)
          .convertDurationsTo(TimeUnit.MILLISECONDS)
          .build();
      reporter.start(1, TimeUnit.SECONDS);
  }

  static void wait5Seconds() {
      try {
          Thread.sleep(5*1000);
      }
      catch(InterruptedException e) {}
  }
}
{%endhighlight%}

{%highlight xml linenos%}
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>somegroup</groupId>
  <artifactId>sample</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <name>Example project for Metrics</name>

  <dependencies>
    <dependency>
      <groupId>io.dropwizard.metrics</groupId>
      <artifactId>metrics-core</artifactId>
      <version>${metrics.version}</version>
    </dependency>
  </dependencies>
</project>
{%endhighlight%}

{%highlight markdown%}
### 注意
确保你的`metrics.version`为最新的版本：3.1.0
{%endhighlight%}

使用以下命令来运行
{%highlight shell%}
mvn package exec:java -Dexec.mainClass=sample.First
{%endhighlight%}

# The Registry
Metrics的核心是`MetricRegistry`类，它是存放应用中所有metrics的容器。通过以下代码创建：
{%highlight java linenos%}
final MetricRegistry metrics = new MetricRegistry();
{%endhighlight%}
你可能想讲它整合到应用程序的生命周期中(可能是用你的依赖注入框架)，但是`static`就足够了。

# Gauges
Gauge可以取到一个变量的瞬时值。例如，我们想要取得队列中的等待任务的个数：
{%highlight java linenos%}
public class QueueManager {
    private final Queue queue;

    public QueueManager(MetricRegistry metrics, String name) {
        this.queue = new Queue();
        metrics.register(MetricRegistry.name(QueueManager.class, name, "size"),
                         new Gauge<Integer>() {
                             @Override
                             public Integer getValue() {
                                 return queue.size();
                             }
                         });
    }
}
{%endhighlight%}
当这个Gauge测量过了，将会返回队列中的任务数。
每个Metrics在registry中都有一个唯一的名字，名字的格式是带点的字符串，例如：`things.count`或`com.example.Thing.latency`。`MetricRegistry`有静态的帮助方法来结构化这些名称：
{%highlight java linenos%}
MetricRegistry.name(QueueManager.class, "jobs", "size")
{%endhighlight%}
上面的方法会返回一个字符串：`com.example.QueueManager.jobs.size`
对于大多数的队列和类队列结构，我们并不想简单的返回`queue.size()`，因为`java.util`和`java.util.concurrent`中实现的`#size()`方法很多都是O(n)的复杂度，这意味着你的gauge性能不会好（会存在潜在的锁）。

# Counters
Conter是gauge的一个`Atomiclong`的实现。你可以increment或者decrement它的值。例如，我们想用一个更有效率的方法来测量队列中等待任务的大小：
{%highlight java linenos%}
private final Counter pendingJobs = metrics.counter(name(QueueManager.class, "pending-jobs"));

public void addJob(Job job) {
    pendingJobs.inc();
    queue.offer(job);
}

public Job takeJob() {
    pendingJobs.dec();
    return queue.take();
}
{%endhighlight%}

每次这个counter被测量，将会返回再给队列中等待的任务。
正如你所看到的，counter的API与其他的有所不同：`#counter(String)`而不是`#register(String, Metric)`。你可以用`register`并创建你自己的`counter`实现，但是`#counter(String)`为你做了所有的工作，它允许你用相同的名字来重新使用metrics。
同时，我们已经静态的导入`MetricRegistry`的`name`方法在它的作用域中来减少混淆。
# Histograms
Histogram测量流数据中的统计学分布的值。除minimum, maximum, mean等，还有中间值，百分位中的75,90,95,98,99以及99.9。
{%highlight java linenos%}
private final Histogram responseSizes = metrics.histogram(name(RequestHandler.class, "response-sizes"));

public void handleRequest(Request request, Response response) {
    // etc
    responseSizes.update(response.getContent().length);
}
{%endhighlight%}
这个histogram将会测量responses的byte大小。

# Timers
Timer会测量一特定代码块的速率和其持续时间的分布。也就是说是 Histogram 和 Meter 的结合， histogram 某部分代码/调用的耗时， meter统计TPS。
{%highlight java linenos%}
private final Timer responses = metrics.timer(name(RequestHandler.class, "responses"));

public String handleRequest(Request request, Response response) {
    final Timer.Context context = responses.time();
    try {
        // etc;
        return "OK";
    } finally {
        context.stop();
    }
}
{%endhighlight%}
以上的timer会测量处理每个请求的耗时之和(纳秒)，并提供一个每秒请求的平均速率。

# Health Checks
使用`metrics-healthchecks`模块的话，Metrics也有集中你的服务的健康检查的能力。
第一步，创建一个新的`HealthCheckRegistry`实例：
{%highlight java linenos%}
final HealthCheckRegistry healthChecks = new HealthCheckRegistry();
{%endhighlight%}
第二步，实现`HealthCheck`的子类：
{%highlight java linenos%}
public class DatabaseHealthCheck extends HealthCheck {
    private final Database database;

    public DatabaseHealthCheck(Database database) {
        this.database = database;
    }

    @Override
    public HealthCheck.Result check() throws Exception {
        if (database.isConnected()) {
            return HealthCheck.Result.healthy();
        } else {
            return HealthCheck.Result.unhealthy("Cannot connect to " + database.getUrl());
        }
    }
}
{%endhighlight%}
然后，注册实例到Metrics：
{%highlight java linenos%}
healthChecks.register("postgres", new DatabaseHealthCheck(database));
{%endhighlight%}
运行所有注册的Health checks:
{%highlight java linenos%}
final Map<String, HealthCheck.Result> results = healthChecks.runHealthChecks();
for (Entry<String, HealthCheck.Result> entry : results.entrySet()) {
    if (entry.getValue().isHealthy()) {
        System.out.println(entry.getKey() + " is healthy");
    } else {
        System.err.println(entry.getKey() + " is UNHEALTHY: " + entry.getValue().getMessage());
        final Throwable e = entry.getValue().getError();
        if (e != null) {
            e.printStackTrace();
        }
    }
}
{%endhighlight%}
Metrics还可以通过pre-built health check: `ThreadDeadlockHealthCheck` 来检查线程是否死锁。