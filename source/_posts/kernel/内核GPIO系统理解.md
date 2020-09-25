---
title: 内核GPIO系统理解
type: kernel
description: linux内核中对于硬件平台的GPIO口的管理已经标准化了，目前通过gpio子系统和pinctrl子系统可以本方便的将所有GPIO进行配置管理。对驱动工程师而言，完全可以不关注board相关的内容，只需要知道“我要使用哪个GPIO，我要干什么”，然后直接request、set等操作，可以更加专注应用层面的逻辑，无需关心偏底层的细节了。
date: 2020-09-15 10:00:00
---

linux内核中对于硬件平台的GPIO口的管理已经标准化了，目前通过gpio子系统和pinctrl子系统可以本方便的将所有GPIO进行配置管理。对驱动工程师而言，完全可以不关注board相关的内容，只需要知道“我要使用哪个GPIO，我要干什么”，然后直接request、set等操作，可以更加专注应用层面的逻辑，无需关心偏底层的细节了。


---
## GPIO系统整体架构

<img src="/images/gpio_framework.png">


根据上图，驱动程序根据设备树**compatible**匹配到一个或一组GPIO，在**probe**中调用pinctrl子系统API设置GPIO口的为哪种功能（有些GPIO口可以复用I2C/SPI/PWM等），然后调用GPIO子系统的API来获取GPIO资源，读写GPIO。

至于在往下层的东西，都是BSP来做的事情，linux内核已经搭建好了这一套框架，在这套框架内，**驱动开发者**和**BSP开发者**分属于这个框架的两个角色，两者通过框架提供的API来联系，屏蔽了众多细节，从而实现让专业的人来做专业的事。

---
## 驱动工程师如何开发GPIO驱动程序

### 1. 首先了解两个子系统的API

GPIO子系统的API分为descriptor-based 新版API，和旧版API（integer-based）

---
#### GPIO子系统API
   
|新API（descriptor-based ）|旧API（integer-based）| 功能 | 
|--------------------------|---------------------|-----|
|#include <linux/gpio.h|#include <linux/gpio/consumer.h>|头文件|
||gpio_is_valid()|检测gpio是否可用|
|gpiod_get()|gpio_request()|申请gpio资源|
|gpiod_get_index()|gpio_request_one()||
|gpiod_get_array()|gpio_request_array()||
|devm_gpiod_get()|||
|devm_gpiod_get_index()|||
|devm_gpiod_get_array()|||
|gpiod_direction_input()|gpio_direction_input()|设置gpio输入、输出模式|
|gpiod_direction_output()|gpio_direction_output()||
|gpiod_put()|gpio_free()|释放gpio资源|
|gpiod_put_array()|gpio_free_array()||
|gpiod_get_value()|gpio_get_value(unsigned gpio)|读写gpio值|
|gpiod_set_value()|gpio_set_value(unsigned gpio, int value)||
|devm_gpiod_put()|||
|devm_gpiod_put_array()|||

---
#### pinctrl子系统API

* struct pinctrl * __must_check pinctrl_get(struct device *dev);
* void pinctrl_put(struct pinctrl *p);
* struct pinctrl * __must_check devm_pinctrl_get(struct device *dev);
* void devm_pinctrl_put(struct pinctrl *p);
* struct pinctrl_state * __must_check pinctrl_lookup_state(struct pinctrl *p,const char *name);
* int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s);

### 2. 配置设备树，确定你要使用的GPIO资源

---
#### pinctrl子系统设备树

![图一](/images/pinctrl.png)

上图是厂家构建好的pinctrl子系统的设备树，厂家也已经写好了匹配 **compatible = "qcom,ipq6018-pinctrl";**的BSP驱动程序，对于驱动开发只要能看懂这个就行了。

![图二](/images/pinctrl_node.png)

该图描述了pinctrl子系统中，包含了4个GPIO节点作为led节点，分别对应GPIO 16/18/24/25。事实上，根据pinctrl子系统的概念，这里定义的 **led_pins**的含义是：把4个GPIO归为一组用于led控制，对每个GPIO的电气特性分别作了定义：

* **pins = "gpio18";** 指定配置的GPIO口
* **function = "gpio";**  GPIO的复用功能，这里作为普通GPIO口使用
* **drive-strength = <8>;** GPIO口的电流驱动能力（根据厂家定义，每个数字对应一个输出电流值mA）
* **bias-pull-down;** 这是对引脚上/下拉电阻的配置 有关概念参考[GPIO上拉下拉概念的理解](http://mcu.eetrend.com/blog/2019/100046180.html)

也就是说，如果你在自己配置驱动设备树时采用了这一组配置，那么在你执行pinctrl子系统API**pinctrl_lookup_state**时，pinctrl的BSP部分就会把这4个GPIO口的电气特性配置成上述值。

当然，在这里这4个GPIO仅仅是作为普通GPIO口使用的，如果这些GPIO口还有其它复用功能，比如：spi总线的功能，那么你可以再做一组配置，例如：

```
&tlmm {
	spi_pins: spi_pins {
		spi_cs {
			pins = "gpio25";
			function = "spi";
			drive-strength = <8>;
			bias-pull-down;
		};
		spi_rx {
			pins = "gpio24";
			function = "spi";
			drive-strength = <8>;
			bias-pull-down;
		};
		spi_tx {
			pins = "gpio18";
			function = "spi";
			drive-strength = <8>;
			bias-pull-down;
		};
		spi_clk {
			pins = "gpio16";
			function = "spi";
			drive-strength = <8>;
			bias-pull-down;
		};
	};	
};
```

这样的话呢，在下面的图中，你可以修改成这样：
```
	led_device {
		compatible = "gbcom,leds";
		pinctrl-0 = <&led_pins>;
		pinctrl-1 = <&spi_pins>;
		pinctrl-names = "default","spi";
		gpios = <&tlmm 16 GPIO_ACTIVE_HIGH>;
		eth_gpios = <&tlmm 18 GPIO_ACTIVE_HIGH>;
		two_gpios = <&tlmm 24 GPIO_ACTIVE_HIGH>,<&tlmm 24 GPIO_ACTIVE_HIGH>;
	};
```

如上所示，当你调用 **pinctrl_lookup_state**时，第二个参数 指定**default**,那么4个GPIO配置成led_pins的特性，如果指定**spi**,那么4个GPIO就可以复用为SPI总线使用了。

上面的修改只是举个例子，有助于理解pinctrl子系统

---
#### 配置gpio设备树资源

![图三](/images/pinctrl_driver_node.png)

上面的额设备树就是需要自行构建的了，原理是这样的：

* 厂家实现pinctrl的BSP驱动，将所有GPIO口都纳入了pinctrl控制，同时定义了部分GPIO的电气特性（也就是图二）
* 然后我们要使用这些GPIO口，并想使用这个GPIO口的某种复用功能，那么需要我们自行定义。

设备树中字段的含义如下：

* compatible = "gbcom,leds";  //用于驱动进行probe
* pinctrl-0 = <&led_pins>; // 选择GPIO口的复用功能（哪种电气特性）
* pinctrl-names = "default"; // 起个名字，用于 pinctrl_select_state 传参
* gpios = <&tlmm 16 GPIO_ACTIVE_HIGH>; // 指定使用的GPIO口
* eth-gpios = <&tlmm 18 GPIO_ACTIVE_HIGH>;
* two-gpios = <&tlmm 24 GPIO_ACTIVE_HIGH>,<&tlmm 24 GPIO_ACTIVE_HIGH>;

>这里需要注意，指定GPIO口的属性名称必须是 gpio 或者gpios 或者 xxx-gpio 或者 xxx-gpios，如果没有前缀，那么在使用**新API**申请GPIO资源时 **gpiod_get(struct device *dev,const char *con_id,enum gpiod_flags flags)** 其中参数 **con_id** 应为 **NULL**，如果有前缀，那么 **con_id** 应为 **前缀xxx**


---
### 3. 写驱动

#### 第一步：platfrom驱动固定套路

```
static struct of_device_id leds_match_table[] = {
  {.compatible = "gbcom,leds"},
  {},
};
MODULE_DEVICE_TABLE(of, leds_match_table);

static struct platform_driver leds_driver = {
  .probe  = leds_probe,
  .remove = leds_remove,
  .driver = {
    .name   = "led_power",
    .owner  = THIS_MODULE,
    .of_match_table = leds_match_table,
  },
};

module_platform_driver(leds_driver);

MODULE_LICENSE("GPL v2");
MODULE_VERSION("0.1");
MODULE_ALIAS("platform:led_power");
```

注意：`.compatible = "gbcom,leds"` 跟设备树对应

#### 第二步：重点是probe函数

```
static int leds_probe(struct platform_device *pdev)
{
  int ret,loop;
  struct device *dev = &pdev->dev;
  struct pinctrl_state *dft_state;
  struct gpio_leds_dat *pleds;//我们自定义了一个led数据结构，用来存储每个led的信息

    //申请空间，使用 devm_kzalloc 是基于device的内存，当设备卸载时，内存会被自动回收
	pleds = devm_kzalloc(dev,sizeof(struct gpio_leds_dat),GFP_KERNEL);
	if (!pleds)
		return ERR_PTR(-ENOMEM);
	memset(pleds,0,sizeof(struct gpio_leds_dat));
	pleds->num = 4;

    // pinctrl API，获取pinctrl句柄
  pleds->pinctrl_leds = devm_pinctrl_get(dev);
  if (IS_ERR(pleds->pinctrl_leds)) {
    ret = PTR_ERR(pleds->pinctrl_leds);
    printk("devm_pinctrl_get fail\n");
    goto error;
  }

    /*查找我们设备树中设置的GPIO默认状态*/
  dft_state = pinctrl_lookup_state(pleds->pinctrl_leds,"default");
  if (IS_ERR(dft_state)) {
    ret = PTR_ERR(dft_state);
    printk("pinctrl_lookup_state fail\n");
    goto error;
  }
    /*是默认状态的电气特性生效*/
  pinctrl_select_state(pleds->pinctrl_leds,dft_state);

  /*GPIO API使用，获取我们设备树中配置的4个GPIO*/
  pleds->leds[0].led_desc = devm_gpiod_get(dev,NULL,GPIOD_OUT_HIGH);            /*设备树中属性名是gpio，第二个参数是NULL*/
  pleds->leds[1].led_desc = devm_gpiod_get(dev,"eth",GPIOD_OUT_HIGH);           /*找设备树中属性名是eth-gpio(s)的GPIO，第二个参数是“eth”*/
  pleds->leds[2].led_desc = devm_gpiod_get_index(dev,"two",0,GPIOD_OUT_HIGH);   /*我们设备树中属性two-gpios指定了2个GPIO，那么第三个参数用来确定要找的GPIO位置索引*/
  pleds->leds[3].led_desc = devm_gpiod_get_index(dev,"two",1,GPIOD_OUT_HIGH);

    /*每个GPIO都指定一个名称，我们这里只是为了简单，其实可以在设备树中指定名称，然后读取的*/
  strncpy(pleds->leds[0].name,"none",sizeof("none")); 
  strncpy(pleds->leds[1].name,"eth",sizeof("eth"));
  strncpy(pleds->leds[2].name,"two0",sizeof("two0"));
  strncpy(pleds->leds[3].name,"two1",sizeof("two1"));
  
  /* 1. 检测获取的GPIO描述符是否有效，并标记
     2. 获取每个GPIO是否是睡眠GPIO（cansleep GPIO的配置需要在任务上下文中）
     3. 每个GPIO初始化一个工作任务，如果是cansleep的，那么需要通过这个WORK来设置GPIO的输入、输出
  */
  for(loop = 0;loop < 4; loop ++){
    pleds->leds[loop].vaild = 1;
    if(IS_ERR(pleds->leds[loop].led_desc)){
      printk("Error:get %s's gpio_desc fail\n",pleds->leds[loop].name);
      pleds->leds[loop].vaild = 0;
      continue;
    }
    pleds->leds[loop].can_sleep = gpiod_cansleep(pleds->leds[loop].led_desc);
    INIT_WORK(&pleds->leds[loop].work,gpio_led_work);
    gpiod_direction_output(pleds->leds[loop].led_desc,0);
    printk("led[%d] data:%-6s,can_sleep=%d\n",loop,pleds->leds[loop].name,pleds->leds[loop].can_sleep);
  }
  
    /*创建一个sysfs属性节点，方便我们在用户态命令行测试*/
	ret = sysfs_create_group(&dev->kobj,&led_attr_group);
	if(ret < 0){
    printk("leds_probe:sysfs_create_group 'state/foo' fail\n");
    goto error;
	}

    /*将数据放入到device的私有数据区，这样，我们在其它地方都可以获取到这个数据*/
	platform_set_drvdata(pdev,pleds);
	printk("leds_probe ok\n");
  return 0;
error:
  devm_kfree(dev,pleds);
  return ret;
}
```

### 4. 完整的程序

```
#include <linux/module.h>
#include <linux/timer.h>
#include <linux/device.h>
#include <linux/err.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/pinctrl/consumer.h>
#include <linux/platform_device.h>
#include <linux/gpio/consumer.h>
#include <linux/property.h>
#include <linux/workqueue.h>

struct leds_dat{
  int vaild;
  char name[32];
  struct gpio_desc *led_desc;
	struct work_struct work;
	int can_sleep;
	int status;
};
struct gpio_leds_dat{
  int num;
  struct pinctrl *pinctrl_leds;
  struct leds_dat leds[4];
};
ssize_t led_state_show(struct device *dev, struct device_attribute *attr,
		char *buf)
{
  struct gpio_leds_dat *pleds = dev_get_drvdata(dev);
  return sprintf(buf,"%d\n",pleds->leds[0].status);
}
ssize_t led_state_store(struct device *dev, struct device_attribute *attr,
    const char *buf, size_t count)
{
  int ret,status,loop;
  struct gpio_leds_dat *pleds = dev_get_drvdata(dev);
  
  ret = kstrtoint(buf,10,&status);
  if(ret < 0){
    status = -1;
    printk("Error,input param '%s' wrong\n",buf);
    goto error;
  }

  for(loop = 0;loop < pleds->num;loop ++){
    if(pleds->leds[loop].vaild == 0)
      continue;
    pleds->leds[loop].status = status;
    if(pleds->leds[loop].can_sleep){
      schedule_work(&pleds->leds[loop].work);
    }else{
       gpiod_set_value(pleds->leds[loop].led_desc,status);
    }
  }
error:
	return count;
}

static DEVICE_ATTR(state, 0644, led_state_show, led_state_store);

static struct attribute *led_attributes[] = {
  &dev_attr_state.attr,
  NULL
};
static const struct attribute_group led_attr_group = {
  .attrs = led_attributes,
};
static void gpio_led_work(struct work_struct *work)
{
  struct leds_dat *pdat = container_of(work,struct leds_dat,work);

  if(pdat->vaild){
    gpiod_set_value(pdat->led_desc,pdat->status);
    printk("led work set %s status[%d]\n",pdat->name,pdat->status);
  }else{
    printk("led work %s vaild\n",pdat->name);
  }
  return ;
}

static int leds_probe(struct platform_device *pdev)
{
  int ret,loop;
  struct device *dev = &pdev->dev;
  struct pinctrl_state *dft_state;
  struct gpio_leds_dat *pleds;

	pleds = devm_kzalloc(dev,sizeof(struct gpio_leds_dat),GFP_KERNEL);
	if (!pleds)
		return ERR_PTR(-ENOMEM);
	memset(pleds,0,sizeof(struct gpio_leds_dat));
	pleds->num = 4;

	
  pleds->pinctrl_leds = devm_pinctrl_get(dev);
  if (IS_ERR(pleds->pinctrl_leds)) {
    ret = PTR_ERR(pleds->pinctrl_leds);
    printk("devm_pinctrl_get fail\n");
    goto error;
  }

  dft_state = pinctrl_lookup_state(pleds->pinctrl_leds,"default");
  if (IS_ERR(dft_state)) {
    ret = PTR_ERR(dft_state);
    printk("pinctrl_lookup_state fail\n");
    goto error;
  }

  pinctrl_select_state(pleds->pinctrl_leds,dft_state);


  pleds->leds[0].led_desc = devm_gpiod_get(dev,NULL,GPIOD_OUT_HIGH);
  pleds->leds[1].led_desc = devm_gpiod_get(dev,"eth",GPIOD_OUT_HIGH);
  pleds->leds[2].led_desc = devm_gpiod_get_index(dev,"two",0,GPIOD_OUT_HIGH);
  pleds->leds[3].led_desc = devm_gpiod_get_index(dev,"two",1,GPIOD_OUT_HIGH);

  strncpy(pleds->leds[0].name,"none",sizeof("none"));
  strncpy(pleds->leds[1].name,"eth",sizeof("eth"));
  strncpy(pleds->leds[2].name,"two0",sizeof("two0"));
  strncpy(pleds->leds[3].name,"two1",sizeof("two1"));
  
  for(loop = 0;loop < 4; loop ++){
    pleds->leds[loop].vaild = 1;
    if(IS_ERR(pleds->leds[loop].led_desc)){
      printk("Error:get %s's gpio_desc fail\n",pleds->leds[loop].name);
      pleds->leds[loop].vaild = 0;
      continue;
    }
    pleds->leds[loop].can_sleep = gpiod_cansleep(pleds->leds[loop].led_desc);
    INIT_WORK(&pleds->leds[loop].work,gpio_led_work);
    gpiod_direction_output(pleds->leds[loop].led_desc,0);
    printk("led[%d] data:%-6s,can_sleep=%d\n",loop,pleds->leds[loop].name,pleds->leds[loop].can_sleep);
  }
  
	ret = sysfs_create_group(&dev->kobj,&led_attr_group);
	if(ret < 0){
    printk("leds_probe:sysfs_create_group 'state/foo' fail\n");
    goto error;
	}

	platform_set_drvdata(pdev,pleds);
	printk("leds_probe ok\n");
  return 0;
error:
  devm_kfree(dev,pleds);
  return ret;
}
static int leds_remove(struct platform_device *pdev)
{
  int loop;
  struct device *dev = &pdev->dev;
  struct gpio_leds_dat *pleds = platform_get_drvdata(pdev);
  
  sysfs_remove_group(&dev->kobj,&led_attr_group);

  devm_pinctrl_put(pleds->pinctrl_leds);

  for(loop = 0;loop < pleds->num;loop ++){
    if(pleds->leds[loop].vaild == 0)
      continue;
      
    cancel_work_sync(&pleds->leds[loop].work);
    devm_gpiod_put(dev,pleds->leds[loop].led_desc);
  }
  
	printk("leds_remove ok\n");
  return 0;
}

static struct of_device_id leds_match_table[] = {
  {.compatible = "gbcom,leds"},
  {},
};
MODULE_DEVICE_TABLE(of, leds_match_table);

static struct platform_driver leds_driver = {
  .probe  = leds_probe,
  .remove = leds_remove,
  .driver = {
    .name   = "led_power",
    .owner  = THIS_MODULE,
    .of_match_table = leds_match_table,
  },
};

module_platform_driver(leds_driver);

MODULE_LICENSE("GPL v2");
MODULE_VERSION("0.1");
MODULE_ALIAS("platform:led_power");

```

### 5. 测试

```
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# ls
driver           driver_override  modalias         of_node          power            state            subsystem        uevent
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# echo 1 > state
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# 
root@OpenWrt:/sys/devices/platform/soc/soc:led_device#
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# echo 0 > state
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# 
root@OpenWrt:/sys/devices/platform/soc/soc:led_device# while true;do echo 1 > state ;sleep 1;echo 0 > state;sleep 1;done
```

### 6. 效果

![led闪烁](/images/led.gif)



