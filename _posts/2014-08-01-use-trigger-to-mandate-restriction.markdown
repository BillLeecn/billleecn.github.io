---
layout: post
title: 利用数据库触发器约束数据完整性的例子
tags:
    - 数据库
    - database
    - trigger
    - MySQL
comments: true
---

据说，80% 的业务逻辑可以在数据库中完成。

在关系数据库中，事务和外键约束是用来保护一致性的常用方法。对于更复杂的业务逻辑，我们还可以通过触发器来提供保证。

首先描述一下这个例子中的场景：有一个表 order, 保存订单数据，字段为 id, customer_id, paid. 其中， paid 字段为 1 时表示订单已经付款。另一个表 order_item, 保存订单的内容，字段为 order_id, item_id, count. 

显然，一个已经付款的订单是不应该修改的，为了在数据库层面上强制这个约束，就需要用到触发器，在修改 order_item 表之前，查询 order 表中对应订单的 paid 字段。触发器的细节可能和具体的实现相关，这个例子中，使用的是 MySQL.

在 MySQL 中，每个表可以安装6个[触发器](http://dev.mysql.com/doc/refman/5.7/en/trigger-syntax.html)。分别为 before insert, before update, before delete, after insert, after update, after delete. 其中 after 系列在操作成功完成后执行，before 系列在操作执行之前执行。当 before 触发器执行出错时，对表的修改不会被执行。通过在 before 触发器中引发错误，就可以阻止对表的修改。

而在 MySQL 中，有一个 [SIGNAL](http://dev.mysql.com/doc/refman/5.7/en/signal.html) 语句专门用于抛出错误或警告。在这里，可以使用代号为 '45000' 的一般错误，并制定错误消息。

这里，以 before insert 为例，说明如何用触发器组织 insert 操作。SQL 代码逻辑很清晰，就不多做解释了。

{% highlight mysql %}
delimiter //
CREATE TRIGGER trigger_before_insert_order_item BEFORE INSERT on order_item
FOR EACH ROW
BEGIN
    IF EXSIT (SELECT id FROM order WHERE id = NEW.order_id AND paid = 1) THEN
        SIGNAL '45000' SET MESSAGE_TEXT = 'A paid order cannot be modified!';
    END IF;
END;//
delimiter ;
{% endhighlight %}

按照类似的方式安装 before update, before delete 触发器，就可以在数据库层面保证已付款的订单不被修改该。
