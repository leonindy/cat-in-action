## Transaction报表
### 主要功能

监控一段代码运行情况：`运行次数、QPS、错误次数、失败率、响应时间统计（平均影响时间、Tp9XXX分位值）等等`。

应用启动后默认会打点的部分:

|  打点 | 来源组件 | 描述 |
| --- | --- | --- |
| System | cat | 上报监控数据的打点信息 |
| URL |  cat-filter | URL访问的打点信息 |


### 报表介绍
#### 第一级分类（Type）统计界面

Type统计界面展示了一个Transaction的第一层分类的视图，可以知道这段时间里面一个分类运行的次数，平均响应时间，延迟，95线以及99线。

![](../../resources/ch1-report/transacition_view.jpg)


从上而下分析报表：

1. 点击选择时间：小时报表，默认是以一小时为统计时间跨度；点击可以进行不同时间快速选择，例如要看2018-05-27 18:00 ~ 2018-05-27 18:59数据，点击选择2018-05-27 18:00即可。
2. 输入项目名，查看项目数据：页面会默认打开上次查看的项目名数据，如果需要切换其他项目数据，输入项目名，回车即可。
3. 点击切换到历史模式：默认页面是小时模式，与之对应的右侧快速导航，分别对应-1d -1h +1h +1d now，这四种都是切换到不同时间的小时报表；切换为历史模式后，右侧快速导航，变为month(月报表)、week(周报表)、day(天报表)，可以点击进行查看，注意时间跨度会有所不同。  
4. 机器列表：展示了当前小时所有上报了监控数据的机器列表
5. 第一层分类（Type），点击查看第二级分类（称为<font color=#FF4500>name</font>）数据：
    - Transaction的埋点的Type和Name由业务自己定义，当打点了Cat.newTransaction(type, name)时，第一层分类是type，第二级分类是name。
    - 第二级分类数据叫是统计相同type下的所有name数据，如下图，数据均与第一级（type）一样的展示风格：
    ![](../../resources/ch1-report/transaction_name_view.jpg)

6. 点击show，查看Type所有name分钟级统计，如下图:
![](../../resources/ch1-report/transaction_name_1.jpg)
![](../../resources/ch1-report/transaction_name_2.jpg)
![](../../resources/ch1-report/transaction_name_3.jpg)

7. 显示的是小时粒度第一级分类（type）的次数、错误数、失败率等数据
8. 小时粒度的时间第一级分类（type）相关统计

    - 95line表示95%的请求的响应时间比参考值要小，99line表示99%的响应时间比参考值要小，95line以及99line，也称之为tp95,tp99。

#### Transaction历史报表

Transaction历史报表支持每天、每周、每月的数据统计以及趋势图，点击导航栏的切换历史模式进行查询。Transaction历史报表以响应时间、访问量、错误量三个维度进行展示，以天报表为例：选取一个type，点击show

![](../../resources/ch1-report/transaction_history.jpg)

### 最佳实践
#### TP95 TP99 等计算逻辑
- cat这边是基于全量数据进来，在单台机器的上单个小时内部，做全量的区间分桶，在区间分桶也做了部分优化，时间越大桶的区间粒度越大，最后根据count找出99分位。
- cat内部是自己实现的模型和存储，本身支持增加合并和计算，tp99这类并不是适合增量计算，必须依赖全量数据才可以，但是全量的数据存储量非常之大。<font color=#FF4500>Transaction报表目前准确统计小时范围单台机器级别分钟级的TP数据，业务设置告警可以设置机器级别的TP值告警。</font>。如果看小时级别ALL的统计，是根据当前小时所有的机器进行合并的结果，主要一段时间内所有的机器响应时间波动基本一致，小时级别基本也准确，但是如果个别机器抖动特别大，这个所有的统计会失真。
- API报表统计了URL、PigeonService、OctoService三个机器粒度TP值，也有TP9999的值，API报表没有IP的细分统计，所有TP值的统计相比transaction里面的ALL合并的TP值统计更加准确。
- cat后端天、周、月级别的数据是就小时数据的合并，主要用了加权来做，如果一天整个项目的99的波动不大，天级别的99约等于小时级别99。如果一天内的某几个小时的tp99 比较大的话，此做法就会有误差，比如某个服务一天有好几个小时tp99的数据都是正常其他时间段的10倍，这样天级别的99误差就比较大。简而言之天、周、月TP值的统计是一个根据加权算法的近似估计值。
- TP9XX的统计数据，单台机器小时报表的小时粒度和分钟粒度是完全准确的，天、周、月级别、包括集群的总的TP9XX都是加权来计算的。计算逻辑如下：
    - 机器A的tp999是timeA，访问总量是countA
    - 机器B的tp999是timeB,访问总量是countB
    - 集群（假设只有A和B两个机器）的tp999的计算公式是：tp999 = (timeA * countA + timeB * countB) /(countA + countB)
    - 天、周、月报表计算逻辑类似，只不过把机器之间的运算换成了时间之间的运算

-  另外catTP9XX等小时内部的统计为了节省内存，也是用了近似的统计方式，当响应时间越大的话，使用的误差也越大

```
	public static int computeDuration(int duration) {
		if (duration < 20) {
			return duration + 1;
		} else if (duration < 200) {
			return duration - duration % 5;
		} else if (duration < 500) {
			return duration - duration % 20;
		} else if (duration < 2000) {
			return duration - duration % 50;
		} else {
			return duration - duration % 200;
		}
	}

```

#### logview中transaction有时候状态为unset含义
- 早期版本cat-client，transaction状态中默认值为unset，如果业务代码没有主动设置就会是默认值unset。

    比如如下埋点场景，比如在businessCode 中抛出了一个Error，比如NoSuchMothod，或者OutOfMemory等，就会导致直接complete，既走不到t.setStatus，也走不到catch代码里面。

```
			Transaction t = Cat.newTransaction("AlertApp", TimeHelper.getMinuteStr());

			try {
				businessCode(); 
				
				t.setStatus(Transaction.SUCCESS);
			} catch (Exception e) {
				t.setStatus(e);
				Cat.logError(e);
			} finally {
				t.complete();
			}

```


### 示例说明
#### 我想监控一段核心代码的执行的情况，能看到统计报表，怎么办？
- 首先，确定监控对象
   - 项目名为cat
        - 监控一个Class（比如MVC.java）中多个接口
        - 接口包括：InboundPhase,OutboundPhase,TransitionPhase
- 其次，确定监控指标
        - 每分钟调用次数
        - 每分钟的平均响应时间
        - 最慢会多久
        - 大部分的请求时间分布
- 然后，进行埋点（前提是已经集成cat-client)
    
   - 假设原有MVC.java代码为：
   
```java
 public boolean InboundPhase() {        
   		boolean result = true;
   
   		try {
   			result = doMyJob();
   		} catch (Exception e) {
   			result = false;
   			logger.error(e);
			 // 也可以选择向上抛出： throw e;
   		}
           		
   		return result;
   	}

```
        	
   - 在接口MVC.java中加入Cat埋点
        
```java
  Transaction transaction = Cat.newTransaction("MVC", "InboundPhase");
   
	try {
	
      // TODO  原本Interface01所有代码逻辑
      
		transaction.setSuccessStatus();
	} catch (Exception e) {
		transaction.setStatus(e); // catch 到异常，设置状态，代表此请求失败
		Cat.logError(e); // 将异常上报到cat上
		// 也可以选择向上抛出： throw e;
	} finally {
		transaction.complete();
	}
```

   - 最终InboundPhase完整代码为：
   
```java
public boolean Interface01() {
		Transaction transaction = Cat.newTransaction("MVC", "InboundPhase");
		boolean result = true;
     
		try {
			result = doMyJob();
     
			if (result) {
				transaction.setSuccessStatus();
			} else {
				transaction.setStatus("failed"); // 除"0"之外的任意字符均为失败
			}
		} catch (Exception e) {
			transaction.setStatus(e);
			Cat.logError(e);
			// 也可以选择向上抛出： throw e;
		} finally {
			transaction.complete();
		}
		return result;
}
```

