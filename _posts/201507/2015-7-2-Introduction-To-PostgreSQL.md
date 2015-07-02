---
layout: post
title:  "Using Postgresql in Rails"
date:   2015-07-02 13:55:37
categories: programming
tags: programming
author: "ScorpiusZ"
---


## PostgreSQL 介绍 

###简介 
>
PostgreSQL 是一个自由的对象-关系数据库服务器(数据库管理系统)，它在灵活的 BSD-风格许可证下发行。它提供了相对其他开放源代码数据库系统(比如 MySQL 和 Firebird)，和专有系统(比如 Oracle、Sybase、IBM 的 DB2 和 Microsoft SQL Server)之外的另一种选择。
>

#### 优点 
>
事实上， PostgreSQL 的特性覆盖了 SQL-2/SQL-92 和 SQL-3/SQL-99，首先，它包括了可以说是目前世界上最丰富的数据类型的支持，其中有些数据类型可以说连商业数据库都不具备， 比如 IP 类型和几何类型等；其次，PostgreSQL 是全功能的自由软件数据库，很长时间以来，PostgreSQL 是唯一支持事务、子查询、多版本并行控制系统（MVCC）、数据完整性检查等特性的唯一的一种自由软件的数据库管理系统。 Inprise 的 InterBase 以及SAP等厂商将其原先专有软件开放为自由软件之后才打破了这个唯一。最后，PostgreSQL拥有一支非常活跃的开发队伍，而且在许多黑客的努力下，PostgreSQL 的质量日益提高。
>
从技术角度来讲，PostgreSQL 采用的是比较经典的C/S（client/server）结构，也就是一个客户端对应一个服务器端守护进程的模式，这个守护进程分析客户端来的查询请求，生成规划树，进行数据检索并最终把结果格式化输出后返回给客户端。为了便于客户端的程序的编写，由数据库服务器提供了统一的客户端 C 接口。而不同的客户端接口都是源自这个 C 接口，比如ODBC，JDBC，Python，Perl，Tcl，C/C++，ESQL等， 同时也要指出的是，PostgreSQL 对接口的支持也是非常丰富的，几乎支持所有类型的数据库客户端接口。这一点也可以说是 PostgreSQL 一大优点。
>

#### 缺点
>
从 Postgres 开始，PostgreSQL 就经受了多次变化。	
>
首先，早期的 PostgreSQL 继承了几乎所有 Ingres, Postgres, Postgres95 的问题：过于学院味，因为首先它的目的是数据库研究，因此不论在稳定性， 性能还是使用方方面面，长期以来一直没有得到重视，直到 PostgreSQL 项目开始以后，情况才越来越好，PostgreSQL 已经完全可以胜任任何中上规模范围内的应用范围的业务。目前有报道的生产数据库的大小已经有 TB 级的数据量，已经逼近 32 位计算的极限。不过学院味也给 PostgreSQL 带来一个意想不到的好处：大概因为各大学的软硬件环境差异太大的缘故，它是目前支持平台最多的数据库管理系统的一种，所支持的平台多达十几种，包括不同的系统，不同的硬件体系。至今，它仍然保持着支持平台最多的数据库管理系统的称号。
>
其次，PostgreSQL 的确还欠缺一些比较高端的数据库管理系统需要的特性，比如数据库集群，更优良的管理工具和更加自动化的系统优化功能 等提高数据库性能的机制等。
>

###数据类型

#### 1.[Byte](http://www.postgresql.org/docs/current/static/datatype-binary.html)
```ruby
create_table :documents do |t|
  t.binary 'payload'
end
 
# app/models/document.rb
class Document < ActiveRecord::Base
end
 
# Usage
data = File.read(Rails.root + "tmp/output.pdf")
Document.create payload: data
```

#### 2.[Array](http://www.postgresql.org/docs/current/static/arrays.html)
```ruby
create_table :books do |t|
  t.string 'title'
  t.string 'tags', array: true
  t.integer 'ratings', array: true
end
add_index :books, :tags, using: 'gin'
add_index :books, :ratings, using: 'gin'
 
# app/models/book.rb
class Book < ActiveRecord::Base
end
 
# Usage
Book.create title: "Brave New World",
            tags: ["fantasy", "fiction"],
            ratings: [4, 5]
 
## Books for a single tag
Book.where("'fantasy' = ANY (tags)")
 
## Books for multiple tags
Book.where("tags @> ARRAY[?]::varchar[]", ["fantasy", "fiction"])
 
## Books with 3 or more ratings
Book.where("array_length(ratings, 1) >= 3")
```

#### 3.[Hstore](http://www.postgresql.org/docs/current/static/hstore.html)
```ruby
create_table :books do |t|
  t.string 'title'
  t.string 'tags', array: true
  t.integer 'ratings', array: true
end
add_index :books, :tags, using: 'gin'
add_index :books, :ratings, using: 'gin'
 
# app/models/book.rb
class Book < ActiveRecord::Base
end
 
# Usage
Book.create title: "Brave New World",
            tags: ["fantasy", "fiction"],
            ratings: [4, 5]
 
## Books for a single tag
Book.where("'fantasy' = ANY (tags)")
 
## Books for multiple tags
Book.where("tags @> ARRAY[?]::varchar[]", ["fantasy", "fiction"])
 
## Books with 3 or more ratings
Book.where("array_length(ratings, 1) >= 3")
```

#### 4.[JSON](http://www.postgresql.org/docs/current/static/datatype-json.html)
```ruby
create_table :events do |t|
  t.json 'payload'
end
 
# app/models/event.rb
class Event < ActiveRecord::Base
end
 
# Usage
Event.create(payload: { kind: "user_renamed", change: ["jack", "john"]})
 
event = Event.first
event.payload # => {"kind"=>"user_renamed", "change"=>["jack", "john"]}
 
## Query based on JSON document
# The -> operator returns the original JSON type (which might be an object), whereas ->> returns text
Event.where("payload->>'kind' = ?", "user_renamed")
```

#### 5.[Range Types](http://www.postgresql.org/docs/current/static/rangetypes.html)
```ruby
create_table :events do |t|
  t.daterange 'duration'
end
 
# app/models/event.rb
class Event < ActiveRecord::Base
end
 
# Usage
Event.create(duration: Date.new(2014, 2, 11)..Date.new(2014, 2, 12))
 
event = Event.first
event.duration # => Tue, 11 Feb 2014...Thu, 13 Feb 2014
 
## All Events on a given date
Event.where("duration @> ?::date", Date.new(2014, 2, 12))
 
## Working with range bounds
event = Event.
  select("lower(duration) AS starts_at").
  select("upper(duration) AS ends_at").first
 
event.starts_at # => Tue, 11 Feb 2014
event.ends_at # => Thu, 13 Feb 2014
```

#### 6.[Enumerated Types](http://www.postgresql.org/docs/current/static/datatype-enum.html)
#### 7.[UUID](http://www.postgresql.org/docs/current/static/datatype-uuid.html)
#### 8.[Bit String Types](http://www.postgresql.org/docs/current/static/datatype-bit.html)
#### 9.[Network Address Types](http://www.postgresql.org/docs/current/static/datatype-net-types.html)
```ruby
create_table(:devices, force: true) do |t|
  t.inet 'ip'
  t.cidr 'network'
  t.macaddr 'address'
end
 
# app/models/device.rb
class Device < ActiveRecord::Base
end
 
# Usage
macbook = Device.create(ip: "192.168.1.12",
                        network: "192.168.2.0/24",
                        address: "32:01:16:6d:05:ef")
 
macbook.ip
# => #<IPAddr: IPv4:192.168.1.12/255.255.255.255>
 
macbook.network
# => #<IPAddr: IPv4:192.168.2.0/255.255.255.0>
 
macbook.address
# => "32:01:16:6d:05:ef"
```

#### 10.[Geometric Types](http://www.postgresql.org/docs/current/static/datatype-geometric.html)

###列子
实现一个 Model Member 	

```ruby
class CreateMembers < ActiveRecord::Migration
  def change
    create_table :members do |t|
      t.string :nickname
      t.json :info
      t.json :detail
      t.json :relation_proposal
      t.string :interests, array:true
      t.string :hobbies, array:true
      t.string :tags, array:true
      t.timestamps null: false
    end
  end
end
```
给 Aarry Column add index

```ruby
class AddIndexOnInterestToMember < ActiveRecord::Migration
  def change
    add_index :members, :interests, using: 'gin'
  end
end
```
```ruby
2.1.5 :001 > Member.count
   (443.0ms)  SELECT COUNT(*) FROM "members"
 => 1030004 
```
查询兴趣

```ruby
2.1.5 :009 > Member.where(" interests @> ARRAY[?]::varchar[]",["宅", "有责任心"]).count
   (44.9ms)  SELECT COUNT(*) FROM "members" WHERE ( interests @> ARRAY['宅','有责任心']::varchar[])
 => 7012 
2.1.5 :010 > Member.where(" interests @> ARRAY[?]::varchar[]",["宅", "有责任心"]).limit(10).map(&:interests)
  Member Load (35.7ms)  SELECT  "members".* FROM "members" WHERE ( interests @> ARRAY['宅','有责任心']::varchar[]) LIMIT 10
 => [["有责任心", "宅", "孝顺"], ["冷静", "热情", "宅", "有责任心"], ["有责任心", "讲义气", "宅"], ["贤惠", "自我", "有责任心", "宅", "随和"], ["冷静", "稳重", "宅", "有责任心", "可爱"], ["宅", "孝顺", "纯真", "有责任心"], ["有责任心", "自我", "孝顺", "宅"], ["体贴", "孝顺", "宅", "有责任心", "低调"], ["温柔", "爱吃", "热情", "有责任心", "宅"], ["有责任心", "温柔", "坚强", "宅"]] 

```

查询性别及地区

```ruby
2.1.5 :018 > Member.where("info->> 'sex' = ?",'1').where("info->> 'location_id' = ?",'1').limit(10).map{|e| e.info}
  Member Load (10.3ms)  SELECT  "members".* FROM "members" WHERE (info->> 'sex' = '1') AND (info->> 'location_id' = '1') LIMIT 10
 => [{"nickname"=>"name43", "sex"=>1, "birthday"=>"1992-01-01", "location_id"=>1, "height"=>175, "weight"=>115}, {"nickname"=>"name16", "sex"=>1, "birthday"=>"1979-01-01", "location_id"=>1, "height"=>153, "weight"=>161}, {"nickname"=>"name6", "sex"=>1, "birthday"=>"1979-01-01", "location_id"=>1, "height"=>143, "weight"=>162}, {"nickname"=>"name8", "sex"=>1, "birthday"=>"1979-01-01", "location_id"=>1, "height"=>186, "weight"=>167}, {"nickname"=>"name88", "sex"=>1, "birthday"=>"1975-01-01", "location_id"=>1, "height"=>162, "weight"=>172}, {"nickname"=>"name82", "sex"=>1, "birthday"=>"1975-01-01", "location_id"=>1, "height"=>152, "weight"=>101}, {"nickname"=>"name20", "sex"=>1, "birthday"=>"1979-01-01", "location_id"=>1, "height"=>146, "weight"=>143}, {"nickname"=>"name3", "sex"=>1, "birthday"=>"1992-01-01", "location_id"=>1, "height"=>197, "weight"=>175}, {"nickname"=>"name51", "sex"=>1, "birthday"=>"1974-01-01", "location_id"=>1, "height"=>152, "weight"=>161}, {"nickname"=>"name31", "sex"=>1, "birthday"=>"1978-01-01", "location_id"=>1, "height"=>174, "weight"=>99}] 
 ```
 
 对比一下 Array 加索引和 不加索引的速度
 
 ```ruby
 2.1.5 :024 >   Member.where(" interests @> ARRAY[?]::varchar[]",["宅", "有责任心"]).count
   (24.2ms)  SELECT COUNT(*) FROM "members" WHERE ( interests @> ARRAY['宅','有责任心']::varchar[])
 => 7012 
2.1.5 :025 > Member.where(" hobbies @> ARRAY[?]::varchar[]",["上网", "听音乐"]).count
   (1078.5ms)  SELECT COUNT(*) FROM "members" WHERE ( hobbies @> ARRAY['上网','听音乐']::varchar[])
 => 11064 
 ```
 
 对比一下 JSON 加索引 与不加索引的 差距
 给 info 的 location_id 添加索引
 
 ```ruby
 CREATE INDEX ON members((info->>'location_id'));
 ```
 ```ruby
 2.1.5 :004 > Member.where("info->> 'location_id' = ?",'2').count
   (1221.2ms)  SELECT COUNT(*) FROM "members" WHERE (info->> 'location_id' = '2')
 => 30249 
2.1.5 :005 > Member.where("info->> 'sex' = ?",'1').count
   (4119.7ms)  SELECT COUNT(*) FROM "members" WHERE (info->> 'sex' = '1')
 => 515445 
2.1.5 :006 > 
```


###参考链接
[rails postgresql guide](http://edgeguides.rubyonrails.org/active_record_postgresql.html)	
[install postgresql on mac](http://www.moncefbelyamani.com/how-to-install-postgresql-on-a-mac-with-homebrew-and-lunchy/)
