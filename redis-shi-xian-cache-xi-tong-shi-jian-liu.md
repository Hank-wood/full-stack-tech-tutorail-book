#### 1. 介绍

rails中就自带有cache功能，不过它默认是用文件来存储数据的。我们要改为使用redis来存储。而且我们也需要把sessions也存放到redis中。关于rails实现cache功能的源码可见于这几处:

* https://github.com/rails/rails/blob/master/activesupport/lib/active_support/cache.rb
* https://github.com/rails/rails/tree/master/activesupport/lib/active_support/cache
* https://github.com/rails/rails/blob/master/actionview/lib/action_view/helpers/cache_helper.rb

#### 2. 使用

我们一步步在rails中使用cache实现我们的需求。

##### 2.1 开启cache模式

首先第一步我们要来开启cache模式。默认情况下，production环境是开启的，但是development没有，所以要开启它。

进入`config/environments/development.rb`文件中，把`config.action_controller.perform_caching`设为true。

``` ruby
config.action_controller.perform_caching = true
```

修改完，记得重启服务器。

##### 2.2 使用html片断cache

为了方便测试和了解整个原理，我们先不使用redis来存放cache数据，只使用默认的文件来存放数据。

以本站为例，我们要把首页的"最近的文章"那部分加上html片断的cache。

使用html片断cache，rails提供了一个helper方法可以办到，很简单，只需要把需要的html用cache包起来。

``` ruby
.row
  - cache do
    .col-md-6
      .panel.panel-default
        .panel-heading
          div 最近的文章
        .panel-body
          - @articles.each do |article|
            p.clearfix
              span.pull-right = datetime article.created_at
              = link_to article.title, article_path(article)
```

我们先在页面刷新一下，然后通过日志来观察。

先发现访问起来比平时慢一点点，因为它在把cache存到文件中，具体的log是下面这样的。

```
Started GET "/" for 127.0.0.1 at 2015-10-30 16:19:27 +0800
Processing by HomeController#index as HTML

  Cache digest for app/views/home/index.html.slim: 8e89c7a7d1da1d9719fca4639859b19d

Read fragment views/localhost:4000//8e89c7a7d1da1d9719fca4639859b19d (0.3ms)

  Article Load (2.0ms)  SELECT  "articles"."title", "articles"."created_at", "articles"."published", "articles"."group_id", "articles"."slug", "articles"."id" FROM "articles" WHERE "articles"."published" = $1  ORDER BY id DESC LIMIT 10  [["published", "t"]]
  Group Load (3.5ms)  SELECT "groups".* FROM "groups" WHERE "groups"."id" IN (1, 5, 3, 4)
  Article Load (0.9ms)  SELECT  "articles"."title", "articles"."created_at", "articles"."published", "articles"."group_id", "articles"."slug", "articles"."id", "articles"."visit_count" FROM "articles" WHERE "articles"."published" = $1  ORDER BY visit_count DESC LIMIT 10  [["published", "t"]]
  Group Load (2.3ms)  SELECT "groups".* FROM "groups" WHERE "groups"."id" IN (1, 3, 4)
  Group Load (4.4ms)  SELECT "groups".* FROM "groups"

Write fragment views/localhost:4000//8e89c7a7d1da1d9719fca4639859b19d (41.7ms)
...
```

主要就是`Cache digest`、`Read fragment`和`Write fragment`部分。

`Cache digest`是产生一个md5码，这个码来标识html的片断，会很有用，我们等下再来细说。

`Read fragment`是读取html片断(以文件形式存储)，根据之前产生的md5标识，发现不存在，就会生成一个html片断并存起来，就是`Write fragment `。

默认情况下，产生的html片断文件是存在/tmp/cache目录里的，如下：

```
~/codes/rails365 (master) $ tree tmp/cache/
tmp/cache/
├── 20B
│   └── 6F1
│       └── views%2Flocalhost%3A4000%2F%2F8e89c7a7d1da1d9719fca4639859b19d
```

打开`views%2Flocalhost%3A4000%2F%2F8e89c7a7d1da1d9719fca4639859b19d`这个文件，就会发现里面存储的就是html的片断。

现在我们在刷新一遍页面，再来看看日志。

```
Started GET "/" for 127.0.0.1 at 2015-10-30 16:53:18 +0800
Processing by HomeController#index as HTML
  Cache digest for app/views/home/index.html.slim: 8e89c7a7d1da1d9719fca4639859b19d
Read fragment views/localhost:4000//8e89c7a7d1da1d9719fca4639859b19d (0.3ms)
...
```

就会发现`Write fragment`没有了，也不查询数据库了，数据都从html片断cache取了。

这样还不算完成。我们要考虑一个问题，就是我们改了数据或html的内容的时候，cache会自动更新吗？

##### 2.3 Cache digest

先来说更改html片断代码本身的情况。

我们把"最近的文章"改成”最新的文章"，然后我们来观察是否会生效。

最终通过查看日志，发现还是产生了`Write fragment`，说明是生效的。

这个原理是什么呢？

我们找到cache这个helper方法的[源码](https://github.com/rails/rails/blob/master/actionview/lib/action_view/helpers/cache_helper.rb)。

``` ruby
def cache(name = {}, options = {}, &block)
  if controller.respond_to?(:perform_caching) && controller.perform_caching
    safe_concat(fragment_for(cache_fragment_name(name, options), options, &block))
  else
    yield
  end

  nil
end
```

发现关键在`cache_fragment_name`这个方法里，顺应地找到下面两个方法。

``` ruby
def cache_fragment_name(name = {}, skip_digest: nil, virtual_path: nil)
  if skip_digest
    name
  else
    fragment_name_with_digest(name, virtual_path)
  end
end

def fragment_name_with_digest(name, virtual_path) #:nodoc:
  virtual_path ||= @virtual_path
  if virtual_path
    name  = controller.url_for(name).split("://").last if name.is_a?(Hash)
    digest = Digestor.digest name: virtual_path, finder: lookup_context, dependencies: view_cache_dependencies
    [ name, digest ]
  else
    name
  end
end
```

关键就在`fragment_name_with_digest`这个方法里，它会找到cache的第一个参数，然后用这个参数的内容生成md5码，我们刚才改变了html的内容，也就是参数改变了，md5自然就变了，md5一变就得重新生成html片断。

所以cache方法的第一个参数是关键，它的内容是判断重不重新产生html片断的依据。

改变html片断代码之后，是会重新生成html片断的，但如果是在articles中增加一条记录呢？通过尝试发现不会重新生成html片断的。

那我把@artilces作为第一个参数传给cache方法。

``` ruby
.row
  - cache @articles do
    .col-md-6
      .panel.panel-default
        .panel-heading
          div 最近的文章
        .panel-body
          - @articles.each do |article|
            p.clearfix
              span.pull-right = datetime article.created_at
              = link_to article.title, article_path(article)
```

发现生成了`Write fragment`，说明是可以的，页面也会生效。

```
Cache digest for app/views/home/index.html.slim: 1c628fa3d96abde48627f8a6ef319c1c

Read fragment views/articles/15-20151027051837664089000/articles/14-20151030092311065810000/articles/13-20150929153356076334000/articles/12-20150929144255631082000/articles/11-20151027064325237540000/articles/10-20150929153421707840000/articles/9-20150929123736371074000/articles/8-20150929144346413579000/articles/7-20150929144324012954000/articles/6-20150929144359736164000/1c628fa3d96abde48627f8a6ef319c1c (0.1ms)

Write fragment views/articles/15-20151027051837664089000/articles/14-20151030092311065810000/articles/13-20150929153356076334000/articles/12-20150929144255631082000/articles/11-20151027064325237540000/articles/10-20150929153421707840000/articles/9-20150929123736371074000/articles/8-20150929144346413579000/articles/7-20150929144324012954000/articles/6-20150929144359736164000/1c628fa3d96abde48627f8a6ef319c1c (75.9ms)

Article Load (2.6ms)  SELECT  "articles"."title", "articles"."created_at", "articles"."updated_at", "articles"."published", "articles"."group_id", "articles"."slug", "articles"."id", "articles"."visit_count" FROM "articles" WHERE "articles"."published" = $1  ORDER BY visit_count DESC LIMIT 10  [["published", "t"]]
```

但是，除此之外，还有sql语句生成，而且以后的每次请求都有，即使命中了cache，因为@articles作为第一个参数，它的内容是要通过数据库来查找的。

那有一个解决方案是这样的：把@articles的内容也放到cache中，这样就不用每次都查找数据库了，而一旦有update或create数据的时候，就让@articles过期或者重新生成。

为了方便测试，我们先把cache的存储方式改为用redis来存储数据。

添加下面两行到Gemfile文件，执行`bundle`。

``` ruby
gem 'redis-namespace'
gem 'redis-rails'
```

在`config/application.rb`中添加下面这一行。

``` ruby
config.cache_store = :redis_store, {host: '127.0.0.1', port: 6379, namespace: "rails365"}
```

`@articles`的内容要改为从redis获得，主要是读redis中健为`articles`的值。


``` ruby
class HomeController < ApplicationController
  def index
    @articles = Rails.cache.fetch "articles" do
      Article.except_body_with_default.order("id DESC").limit(10).to_a
    end
  end
end
```

创建或生成一条article记录，都要让redis的数据无效。

``` ruby
class Admin::ArticlesController < Admin::BaseController
  ...
  def create
    @article = Article.new(article_params)
    Rails.cache.delete "articles"
    ...
  end
end
```

这样再刷新两次以上，就会发现不再查数据库了，除非添加或修改了文章(article)。

完结
