---
layout: post
title: Jersey 中的依赖注入
tags:
    - dependency injection
    - jersey
    - hk2
    - java
    - inversion of control
commments: true
---

[Jersey][jersey framework] 是 Java API for RESTful Service (JAX-RS) 的参考实现。Jersey 利用 Java 中的标注（Annotion），声明式地定义每个资源的路径、数据类型，并能自动把 Java bean 转换成定义的数据类型，同时支持从指定的 Java packages 中发现资源类；这些功能使开发者能够免去手工编写初始化代码、解析 JSON 结构等重复性的劳动，专注于业务逻辑的编程。

## 背景

Jersey 使用类来表示表示资源。加在类上的标注 [@Path(value)][@Path reference] 用来声明这是一个资源类，并通过 `value` 来指定该资源的路径。资源类中用 `@GET`, `@POST`, `@PUT` 等标注的方法用来响应该资源对应类型的请求。Jersey 会为每个请求创建资源类的一个对象实例。

每个请求响应方还可以用 `@Consumes` 和 `@Produces` 指定接受的 Content-Type 和返回的 Content-Type. 关于资源类和 Content-Type 的细节请参见 User Guide 中的[第3章][UG jaxrs-resources]和[第9章][UG media].

当 Jersey 在 servlet container 中运行时，需要在 deployment descriptor `web.xml` 中把 `servlet-class` 指定为 `org.glassfish.jersey.servlet.ServletContainer`, 然后在 `init-param` 设定 `jersey.config.server.provider.packages` 为包含资源类的包，这样 Jersey 就会自动在指定的包中寻找带有 `@Path` 标记的类并加载。

## 依赖注入

上述自动查找资源类的功能，正是通过控制反转和依赖注入框架 [hk2][hk2 framework] 实现的。其实资源类也要依靠依赖注入来访问 URL 中的请求路径参数、查询参数，在需要直接访问 Servlet 提供的功能时，也要通过依赖注入获得。而 RESTful 服务的开发者（作为 Jersey 框架的用户）编写的资源类也常常会有复杂的依赖关系，当然最常见的依赖就是数据库啦。所以 Jersey 也提供了注册新服务的接口。

对于向 hk2 中注册新服务这一部分的 API, Jersey 的文档里写的非常简单。而 hk2 的 User Guide 更是简单得没法用。在开始用这个功能时，自己遇到了很多坑。写这篇文章就是为了整理一下在 Jersey 正确使用依赖注入的方法。

## 基本概念

在 hk2 中，有 `@Service` 和 `@Contract` 两个概念。每当使用标注 `@Inject` 请求依赖注入的时候，请求的类型就应该是某个 `@Service`. 而 `@Contract` 则是 `@Service` 的一个实现。

所以 `@Service` 应该是一个接口或一个基类，而 `@Contract` 则是某个 `@Service` 的子类。

## Tutorial

下面将以一个例子来说明这套框架的正确使用方法。这个例子中，有一个资源 `counter`, 会在每次收到 POST 请求时把计数器 +1. 而这个计数器的值存放在 redis 中。所以我们的资源类要依赖 Jedis 来访问这个计数器。

### 请求注入

我们需要在资源类中使用 `@Inject` 请求框架注入一个 Jedis 对象进来。`@Inject` 可以标注在实例变量、构造函数或 setter 函数上。例如：

{% highlight java %}
package info.yangli.example.restful;

@Path("counter")
public class CounterResource {
    @Inject
    private jedis;

    @Inject
    public void setRng(SecureRandom rng) { mRng = rng; }
    protected SecureRandom getRng() {    return mRng;  }
    private mRng;

    public CounterResource() {}

    //public CounterResource(@Inject Bomb bomb) { mBomb = bomb; }
    //private mBomb;      // Caution: It may explode!

    @POST
    public Response doPost() {
        mJedis.incr("myapp:counter", rng.next(8));
        return Response.status(204).build();
    }
}
{% endhighlight %}

简单的情况直接注入到实例变量就可以了。注入到构造函数或者 setter 函数的设计，主要是可以方便在单元测试时传递 mock 对象。

### 注册服务

服务需要在 `Application` 对象构造时被注册，所以需要继承一个 `Application` 的子类，一般我们从 `ResourceConfig` 继承。`ResourceConfig` 类提供了很多方便我们配置资源类和依赖注入的方法。

为了让 Jersey 知道我们的 `Application` 子类，`web.xml` 中需要给 servlet 增加一个 `init-param`: `javax.ws.rs.Application`, 并设置为那个子类。这里我们以 `info.yangli.example.restful.Application` 为例。

`ResourceConfig` 的 `register` 方法接受一个 `AbstractBinder` 类型的参数，用于注册新的服务。而我们要做的是覆盖 `AbstractBinder` 中的 `configure()` 方法，利用 `bind`, `bindAsContract` 和 `bindFactory` 中的一个来注册我们自己的依赖注入关系。

{% highlight java %}
package info.yangli.example.restful;

public Application extends ResourceConfig {
    public Application() {
        packages("info.yangli.example.restful");    //设置自动查找资源类的包

        register(new AbstractBinder() {
            @Override
            protected void configure() {
                bindAsContract(SecureRandom.class).in(java.inject.Singleton.class);
            }
        });

        register(new RedisBinder());
    }
}

class RedisBinder extends AbstractBinder implements Factory<Jedis> {
    protected JedisPool pool;

    public RedisBinder() {
        pool = new JedisPool(new JedisPoolConfig(), "localhost");
    }

    @Override
    public void dispose(Jedis arg0) {
        arg0.close();
    }

    @Override
    public Jedis provide() {
        return pool.getResource();
    }

    @Override
    protected void configure() {
        bindFactory(this).to(Jedis.class).in(RequestScoped.class);
    }
}
{% endhighlight %}

上面的代码注册了两个服务，一个是 `SecureRandom`, 标准库里的安全随机数生成器，注册到 singleton scope, 也就是所有请求注入这个 service 的地方会被注入 `SecureRandom` 的同一个实例。

注册的第二个服务是 `Jedis`. 由于 `Jedis` 不是线程安全的，所以我们写了一个工厂类来 `provide` `Jedis` 对象，用 `bindFactory` 把这个工厂类绑定到 `Jedis` 类上，由这个工厂对象来产生和 terminate `Jedis` 对象。这个服务是注册到 `RequestScoped` 里的，这意味这每个请求会从工厂对象获得一个新的 `Jedis` 对象，并在请求结束时由工厂对象来 `close`, 把连接还给连接池 `pool`.

`AbstractBinder` 中还有一个 `bind(Class<T> service).to(Class<? super T> contract)` 的方法，但由于这个例子中并没有复杂的继承关系，所以没有用上这个方法。

在 Jersey 能用的 scope 中，除了 `Singleton` 和 `RequestScoped`, 还有一个 `PerLookup`. 这是 hk2 定义的，根据文档是在每次查找依赖的时候都新建一个实例。但是在 Jersey 中，使用工厂类时，PerLookup scoped 的对象并不能被 dispose, 所以我也没有用。

## License

你可以在程序和文档中使用本文中的两段源代码，无须取得我的许可。

这些源代码并未经过任何测试，可能是错误的，可能不能按照你的预期工作，可能包含安全漏洞，USE IT AT YOUR OWN RISK!

[jersey framework]: https://jersey.java.net/ "Jersey - RESTful Web Service in Java."
[@Path reference]: http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/Path.html "Annotation Type Path"
[UG jaxrs-resources]: https://jersey.java.net/documentation/latest/jaxrs-resources.html "Chapter 3. JAX-RS Application, Resources and Sub-Resource"
[UG media]: https://jersey.java.net/documentation/latest/media.html "Chapter 9. Support for Common Media Type Representations"
[hk2 framework]: https://hk2.java.net/ "HK2 - A light-weight and dynamic dependency injection framework."

