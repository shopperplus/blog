---
layout: post
title: "Service Object: What Why and How?"
author: Hooopo
---

首先想说的是，Service Object 是在Rails里实践SRP的一种手段和模式，不仅仅是一个文件夹。

## Controller是交互入口

像c/c++里，每个应用都会有一个入口，像下面这样：

```c
#include <iostream>
// Many includes...

int main(int argc, char *argv[]) {
  // Fetch your data.
  // Ex. Input data = Input.readFromUser(argc, argv);

  Application app = Application(data);
  app.start();

  // Cleanup logic...
  return 0;
}
```
如果运行上面的应用，main函数被调用，所有参数都被传递到`argv`变量。

随着c/c++程序代码量增长，没有人会把逻辑放到main函数里面，main函数里只初始化一些常驻对象，然后调用start之类的方法去调用我们的业务逻辑。

Rails的每个Action其实就相当于c/c++里的main函数。

Rails里，每个Action和main函数一样都是与外部交互的入口。Controller和Action虽然表现为类和方法，但是不同Action相互之间是没有交互的。
另外，Controller已经承担很多职责：

* 解析用户输入 -> params
* 渲染view -> render
* logging -> log
* routing -> redirect
* 输出提示 -> flash

## Servies Object

Service Object 封装了每一个业务流程。它负责组织应用领域模型（Model）之间的交互，并且不依赖于框架（Controller层）。你可以想象怎样的代码从Sinatra程序改成Rails会更容易，当然你不一定要这么做，我只是想解释一下什么是 `不依赖框架层`。

引人Service Object之后，可以带来很多好处：

* Controller更容易测试。
* 业务逻辑从Controller中剥离，更容易独立测试。
* 业务与框架低耦合。
* 让Controller更slim。

## Example 1

重构之前的charge more controller：

```ruby
class OrdersController
  def charge_execute
    @order = Order.find(params[:id])
    @order.total = params[:order][:total].to_f

    redirect_to :action => 'charge_more', :id => @order and return unless @order.total > 0

    # 1. init beanstream payment gateway
    gateway = ActiveMerchant::Billing::BeanstreamGateway.new(
            :login    => $BEAN_STREAM_ID,
            :user     => $BEAN_STREAM_LOGIN,
            :password => $BEAN_STREAM_PASSWD
    )

    # 2. init creditcard, options or pair values
    options = {
      :order_id => @order.id,
      :billing_address => {
        :name     => "#{@order.billing_first_name} #{@order.billing_last_name}",
        :phone    => @order.billing_phone,
        :address1 => @order.billing_street_address,
        :address2 => @order.billing_street_address_2,
        :city     => @order.billing_city,
        :state    => @order.get_state_code,
        :country  => @order.get_country_code,
        :zip      => @order.billing_zip
      },
      :email  => @order.email,
      :ref1   => "#{$DOMAIN_NAME} Order"
    }

    # 3. send payment info to gateway and deal with response
    response = gateway.purchase(@order.total_in_cents, @order.get_creditcard, options)

    if response.success?
      credit_card = @order.credit_card
      credit_card.transaction_id = credit_card.transaction_id + ',' + response.authorization
      credit_card.save
      flash[:notice] = "Extra money #{@order.total} has been charged"
    else
      flash[:notice] = "Error of processing charge: #{response.message}."
    end
    redirect_to :action => 'show', :id => @order and return
  end
end
```

重构之后：

```ruby
class OrdersController
  def charge_execute
    @order = Order.find(params[:id])
    @amount = params[:order][:total].to_f
    redirect_to :action => 'charge_more', :id => @order and return unless @amount > 0
    charge_logic = OrderChargeLogic.new(@order, @amount * 100)
    charge_logic.execute

    if charge_logic.success?
      flash[:notice] = "Extra money #{@amount} has been charged"
    else
      flash[:notice] = "Error of processing charge: #{charge_logic.message}."
    end
    redirect_to :action => 'show', :id => @order and return
  end
end
```

Servies:

```ruby
class OrderChargeLogic
  attr_reader :order, :amount, :gateway, :response, :creditcard_options, :credit_card

  def initialize(order, amount)
    @order, @amount = order, amount
    @credit_card = @order.credit_card
  end

  def success?
    response.success?
  end

  def message
    response.message
  end

  def execute
    init_beanstream_payment_gateway
    init_creditcard_options
    send_payment_info_to_gateway
  end

  private

  def init_beanstream_payment_gateway
    @gateway = ActiveMerchant::Billing::BeanstreamGateway.new(
            :login    => $BEAN_STREAM_ID,
            :user     => $BEAN_STREAM_LOGIN,
            :password => $BEAN_STREAM_PASSWD
          )
  end

  def init_creditcard_options
    @creditcard_options = {
      :order_id => order.id,
      :billing_address => {
        :name     => "#{@order.billing_first_name} #{@order.billing_last_name}",
        :phone    => order.billing_phone,
        :address1 => order.billing_street_address,
        :address2 => order.billing_street_address_2,
        :city     => order.billing_city,
        :state    => order.get_state_code,
        :country  => order.get_country_code,
        :zip      => order.billing_zip
      },
      :email  => order.email,
      :ref1   => "#{$DOMAIN_NAME} Order"
    }
  end

  def send_payment_info_to_gateway
    @response = gateway.purchase(amount, order.get_creditcard, creditcard_options)
    if @response.success?
      log_credit_card_transaction
    end
  end

  def log_credit_card_transaction
    credit_card.transaction_id = credit_card.transaction_id + ',' + @response.authorization
    credit_card.save
  end
end
```

## Example 2

[https://gist.github.com/hooopo/f6a031dac417323dfec6](https://gist.github.com/hooopo/f6a031dac417323dfec6)

引人OrderChargeLogic之后，收款这一个业务逻辑脱离了Controller，可以在任何地方（Rake
Task，Model，Background Task等）复用。Controller只负责调用OrderChargeLogic，根据OrderChargeLogic返回的状态去设置提示信息并且渲染。

也就是说，当你的项目越来越复杂，Model和Database Table不会完全一一对应了，同时也会有多个Model之间衍生出的业务逻辑，Service Object就是用来处理这部分内容。这部分逻辑不属于Controller，也不属于某一个Model。

如果你想让复杂Rails项目也能SRP，那么Service Object是一个值得尝试的手段。

[More Discuss](https://ruby-china.org/topics/24780)