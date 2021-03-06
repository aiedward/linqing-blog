# 如何保证 Meteor 应用的安全性？

[How to secure your Meteor app](https://guide.meteor.com/security.html) 本文为翻译，发现错误请指正，谢谢。

阅读完本文，你将能够了解：

1. Meteor 应用的安全。
2. 如何保证 Meteor 方法，发布和源码的安全性。
3. 在部署和生产中往哪里存储密钥。
4. 如何按照一个安全性检查表来审核应用。

## 简介
保证一个应用的安全就是理解安全域和安全域之间的攻击界面。在一个 Meteor 应用中，其实很简单：

1. 服务器上运行的代码可以被信任。
2. 其他：客户端上运行的代码，通过 Method 和 publication 参数传输的数据等，都是不能被信任的。

实际应用中，应该在这两个域之间的边界做大量的安全性和验证性的工作。简单来说：

1. 验证和检验所有来自客户端的输入。
2. 不要把任何私密信息泄露到客户端。

## 概念：攻击界面
大部分 Meteor 应用都是把客户端代码文件和服务器端代码文件放在一起，所以应该格外注意哪些是在客户端运行的，哪些是在服务器端运行的，两种的界限是什么。下面是 Meteor 应用安全性检查的完整列表：

1. Methods: 任何以 Method 参数带进来的数据都应该被验证有效性， Method 也不应该返回用户没有权限获得的数据。
2. Publications: 任何以 publication 参数带进来的数据都应该被验证有效性，publication 也不应该返回用户没有权限获得的数据。
3. Served files: 应该确保运行在客户端的源码或配置文件不能包含私密数据。

上面讲到的这些点我们下面会详细讲解：

## 避免使用 allow/deny
在这个教程中，我们非常不支持直接在客户端使用 allow 和 deny 查询 MongoDB 的数据。原因在上面的安全性完整列表。验证所有 MongoDB 数据库查询是非常困难的，随着 MongoDB 版本的更新，遇到的困难可能会更多。
有好几篇文章讲到在客户端执行 MongoDB 数据库更新的潜在缺陷，特别是 Allow & Deny 安全性挑战 和它的 results，可以在 Discover Meteor 博客上找到。

按照上面讲到的，我们推荐所有的 Meteor 应用都应该使用 Method 操作来自客户端的数据，并严格限制每个 Method 的参数。

下面的代码片段放在服务器端拒绝所有客户端对数据集的更新。这可以保证应用中也不可以使用 allow：
```
// 拒绝所有客户端对数据集的更新
Lists.deny({
  insert() { return true; },
  update() { return true; },
  remove() { return true; },
});
```
## Methods
Method 是 Meteor 应用从外部接收输入和数据的方式，所以从应用安全性来讲是非常重要的。如果没有保证 Method 的安全性，用户可能会通过你不希望的方式修改数据库——修改其他用户的文件，删除数据，或者搞乱数据库进而搞垮你的应用。

### 验证所有的参数
如果输入是正确的，那么写简洁的代码就会更容易，所有在运行任何代码前验证所有 Method 的参数是很重要的。你不会希望用户输入不正确的数据类型进而导致程序崩溃。

假设你正在写 Method 的单元测试，你需要检查所有可能的 Method 数据输入；验证参数可以把需要做单元测试的输入限定在一个范围内，减少代码量。自我记录也是很好的一个习惯；其他开发者可以通过查看你的代码了解 Method 所要求的参数类型。

下面这个例子可以说明不验证参数的话，可能会带来灾难性后果：
```
Meteor.methods({
  removeWidget(id) {
    if (! this.userId) {
      throw new Meteor.Error('removeWidget.unauthorized');
    }

    Widgets.remove(id);
  }
});
```
如果用户传递一个非 ID 的选择器，如 {}，则整个数据库都会被删除。
```
mdg:validated-method
```
为了帮助你写成全面验证参数的 Method，我们给 Method 写了一个闭包来强制要求参数验证。在Methods 章节了解如何使用。接下来我们写的代码都是在安装了这个包的基础上。如果没有，这些原则也适用只是写出来的代码会有点怪。

### 不要从客户端传递 userId
Meteor Method 的 this 语境可以包含一些当前连接的有用信息，最有用的是`this.userId`。这个属性是 DDP 登录系统在管理，由框架本身保证其安全性。

当前用户的用户 ID 可以根据 this 获取，所以不应该将用户的 ID 作为参数传递给 Method。因为这将使得任何客户端可以传递任何用户 ID。我们看下面这个例子：
```
// #1: 错误！客户端可以传递任何用户的 ID 并更改其姓名。
setName({ userId, newName }) {
  Meteor.users.update(userId, {
    $set: { name: newName }
  });
}

// #2: 正确，客户端只可以设置当前登录用户的名字。
setName({ newName }) {
  Meteor.users.update(this.userId, {
    $set: { name: newName }
  });
}
```
需要传递任何用户 ID 作为参数的只有下面几种情况：

1. 只有管理员才可以接触到的 Method，用于编辑其他用户资料。请查看用户角色章节了解如何判断用户的角色和权限。
2. 不用于修改其他用户的 Method，而是作为目标；例如，该 Method 可以用于发送私信，或者添加其他用户为好友。

### 每个行为用一个 Method
确保应用安全最好的办法就是理解所有的输入都可能来自不安全的渠道，所以要处理好这些输入。要了解什么样的输入可以来自客户端的最简单的方法就是将其限制在尽可能小的空间中。这意味着应用中的 Method 应该都是有具体行为的，而且不应该有多种选择可以改变这种行为。这样做的目标就是我们可以简单查看应用中的 Method 并且验证和测试它来确保安全性。下面是 Todos 应用中一个安全的 Method：
```
export const makePrivate = new ValidatedMethod({
  name: 'lists.makePrivate',
  validate: new SimpleSchema({
    listId: { type: String }
  }).validator(),
  run({ listId }) {
    if (!this.userId) {
      throw new Meteor.Error('lists.makePrivate.notLoggedIn',
        'Must be logged in to make private lists.');
    }

    const list = Lists.findOne(listId);

    if (list.isLastPublicList()) {
      throw new Meteor.Error('lists.makePrivate.lastPublicList',
        'Cannot make the last public list private.');
    }

    Lists.update(listId, {
      $set: { userId: this.userId }
    });

    Lists.userIdDenormalizer.set(listId, this.userId);
  }
});
```
你可以看到这个 Method 在做一件很具体的事——将单个列表标记为私有。还可以使用另外一个 Method setPrivacy，也可以设置列表为公开或私有，但是在这个应用中对于 makePrivate 和 makePublic 的安全性考虑是很不同的。通过将操作分为不同的 Method，会更加简单清楚。从上面 Method 的定义可以清楚知道我们接受的参数，如何检测安全性，以及对于数据库的操作。
但是，这并不是 Method 的灵活性很差。我们来看一个例子：
```
const Meteor.users.methods.setUserData = new ValidatedMethod({
  name: 'Meteor.users.methods.setUserData',
  validate: new SimpleSchema({
    fullName: { type: String, optional: true },
    dateOfBirth: { type: Date, optional: true },
  }).validator(),
  run(fieldsToSet) {
    Meteor.users.update(this.userId, {
      $set: fieldsToSet
    });
  }
});
```
上面的是一个很好的 Method 因为你可以掌控灵活性，可以有一些可选择的域但是只传递希望改变的域。这个方法之所以可行是因为设置用户的全名和生日的安全性考虑是一样的——我们不需要对不同的域设置不同的安全验证。请注意在 MongoDB 执行 $set 查询是在服务器产生的——我们不应该在客户端执行同样的 MongoDB 查询，因为这样很难验证，而且很可能会带来副作用。

### 重构以重新使用安全性规则
在应用中可能会遇到多个 Method 使用同样的安全性验证。这可以通过将安全性验证代码分离出来成为一个模块，封装 Method 本身，或者拓展 Mongo.Collection 类，然后在服务器端的 insert, update 和 remove 里面检测。但是，通过特定的 Method 实施客户端的沟通会比从客户端发送随意的 update 操作要好，因为恶意用户不能发送未经检查的 update 操作。

### 速率限制
跟 REST 端点一样，Meteor Methods 可以在任何地方调用——一个恶毒的项目，浏览器控制台的脚本等。这可以很容易在短时间内启动多个 Method 调用。这意味着黑客可以很轻易地测试所有的输入，然后找到关键输入。Meteor 的密码登录有内置的速率限制来防止暴力破解，但是你可以为你的其他方法设置速率限制。

在 Todos 应用中，我们使用下面的代码给所有的 Method 设置一个基础速率限制：
```
// 获取列表的所有 method 名称
const LISTS_METHODS = _.pluck([
  insert,
  makePublic,
  makePrivate,
  updateName,
  remove,
], 'name');

// 每个链接每秒钟只允许 5 个列表操作
DDPRateLimiter.addRule({
  name(name) {
    return _.contains(LISTS_METHODS, name);
  },

  // 每个链接 ID 的速率限制
  connectionId() { return true; }
}, 5, 1000);
```
这将使得每个 Method 每秒钟每个链接只能调用 5 次。用户不应该注意到这种速率限制，但是可以阻止恶意脚本请求充斥服务器。你可以根据你的应用需要调整速率限制参数。

## Publications
Publications 是 Meteor 服务器提供数据给客户端的主要方式。虽然使用 Method 主要考虑是确保用户不能按照我们不希望的方式修改数据库，但是使用 publication 的主要原因是过滤返回的数据，这样恶意用户就不能看到我们不希望用户看到的数据。

### 你不能在渲染层确保安全性
像 Ruby on Rails 这种在服务器层渲染的架构，有足够的原因不要再返回的 HTML 响应中展示敏感数据。但在 Meteor 中，因为渲染是在客户端发生的，一个 if 语句在 HTML 模板中是不安全；需要在数据层做好安全性检测以确保数据不会在第一时间被发送。

### 有关 Method 的规则仍然适用
上面讲 Method 时提到的点对 publication 也适用：

1. 使用 check 和 aldeed:simple-schema 验证所有参数
2. 不要将当前用户 ID 作为参数传送
3. 不要使用通用参数；你应该清楚知道 publication 从客户端获取什么数据。
4. 使用速率限制阻止收到大量垃圾订阅。

### 总是严格限制域
Mongo.Collection#find 有一个选项叫域，可以让你在获取的文件中过滤这些域。你应该总是在 publication 使用这个功能已确保不会意外发布私密域。

例如，你可以写一个 publication，稍后再往 published 数据集添加一个私有域。现在，publication 会发生私密数据到客户端。如果在写 publication 的时候就过滤这些域，那么稍后添加一个域，改域是不会自动发布的。
```
// #1: 错误！如果我们稍后添加一个私有域，客户端是可以看到的。
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}});
});

// #2: 正确，如果我们稍后往列表添加私有域，客户端将其发布到客户端。
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}}, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});
```
如果你发现经常重复某些域，那就把公共域字典分离出来，这样在过滤的时候就可以多次使用它：
```
// 在定义列表的文件里
Lists.publicFields = {
  name: 1,
  incompleteCount: 1,
  userId: 1
};
这样代码就变得更简单了：
Meteor.publish('lists.public', function () {
  return Lists.find({userId: {$exists: false}}, {
    fields: Lists.publicFields
  });
});
```
### Publications 和 userId
从 publications 返回的数据经常依赖于当前登录用户，或者跟用户属性有关——该用户是管理员，或者该用户拥有某粉特殊文件等。

Publications 是非响应式的，只有在当前 userId 改变的时候才会重新运行，当前用户 ID 可以通过 this.userId 获取。因为这个原因，写的 publication 很可能在第一次运行的时候是安全的，但是不会响应应用环境的改变。我们来看一个例子：
```
// #1: 错误！如果列表的拥有者改变了，原先的拥有者还是会看到它。
Meteor.publish('list', function (listId) {
  check(listId, String);

  const list = Lists.findOne(listId);

  if (list.userId !== this.userId) {
    throw new Meteor.Error('list.unauthorized',
      'This list doesn\'t belong to you.');
  }

  return Lists.find(listId, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});

// #2: 正确！如果列表的拥有者改变了，原先的拥有者不会再看到它
Meteor.publish('list', function (listId) {
  check(listId, String);

  return Lists.find({
    _id: listId,
    userId: this.userId
  }, {
    fields: {
      name: 1,
      incompleteCount: 1,
      userId: 1
    }
  });
});
```
在第一个例子中，如果所选择的列表其 userId 属性改变，publication 查询依然会返回原来的数据，因为在代码开始的安全性检查不会重新运行。在第二个例子中，我们通过把安全性检测放在返回查询中来解决这个问题。

不幸的是，不是所有的 publication 的安全性检测都向上面的例子一样简单。关于如何使用 reywood:publish-composite 在 publication 中处理响应式改变，请查看数据加载文章。

### 传递选项
对于特定的 publication，例如分页，会需要传递选项给 publication 用于控制发送到客户端的文件数量。关于这个有几点需要额外注意：

1. 传递一个限制: 在需要从客户端传递 limit 选项查询查询数据的情况下，确保设置最大限制。否则的话，恶意用户可以一次性请求超多文件，进而引发性能问题。
2. 传递一个过滤器: 如果因为不需要所有的数据，将一个域传递给过滤器，例如搜索查询，请确保使用 MongoDB $and 使来自客户端的域和客户允许看到的文件进行交互——如何客户端可以过滤私密数据，可以运行搜索，找出该数据是什么。
3. 传递一个域: 如果你希望客户端有能力决定获取数据集中的哪个域，那么请确保这些域跟用户可以看到的域交互，以避免用户看到不该看到的数据。

总的来说，你应该确保从客户端传递给 publication 的数据只能限制在我们要求的范围内，而不是范围之外的数据。

### 服务器文件
虽然 Publications 是客户端从服务器获取数据的主要方式，但不是唯一的方式。服务器托管你的应用的一系列源码和静态资产也可能包含敏感数据：

1. 黑客可以通过分析应用的商业逻辑找出应用的薄弱点。
2. 竞争对手可以偷取的私密算法。
3. API 接口密钥

### 私密服务器代码
浏览器一定可以获取应用在客户端的代码，但每个应用或多或少都会在服务器存有私密代码，这些代码是不能给用户看到的。

应用中关于商业逻辑的私密代码应该被存储在服务器。这意味着要把这些代码放在应用中的 server/ 目录，或者放在一个只位于服务器的包，或放在一个只位于服务器的包里面的文件夹。

如果你的 Meteor 应用中有一个 Method 包含私密的商业逻辑，那最好将该 Method 一分为二——一部分是在客户端运行的优化 UI 组件，另外一部分是在服务器运行的私密代码。大部分情况下，把整个 Method 放在服务器运行都不会得到最好的用户体验。我们来看一个例子，在这个例子中有一个计算玩家在游戏中排名(MMR)的私密逻辑：
```
// 放在一个只位于服务器的文件夹
MMR = {
  updateWithSecretAlgorithm(userId) {
    // 私密代码放在这里
  }
}
// 在一个客户端和服务器端共享的文件夹
const Meteor.users.methods.updateMMR = new ValidatedMethod({
  name: 'Meteor.users.methods.updateMMR',
  validate: null,
  run() {
    if (this.isSimulation) {
      // Simulation code for the client (optional)
    } else {
      MMR.updateWithSecretAlgorithm(this.userId);
    }
  }
});
```
注意到虽然我们在客户端定义 Method，但是私密代码是在服务器上运行的。请注意放在 if (Meteor.isServer) 下面的代码还是会发生到客户端，只是不会在客户端执行。所有不要将任何私密代码放在 if (Meteor.isServer)。

私密的 API 接口密钥不应该存储在代码中，下面我们将讲解应该如何处理。

## API 接口密钥安全性
每个应用或多或少都有一些私密的 API 接口密钥或密码：

1. 数据库的密码。
2. 外部 API 接口密钥。

这些密钥或密码在版本控制中不应该跟应用的源码存储在一起，因为开发者可能会无意识中复制包含密钥的源代码到其他地方。可以将密钥分离开来并存储在Dropbox，或LastPass，或其他服务，然后在部署应用的时候再链接到这些密钥。

可以通过一个设置文件或环境变量文件传递设置给你的应用，应用的大部分设置都应该被存储在 JSON 文件中，然后在应用启动的时候传递给应用。在应用启动的时候通过 --settings 就可以传递相关设置信息：

### 在本地运行应用的时候传递 development 设置
```
meteor --settings development.json
```
### 在 Galaxy 部署应用的时候传递 production 设置
```
meteor deploy myapp.com --settings production.json
```
下面是一个带有密钥的设置文件：
```
{
  "facebook": {
    "clientId": "12345",
    "secret": "1234567"
  }
}
```
在你应用的 JavaScript 代码中，这些设置可以通过变量 Meteor.settings 获得。了解更多管理钥匙和设置的请查看部署文章

### 客户端设置
在大部分情况下，设置文件中的 API 接口密钥只会在服务器中使用，而且默认情况下通过 --settings 传递的数据只能在服务器使用。但是，如果把数据放在 public 文件夹下，在客户端就可以看到。当需要用户从客户端调用 API 并且用户可以知道这些密钥的时候，就可以这样做。公共设置通过 Meteor.settings.public 可以在客户端获得。

### OAuth 的 API 接口密钥
accounts-facebook 包需要这些 API 接口密钥，所有需要将其添加到数据库中的服务配置数据集。可以这样做：

首先，添加 service-configuration 包：
```
meteor add service-configuration
```
然后，添加到 ServiceConfiguration 数据集：
```
ServiceConfiguration.configurations.upsert({
  service: "facebook"
}, {
  $set: {
    clientId: Meteor.settings.facebook.clientId,
    loginStyle: "popup",
    secret: Meteor.settings.facebook.secret
  }
});
```
现在， accounts-facebook 包可以找到 API 密钥，可以使用 Facebook 登录系统。

## SSL
这里讲 SSL 的文字不多，但是也有必要讲一下。

每个生产中的 Meteor 应用，如果处理用户数据的话，都是跟 SSL 一起运行的。

对于外行来说，这意味着所有的 HTTP 请求都应该通过 HTTPS，所有的 websocket 数据都应该通过 WSS 发送。

Meteor 确实会在发送你的密码或者登录密钥之前在客户端哈希化，但这只可以阻止黑客识别密码 —— 不能阻止黑客以你的身份登陆，因为他们可以发送哈希密码到服务器实现登录！不管你如何分离代码，登录意味着需要客户端发送敏感数据到服务器，保证这个发送安全的唯一方式就是使用 SSL。注意到当在一个普通的 HTTP 应用使用 cookies 认证的时候也会出现这种问题，所有任何需要可靠地识别用户的应用都应该在 SSL 上运行。

添加 force-ssl 包后可以确保任何不安全的应用链接都会被重定向到安全链接。

### 设置 SSL
1. Galaxy 大部分都设置好了，但需要添加一个认证。请查看 SSL 和 Galaxy 帮助文档。
2. 如果你在自己搭的框架上运行，有几种选择可以设置 SSL, 大部分是通过配置代理 web 服务器。参考文章Josh Owens on SSL and Meteor, SSL on Meteorpedia, 以及 Digital Ocean tutorial with an Nginx config。

## 安全清单
下面的安全清单列表可以捕获一些常见的错误。但是，这不是一份详尽的清单 —— 如果你有补充的请提交一个 PR!

1. 确保应用中删除了 insecure 包和 autopublish 包。
2. 验证所有的 Method 和 publication 参数，并通过 audit-argument-checks 自动检测。
3. Deny writes to the profile field on user documents.拒绝写入用户文件的 profile 域
4. 使用 Methods 而不使用客户端的 insert/update/remove 以及 allow/deny.
5. 在 publication 中使用特殊的选择器和过滤器
6. 不要使用raw HTML inclusion in Blaze，除非你知道怎么用。
7. 确保 API 接口密钥和密码不会在源码中出现
8. Secure the data, not the UI 确保数据安全，而不是 UI安全 —— 在客户端重定向路径并不能保证安全性，只是优化了用户体验。
9. 不要信任从客户端传递的用户 ID，在 Method 和 publication 中使用 this.userId。
10. 设置浏览器策略，但并不是所有的浏览器都支持，只是给使用现代浏览器的用户多加一层保护层。
