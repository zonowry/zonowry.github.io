---
title: 关于代码结构设计的一些思考
date: 2022-12-22
tags: [DDD, 设计模式]
toc: true
categories: 知识
description: "网上冲浪时经常能看到微服务、分布式等技术分享。但我们大多数开发面向的都是政企小群体用户，工作中很难实操这些技术。 也就在我们与高大上工程之间筑起了知识障壁。为了能理解高大上的技术，我们从简单的开始分析。

所以接下来我们不聊庞大的工程设计，来看看如何从我们日常的工作中学习这些模式的底层思想。
希望这篇文章能带来一点代码结构设计的启发，这层障壁或许会变薄一些。"

---

    其实本文标题我想命名为《通过思考一些低端代码结构设计来领略高大上技术知识这件事》来着

## 生疏而难理解的工程

网上冲浪时经常能看到微服务、分布式等技术分享。但我们大多数开发面向的都是政企小群体用户，工作中很难实操这些技术。 也就在我们与高大上工程之间筑起了知识障壁。为了能理解高大上的技术，我们从简单的开始分析。

所以接下来我们不聊庞大的工程设计，来看看如何从我们日常的工作中学习这些模式的底层思想。
希望这篇文章能带来一点代码结构设计的启发，这层障壁或许会变薄一些。

<!-- more -->

---

## 从三层架构开始

    最简单的架构，最单纯的思想。

### 为什么要分层

或者说为什么要分层，工作经验使我们能很快的说出是为了方便维护等。分层划分了代码职责，给编码一点约束，增加了点编码难度，但带来层次清晰的架构。我们来看看怎么通过分层得到一个容易维护的代码，或者说为什么分层可以让代码容易维护。

### 容易维护的代码

`页面/表示层`、`业务逻辑层`、`数据访问层`，三层简单到一句话就可以概括：“各层划分了一下责任，自上而下单向依赖”。按照此模式实现代码过于简单，我们不屑于实现（双手叉腰.jpg）。所以来聊点高大上的架构思路，对了解大型项目，写出容易维护的代码很有帮助。

可以想象如果不分层（划分代码），各种代码写在一个函数中，等到需求变更需要更新代码逻辑时。就像在听歌时需要先掏出牛仔裤裤兜中的有线耳机一样。

为什么会有这种感觉，因为各种职责的代码在一个函数中纵横交错。所以我们通常会根据职责划分代码（分层）。下面是一个简单的 http 后台接口代码，处理用户注册请求。

```kotlin
// 1. 接收浏览器发出的请求
@PostMapping("/user/register")
fun registerUser(
    // 2. 提取请求体中的参数数据
	@RequestParam name: String,
	@RequestParam userRole: UserRole
) {
    // 3. 业务逻辑：用户名不能重复
    val isExists = database.query("select * from user where name=?1", name) != null
    if (isExists) {
	    throw RuntimeException("用户名已存在")
    }

    // 4. 业务逻辑：不能创建管理员角色
	if (isAdminRole(userRole)) {
		throw RuntimeException("不允许注册为管理员用户")
	}
	// 5. 业务逻辑：注册用户（创建用户对象）
	val newUser = User(name)
	user.password = encrypt(randomPwd())
	user.registerTime = Instant.now()
	user.role = userRole

	// 6. 持久化
	database.save("insert into user values(?1)", user)
}
```

以上代码明显是一个面向过程式的写法。在项目简单时，没有任何问题。但是参与的人变多，项目代码量变大，会面临很多问题（代码重用、逻辑分散等）。

所以我们尝试分析这块代码的职责，可以简单的划分一下哪些一些业务逻辑代码，哪些不是业务逻辑代码。

- 首先函数声明部分帮我们从 http 请求中提取参数值以及将此方法映射成请求的处理方法，依靠的 `spring boot` 框架来实现。
- 函数体部分是一些逻辑安全性校验和根据业务逻辑（随机密码，注册时间，用户角色）创建一个用户并保存到数据库中
- 除此之外，还有一些明显的 SQL 语句对验证数据以及持久化数据提供支持

分析完毕，首先我们把函数声明部分的注解，依赖 `spring` 技术框架的地方放到页面层，供外部（前端页面）调用。

然后我们新增一个 `UserService` 服务类放到业务层，它有一个 `registry(xxx)` 方法。我们将业务逻辑部分代码放到该方法中。

然后是持久化 `sql` 脚本，我们业务逻辑层只依赖一个数据访问接口，这个接口的具体实现就是这些 sql。

最终大概写成这样就是简单的三层架构。

```kotlin

// 界面层
// 文件：UserController.kt
class UserController {
	private val userService: UserService
	// 1. 接收浏览器发出的请求
	@PostMapping("/user/register")
	fun registerUser(
	    // 2. 提取请求体中的参数数据
		@RequestParam name: String,
		@RequestParam userRole: UserRole
	) {
		// 3. 调用业务层方法
		this.userService.registry(name, userRole)
	}
}

// 业务逻辑层
// 文件：UserService.kt、或者是 UserBLL.kt
class UserService {
	private val userDao: UserDao
	fun registry(name: String, role: UserRole) {
		// 业务逻辑：用户名不能重复
	    if (this.userDao.findByName(name) != null ) {
		    throw RuntimeException("用户名已存在")
	    }

	    // 业务逻辑：不能创建管理员角色
		if (isAdminRole(userRole)) {
			throw RuntimeException("不允许注册为管理员用户")
		}
		// 业务逻辑：注册用户（创建用户对象）
		val newUser = User(name)
		user.password = encrypt(randomPwd())
		user.registerTime = Instant.now()
		user.role = userRole

		// 持久化
		this.userDao.save(user)

}

// 数据访问层
// 文件：UserDao.kt
interface UserDao {
	fun findByName(name: String): User?
	fun save(user: User)
}

// 文件：UserDaoImpl.kt
class UserDaoImpl : UserDao {
	// Sql 脚本实现
}


```

可见三层架构真的很简单，就像是把过程式的代码分开放了。我们主要理解我们是在划分职责就好，这样每块代码不牵扯多余地任务，只负责干好“自己”的事情，分层架构的优点：

- 单向依赖
- 隔离不同职责的代码块

## 面向接口

上面说了这么多分层相关地代码，我们不难发现，这些思想都离不开面向接口开发。我们首先抽出来接口，随后再考虑实现，用什么数据库、性能是否可以接受、使用什么加密算法等等。这些实现大多时候相较于我们的业务都不是很重要。面向接口开发，我们就可一再前期把我们的重点放在对业务的思考上。

### 接口的好处

接口的好处太多，不过都需要亲身实践过、使用接口受益过，才能体会到接口的种种优点。大家常说的一个优点，项目需要从 Mysql 数据库迁移到 Oracle 数据库。如果我们基于面向接口，确实是一件简单地事情。

```kotlin
interface UserDAL {
    // 向数据库插入一个用户对象
    inserUser(user: User)
}


class UserBLL {

	private val dal : UserDAL = MysqlUserDAL()

	fun addUser() {
		dal.inserUser(xxx)
	}
}

// 需求变动, 改成使用Oracle数据库, 我们实现接口,
class UserBLL {

    private val dal : UserDAL = OracleUserDAL()

    fun addUser() {
        dal.inserUser(xxx)
    }
}
```

我们只替换一下实现，就可以迁移。虽然例子可能不是很好，但却是体现了面向接口的一个好处。

### 依赖倒置（依赖注入）

还是上面这个例子，我们发现我们抽象的还是不够彻底，我们在 BLL（业务层）还是能够看到 Oracle，Mysql 之类的技术实现细节。此时我们就可以通过接口的依赖倒置（依赖注入）来进一步抽象。我们可以改成这样

```kotlin
interface UserDAL {
    // 向数据库插入一个用户对象
    inserUser(user: User)
}


class UserBLL(private val dal : UserDAL) {
  fun addUser() {
      dal.inserUser(xxx)
  }
}

// 需求变动, 改成使用Oracle数据库, 我们实现接口, 有变化吗?

class UserBLL(private val dal : UserDAL) {
    fun addUser() {
        dal.inserUser(xxx)
    }
}

//  在项目的某处项目配置代码中
UserBLL(MysqlUserDAL())
UserBLL(OracleUserDAL()
```

可以发现，数据访问对象被我们声明成从构造函数中传入，我们编写业务逻辑代码时，就不再需要考虑用的是什么数据库了，降低了心智负担。

## 模型层

三层架构对付很多小项目足够了，业务变大后，我们 BLL 层互相之间也会出现重用逻辑。我们会习惯性的封装各种对象 `Model` 。想必大家都叫作 `UserUtils` 之类的吧。可以将这个模型取一个更侧重业务的名字，例如上面提到的 `class User`。即形成了模型层，在 DDD 中也被称作领域层。

### 业务场景、业务模型

我们将 `registry` 方法放到 User 类中。业务层转变为应用层，负责编排业务模型的方法。就初步形成了类似 DDD 模式的四层架构。我们的 Service（应用层）就只剩下了：

```kotlin
class UserService {
	fun registry(name: String, role: UserRole) {
		// 用户名重名判断
		if (this.userDao.findByName(name) != null)  {
			return
		}
		val user = new User(name, role)
		user.registry()
		this.userDao.save(user)
	}
}
```

还是不够明显，我们增加一个用户可以同意订单的业务场景。

```kotlin
class UserBLL {
    fun accpetOrder(orderId:Long,  userId:Long) {
        val user = userDal.get(userId)
        if(!user.role.toList().contains("ADMIN")) {
            throw "非管理员不允许同意订单"
        }

        val order = orderDal.get(orderId)
        order.accpet = true
        order.accpetBy = user.id
        orderDal.update(order)
    }
}
```

这段代码牵扯了两个对象，我们有个判断用户是否是管理员的逻辑，这个逻辑看起来以后会重用，我们可以抽成一个方法。只有管理员可以同意订单，这个业务逻辑也应该是订单自己的“业务知识”，和用户没太大关系，不过用户可以对外提供自己是否是管理员的证明。

我们就可以改成这样：

```kotlin
class User {
    fun isAdmin(): Boolean {
        return roles.toList().contains("ADMIN")
    }
}

class Order {
    fun accpet(user: User) {
        if(!user.isAdmin) {
            throw "非管理员不允许同意订单"
        }
        val order = orderDal.get(orderId)
        order.accpet = true
        order.accpetBy = user.id
    }
}


class OrderBLL {
    fun accpetOrder(orderId:Long,  userId:Long) {
        val order = orderDAL.get(orderId)
        val user = userDAL.get(userId)
        order.accpet(user)
        orderDal.update(order)
    }
}
```

这样是否 BLL 业务逻辑层看起来好点了，似乎只用到了两个对象。不过此时业务逻辑层没有了业务逻辑，变成了应用层。业务逻辑都内聚到了模型中。

最终从用户（调用者）方看，主要就是调用了一个 order.accpet(user) 方法而已，就完成了”同意订单”这个业务。我们需要调整逻辑时，也只需要查看 Order 类的业务方法。

## 结尾

再深入一些其实就是领域驱动的内容了，如何建模模型层，以及会遇见的一些问题。和 DDD 中代码三难问题的取舍。

- 领域模型的纯度
- 领域模型的完整度
- 性能

大部分情况下，我们在以上三个特性中，只能同时满足 2 个特性。所以回到工作中，我们只要确保自己有根据职责划分代码，时刻思考代码结构就好。最后放一个简单地分层图片，也是我们大多情况下使用的分层模式吧。很多开源大项目也都是基于这些分层的思考设计的。

![DDD-layer.png](/images/blog/DDD-layer.png)
