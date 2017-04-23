---
layout: post
title:  "Ruby Sinatra Extension"
date:   2017-03-27 10:30
categories: develop
tags: ruby
excerpt: Sinatra是Ruby的轻量级Web框架，它为extension作者提供了API，以帮助确保为Ruby应用程序开发人员提供一致的行为。
---


### 背景
Sinatra内部的设计学问是需要写很好的extension扩展的。本节提供了Sinatra设计核心类和习语的high-level的概述。Sinatra有两种明确的方式来使用extension扩展:


1. 经典风格，Ruby App定义在顶层。经典的Ruby App一般是单文件的，独立的应用直接通过命令行或rackup文件运行。如果一个经典Ruby App需要extension扩展，那么Ruby App开发者会希望所有扩展功能应该已经存在，而不需要额外的设置（例如include／extending modules）。
2. 模块化风格，其中Sinatra::Base被明确子类化，应用程序在子类的范围内定义。这些应用经常作为libraries来绑定，并在一个庞大的基于Rack的系统中作为组件使用。模块化应用必须在类中明确调用**register ExtensionModule**。
所有extension扩展都不仅与App风格有关，而且需要extension作者在每种风格下都做正确的事情。extension的API（**Sinatra.register**和**Sinatra.helpers**）就是用来帮助extension作者完成这个事情。

> 重要：下面关于Sinatra::Base和Sinatra::Application的文章仅仅是介绍一些背景知识，extension作用不需要直接修改这些类


### Sinatra::Base
Sinatra::Base类为Sinatra应用提供上下文。顶层DSL存在于类范围中，而请求级别的东西存在于实例中。App定义在Sinatra::Base的子类，DSL(e.g., get, post, before, configure, set, etc.)是一组定义在Sinatra::Base的方法,扩展DSL一般通过在Sinatra::Base或它的子类中增加类方法来达成。然而，Base类不能通过extend来扩展。Sinatra.register方法就是为了实现这个功能的。


请求在新的Sinatra :: Base实例中进行评估-routes, before filters, views, helpers, 以及error pages一起共享这个上下文。请求级别的帮助方法包括（erb, haml, halt, content_type，等）都是定在在Sinatra::Base中的简单实例方法，或include到Sinatra::Base的modules。在request级别提供新的功能需要在Sinatra::Base中增加实例方法来实现。


与DSL扩展一样，extension开发者不应该直接将helper模块include到Sinatra::Base中。Sinatra.helpers方法就是用来解决这个事情的。


### Sinatra::Application
Sinatra::Application为经典风格的Ruby应用提供默认的context。它是Sinatra::Base的子类，提供默认的option值和其他顶层App行为。当一个经典Ruby应用运行时，所有Sinatra::Application的public类方法都会可以在顶层App之间共享。


### Extensions规则
1. 不要直接修改Sinatra::Base。不应该include或extend，改变选项值或修改行为。模块化风格的应用通过使用register方法扩展他们的子类。
2. 不要在extension扩展中**require 'sinatra'**。你应该只需要**require 'sinatra/base'**原因是**require 'sinatra'**是经典风格的做法--extensions不应该出发经典风格。
3. 尽可能使用后面提到的API。不应该在Sinatra的核心类中直接include或extend modules或直接定义方法。后面讲到一些专门的方法用于处理这些情况。
4. Extension扩展需要定义在各个分开的Sinatra模块下。例如，增加基础验证可以命名为Sinatra::BaseAuth。

### 通过Sinatra.helpers扩展请求上下文
Extension最普通的类型是在routes，view和helper方法中增加方法。


例如，假设你想写一个extension扩展，方法名为h，用于避开HTML中的特征字符（和其他有名的Ruby Web框架一样）。


```
require 'sinatra/base'

module Sinatra
  module HTMLEscapeHelper
    def h(text)
      Rack::Utils.escape_html(text)
    end
  end

  helpers HTMLEscapeHelper
end
```

对Sinatra.helpers的调用包括在Sinatra::Application中的模块,让所有方法定义在模块中也对经典风格的App可用。在经典风格的extension中使用新的方法就和require extension一样简单:


```
require 'sinatra'
require 'sinatra/htmlescape'

get "/hello" do
  h "1 < 2"     # => "1 &lt; 2"
end
```

另外，对于Sinatra::Base的子类，必须require和include指定的模块来使用helpers方法：


```
require 'sinatra/base'
require 'sinatra/htmlescape'

class HelloApp < Sinatra::Base
  helpers Sinatra::HTMLEscapeHelper

  get "/hello" do
    h "1 < 2"
  end
end
```


### 通过Sinatra.register扩展SDL类上下文
Extension扩展也可以通过Sinatra.register方法扩展Sinatra类级别的DSL。以下是一个extension扩展，它增加block_links_from宏，用于检查每个App接受的请求，并在匹配时返回**403 Forbidden** response.

```
require 'sinatra/base'

module Sinatra
  module LinkBlocker
    def block_links_from(host)
      before {
        halt 403, "Go Away!" if request.referer.match(host)
      }
    end
  end

  register LinkBlocker
end
```

Sinatra.register增加所有public方法到module中，并作为Sinatra::Application类方法。当经典风格的app执行时，它也可以将public方法添加到顶层。


一个经典风格的App可以这样使用这个extension：

```
require 'sinatra'
require 'sinatra/linkblocker'

block_links_from 'digg.com'

get '/' do
  "Hello World"
end
```

模块化风格的App必须明确注册extension到他们的Sinatra::Base子类：

```
require 'sinatra/base'
require 'sinatra/linkblocker'

class Hello < Sinatra::Base
  register Sinatra::LinkBlocker

  block_links_from 'digg.com'

  get '/' do
     "Hello World"
  end
end
```

### 设置选项和其他extension配置
Extension可以通过registered方法定义options，routes，before filters，和错误处理。在extension模块添加到Sinatra::Base子类和通过模块注册的类后,Module.registered方法会被立即调用。下面的例子就是创建一个简单的extension，用于支持基础session认证。增加了username和password两个选项，routes被定义用作登陆，helper方法用于决定用户是否已经被认证过了：

```
require 'sinatra/base'

module Sinatra
  module SessionAuth

    module Helpers
      def authorized?
        session[:authorized]
      end

      def authorize!
        redirect '/login' unless authorized?
      end

      def logout!
        session[:authorized] = false
      end
    end

    def self.registered(app)
      app.helpers SessionAuth::Helpers

      app.set :username, 'frank'
      app.set :password, 'changeme'

      app.get '/login' do
        "<form method='POST' action='/login'>" +
        "<input type='text' name='user'>" +
        "<input type='text' name='pass'>" +
        "</form>"
      end

      app.post '/login' do
        if params[:user] == options.username && params[:pass] == options.password
          session[:authorized] = true
          redirect '/'
        else
          session[:authorized] = false
          redirect '/login'
        end
      end
    end
  end

  register SessionAuth
end
```

经典App需要通过require extension library，重写选项，并使用提供的helpers方法j

```
require 'sinatra'
require 'sinatra/sessionauth'

set :password, 'hoboken'

get '/public' do
  if authorized?
    "Hi. I know you."
  else
    "Hi. We haven't met. <a href='/login'>Login, please.</a>"
  end
end

get '/private' do
  authorize!
  'Thanks for logging in.'
end
```

模块化的App唯一的不同就是必须在模块中register这个extension扩展。

```
require 'sinatra/base'
require 'sinatra/sessionauth'

class MyApp < Sinatra::Base
  register Sinatra::SessionAuth

  set :password, 'hoboken'

  get '/public' do
    if authorized?
      "Hi. I know you."
    else
      "Hi. We haven't met. <a href='/login'>Login, please.</a>"
    end
  end

  get '/private' do
    authorize!
    'Thanks for logging in.'
  end
end
```

### 构建和打包Extension扩展
Sinatra extension扩展应该作为分开的包构建，并打包成gems或单个文件，以方便include到App的lib文件夹。使用extension扩展的理想流程是安装gem并require一个单文件


下面是一个典型的extension扩展的文件目录结构，并打包成gem：

```
sinatra-fu
|-- README
|-- LICENSE
|-- Rakefile
|-- lib
|   `-- sinatra
|       `-- fu.rb
|-- test
|   `-- spec_sinatra_fu.rb
`-- sinatra-fu.gemspec
```
