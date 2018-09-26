# 业务监控Tag接入
## 主要功能
Metric业务指标最新版本支持多维度的tag，比如订单指标，可以额外加上来源、渠道等tag，这样当出现问题时候，可以根据来源、渠道等多种选择条件来看业务指标查询。每个Metric允许的多维度的tag乘积总数限制是1w，其实相比于一些faclon的指标上报，多tag一个指标最多可以充当1w个指标情况。这样可以极大解决业务指标监控和分析功能。

## 接入示例

场景示例：
```
1. 我想监控不同通道支付的不同状态订单数量。
2. 我想监控不同通道支付的不同状态的订单耗时。

```
### Step0-确定监控对象
项目名（appkey）为com.sankuai.cat.metric.demo，订单数量的指标定义为paystatus，订单耗时的指标定义为payduration。

### Step1-埋点
#### 埋点示例

```
public void metricDemo() {
   	// 业务代码
	    String status = "success";
	    String channel = "weixin";
	    long duration = 1000L;  //通过业务代码自己统计出来
	    
		//构建多维度指标tag并赋值
		Map<String, String> tags = new HashMap<String, String>();
		tags.put("status", status);
		tags.put("channel", channel);
		
		// 埋次数统计的点，每调用一次，次数加1
		Cat.logMetricForCount("paystatus", tags);
		
		// 埋次数统计的点，每调用一次，次数加3
		Cat.logMetricForCount("paystatus", 3,  tags);
		
		//埋一个耗时统计的点,duration接口参数为毫秒，cat会统计每分钟上报的duration的平均值。
		Cat.logMetricForDuration("payduration", duration, tags);
   }
```


#### API详细解释

##### 1.API定义

- logMetricForCount(String name, Map<String, String\> tags);

- logMetricForCount(String name, int quantity, Map<String, String\> tags);

- logMetricForDuration(String name, int quantity, Map<String, String\> tags);


##### 2. API说明
- logMetricForCount用于次数这种计数的指标，展示的图表算累计值；
- logMetricForDuration用于耗时类的指标，展示的图表算平均值。
- 多维度的tags的key和value的长度，必须限制在200个字符以内，超过则上报失败。


#### 注意事项
1. `打点尽量用纯英文，不要带一些特殊符号，例如 空格( )、分号(:)、竖线(|)、斜线(/)、逗号(,)、与号(&)、星号(*)、左右尖括号(<>)、以及一些奇奇怪怪的字符。`
2. `如果有分隔需求，建议用下划线(_)、中划线(-)、英文点号(.)等。`
3. `由于数据库不区分大小写，请尽量统一大小写，并且不要对大小写进行改动`

### Step2-报表数据查看
进入CAT后台application板块, business模块的业务指标(分钟)页面, 输入项目名, 可以查看到项目名对应的指标。

### Step3-编辑指标

- CAT会自动为埋点数据生成指标, 以及指标的各个维度标签，如果没有特殊需要，不需要提前配置。

指标和标签的编辑是在CAT后台Config板块，业务标签配置模块进行

说明：

- 如果是count类型的指标，请选择显示次数曲线；如果是duration类型的指标，请选择显示平均曲线。
- 该步骤中，businessKey对应```logMetricForCount(String name, Map<String, String> tags)``` 中的name，也即打点示例中的paystatus和payduration。
- 纵坐标范围： 为了避免个别极端数据对图表造成影响，可以设置纵坐标的范围，则图表的纵坐标展示范围会限制在该范围内，范围外的点不会展示。
- 是否对基线做平滑处理： 对于需要计算基线的指标，默认会进行平滑处理，避免一些突增突降的点对基线造成太大的影响。而在有的场景下，比如外卖定时开关店等，一些时间点的突增突降是合理的，基线需要体现这些突增突降，因此，增加该配置，可以控制是否对基线做平滑处理。
 


