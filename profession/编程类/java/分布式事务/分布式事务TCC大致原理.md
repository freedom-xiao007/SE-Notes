# Seata TCC 大致应用原理
***
## 简介
&ensp;&ensp;&ensp;&ensp;初步看Seata示例代码时，大部分还是知道其意思，但自己具体写的时候，有些东西需要搞明白才能心中有数、灵活运用，这里大致说一下自己对于TCC一些理解。

&ensp;&ensp;&ensp;&ensp;使用伪代码进行不用TCC时程序如果保证原子性的示例，大致推断TCC框架做了那些工作，场景完全虚拟，不具有任何实际意义

### 业务场景
&ensp;&ensp;&ensp;&ensp;分布式事务出现与微服务的场景相关，这里的示例后台服务有如下两个：

- 用户服务：提供用户相关的接口调用，如账号金额变更之类的
- 商品服务：提供商家商品相关接口，如扣除库存之类的

&ensp;&ensp;&ensp;&ensp;比如现在有一个需求：提供一个购买接口，购买成功扣除用户的钱和商家的库存，这时需要去调用两个服务的相关接口函数，大致如下：

```java
public void userAccount(......) {
    // 进行用户金额操作
}

public void goodsAccount(......) {
    // 进行商品库存操作
}
```

&ensp;&ensp;&ensp;&ensp;上面两个接口函数的调用需要是原子性的，也就是必须两者同时成功。如果扣了钱，没买到货；买了或，没扣钱。那就有点严重了，这锅不能背。

&ensp;&ensp;&ensp;&ensp;如果在程序里面要保证这两个函数调用的原子性，按照TCC的思路，代码可能大概是下面这样的：

```java
// 购买：调用用户金额操作和商品库存操作
public void buy() {
    if (!deductUserAccount()) {
        // 用户账户预扣款不成功，直接返回操作失败
    }
    if (!deductGoodsAccount()) {
        // 商品存款预扣除不成功，因为前面扣除用户账户了，所以需要把钱设置回去
        restoreUserAccount();
    }

    if (!confirmUserAccount()) {
        // 用户账户最后更新失败，需要恢复前面两项的操作
        restoreUserAccount();
        restoreGoodsAccount();
    }
    if (!confirmGoodsAccount()) {
        // 用户账户最后更新失败，需要恢复前面两项的操作
        restoreUserAccount();
        restoreGoodsAccount();
    } 

    // 前面都出差，到这就成功了
}
```

&ensp;&ensp;&ensp;&ensp;从上面的代码可以看出，如果全是自己实现的话，代码还是比较繁琐的。如果不只两个的话，那代码写起来是挺酸爽的，每个业务的函数调用都需要确认是否成功，失败了就需要恢复前面业务函数调用前的数据。

&ensp;&ensp;&ensp;&ensp;业务函数的调用还是串行的，大部分业务函数可能分布在不同的服务，这里的服务调用是可以并行的，但是并行的话还是上面的那个问题，代码写起来复杂了