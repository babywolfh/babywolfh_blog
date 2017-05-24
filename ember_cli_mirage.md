---
title: Ember_Cli_Mirage使用说明
tags: [ember,mirage] 
categories: JavaScript
toc: true
---
# Reference:
[官网](http://www.ember-cli-mirage.com/)
[使用例子](http://www.programwitherik.com/ember-mirage-tutorial-and-examples/)
# 安装
```shell
$ ember install ember-cli-mirage
```
# 配置(可以不需要这一步)
By default, Mirage is disabled in production, and also in development when using the -proxy option.
To disable Mirage explicitly, you can set the enabled config option to false. For example, to always disable in development:
``` javascript
// config/environment.js
...
if (environment === 'development') {
  ENV['ember-cli-mirage'] = {
    enabled: false
  };
}
```
# 使用
## 定义Handler
Mirage会模拟你的API服务器，你需要定义route handler来回应的Ember App的AJAX请求。
可以使用get,post,put,patch,del方法。使用语法如下：
```
this.method(url_path, handler)
# handler函数返回实际的response.
```
比如下面的代码：
``` javascript
// mirage/config.js
export default function() {
  this.namespace = 'api';
  //回应GET /api/authors
  this.get('/authors', () => {
    // 返回的数据
    return {
      libraries: [
        {id: 1, name, 'Zelda'},
        {id: 2, name, 'Link},
        {id: 3, name, 'Epona},
      ]
    };
  });
}
```

## 动态数据
### 创建Model
在Mirage的内存数据中建立一个library的表。
```
$ ember g mirage-model author
# 将生成mirage/models/author.js
```
### 修改Handler
``` javascript
this.get('/authors', (schema, request) => {
  return schema.authors.all();
});

```
schema作为第一个参数插入到每个route handler中。schema用来访问models.
在一个session中，你所有的route handler对数据库的操作会保存，使得app就像和真的server在交互一样。
### 创建数据
#### 创建Factory
使用Factories来给数据库填充假数据。Mirage包含Fake.js库。
``` shell
$ ember g mirage-factory author
# 生成mirage/factories/author.js
```
修改代码
``` javascript
// mirage/factories/author.js
import { Factory, faker } from 'ember-cli-mirage';

export default Factory.extend({
  //参数是被建对象的序列号
  name(i) {return `Person ${i}`;},
  age: 28,
  admin: false,
  avatar() {return faker.internet.avatar();}
});
```
也可以使用Faker.js库来产生看起来更真实的数据：
``` js
// mirage/factories/author.js
import { Factory, faker } from 'ember-cli-mirage';

export default Factory.extend({
  firstName() {
    return faker.name.firstName();
  },
  lastName() {
    return faker.name.lastName();
  },
  age() {
    // list method added by Mirage
    return faker.list.random(18, 20, 28, 32, 45, 60)();
  },
});
```
使用这个factory创建类似下面的数据，并自动插入到author这个数据库表中：
``` js
[{
  name: 'Person 1',
  age: 28,
  admin: false,
  avatar: 'https://s3.amazonaws.com/uifaces/faces/twitter/bergmartin/128.jpg'
}]
```
#### 使用Factory
为了使用Factory，我们需要使用如下代码
``` js
// mirage/scenarios/default.js
export default function(server) {
  server.createList('author', 10);
};
```

通过上面的步骤后，就可以在Ember App中直接使用Ember-Data的数据访问接口了。

# 其他特性
## 动态路径和查询参数
插入到route handler中的request对象包含动态路径部分和查询参数。
为了定义route能处理动态部分，需要在path中使用冒号语法(:segment)，动态部分可以通过request.params.[segment]获取到，查询参数可以通过request.queryParams.[param]获取到
``` js
this.get('/authors/:id', (schema, request) => {
  var id = request.params.id;

  return schema.authors.find(id);
})
```
## 关联与序列化
Mirage自带一个ORM来帮助建立数据之间的relationships
``` js
// 一个author有多个blog-posts
// mirage/models/author.js
import { Model, hasMany } from 'ember-cli-mirage';

export default Model.extend({
  blogPosts: hasMany()
});

// mirage/models/blog-post.js
import { Model, belongsTo } from 'ember-cli-mirage';

export default Model.extend({
  author: belongsTo()
});

// 编写route
// mirage/config.js
this.get('/authors/:id/blog-posts', (schema, request) => {
  let author = schema.authors.find(request.params.id);

  return author.blogPosts;
});

// 创建数据
// mirage/scenarios/default.js
let author = server.create('author');
server.createList('post', 10, { author });

```
Mirage的序列化器也关注relationships
``` js
// mirage/serializers/application.js
import { Serializer } from 'ember-cli-mirage';

export default Serializer.extend({
  keyForAttribute(attr) {
    return dasherize(attr);
  },

  keyForRelationship(attr) {
    return dasherize(attr);
  },
});

// mirage/serializers/author.js
import { Serializer } from 'ember-cli-mirage';

export default Serializer.extend({
  include: ['blogPosts']
});

// mirage/config.js
export default function() {
  this.get('/authors/:id', (schema, request) => {
    return schema.authors.find(request.params.id);
  });
}
```
## shoutcuts
减少代码书写量：
``` js
this.get('/authors', (schema, request) => {
  return schema.authors.all();
});
```
可缩写成
``` js
this.get('/authors');
````
还有的快捷方式有：
``` js
this.get('/authors');
this.get('/authors/:id');
this.post('/authors');
this.put('/authors/:id'); // or this.patch
this.del('/authors/:id');
```

## passthrough
``` js
// mirage/config.js
this.get('/comments'); //这部分莫捏

this.passthrough();  //其他部分不模拟，直接透传出去到真实的服务器
```

# 一个真实例子
## 准备工作
``` shell
$ ember  new mirageTest
$ cd mirageTest
$ ember install ember-cli-mirage
$ ember g resource posts 
$ ember g model post title:string text:string
$ ember g mirage-model post
$ ember g mirage-factory post
```
## Ember App 路由 
``` js
// app/router.js
import Ember from 'ember';
import config from './config/environment';

const Router = Ember.Router.extend({
  location: config.locationType,
  rootURL: config.rootURL
});

Router.map(function() {
  this.route('post', {path: '/post/:post_id'});
});
export default Router;

// app/routes/application.js
import Ember from 'ember';

export default Ember.Route.extend({  
    model() {
        return this.store.findAll('post');
    }
});

// app/routes/post.js
import Ember from 'ember';

export default Ember.Route.extend({  
    model(param){
        return this.store.findRecord('post',param.post_id);
    }
});

```
## Ember App 模板
``` js
// app/templates/application.hbs
<h2 id="title">Welcome to Ember</h2>

{{#each model as |post|}}
    {{#link-to 'post' post.id}}{{post.title}}{{/link-to}}<br>
{{/each}}
<br>  
{{outlet}}

// app/templates/posts.hbs
Title:{{model.title}}<br>  
Text:{{model.text}}<br>  
```

## Mirage
``` js
// mirage/config.js 
export default function() {
        this.get('/posts');
        this.get('/posts/:id');
}

// mirage/factories/post.js
import { Factory, faker } from 'ember-cli-mirage';

export default Factory.extend({
  title() { return faker.random.word(); },
  text() { return faker.random.words(); },
});

// mirage/scenarios/default.js
export default function(server) {
  server.createList('post', 5);
}

```

## 运行
``` shell
$ ember server
```
打开浏览器访问http://localhost:4200/
