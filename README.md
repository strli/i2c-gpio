# i2c-gpio.c

4.17内核版本以后，i2c-gpio被重新定义了见：

https://www.kernel.org/doc/html/v4.17/driver-api/gpio/consumer.html

许多内核子系统仍使用传统的基于整数的接口处理GPIO。虽然强烈建议将它们升级到更安全的基于描述符的API，但以下两个函数允许您将GPIO描述符转换为GPIO整数命名空间，反之亦然：

```
int desc_to_gpio（const struct gpio_desc * desc）
struct gpio_desc * gpio_to_desc（unsigned gpio）
```

只要尚未释放GPIO描述符，就可以安全地使用desc_to_gpio（）返回的GPIO编号。同样，必须正确获取传递给gpio_to_desc（）的GPIO号，并且只有在释放GPIO号后才能使用返回的GPIO描述符。

禁止使用其他API释放由一个API获取的GPIO，并且取消选中该错误。



具体改变如下：

1、基础结构体

文件（i2c-gpio.h)

无sda,scl

```
struct i2c_gpio_platform_data {
	int		udelay;
	int		timeout;
	unsigned int	sda_is_open_drain:1;
	unsigned int	scl_is_open_drain:1;
	unsigned int	scl_is_output_only:1;
};
```

2、核心模块i2c-gpio

文件（i2c-gpio.c)

结构体：

//定义i2c参数的结构体。

//其中pdata用于设置头文件结构的参数

//gpio_desc用于定义sda与scl

```
struct i2c_gpio_private_data {

​    struct gpio_desc *sda;

​    struct gpio_desc *scl;

​    struct i2c_adapter adap;

​    struct i2c_algo_bit_data bit_data;

​    struct i2c_gpio_platform_data pdata;

\#ifdef CONFIG_I2C_GPIO_FAULT_INJECTOR

​    struct dentry *debug_dir;

\#endif

};


```

//用与定义sda，scl的结构体

```
static struct gpio_desc *i2c_gpio_get_desc(struct device *dev,

​                       const char *con_id,

​                       unsigned int index,

​                       enum gpiod_flags gflags)

{

​    struct gpio_desc *retdesc;

​    int ret;

​    retdesc = devm_gpiod_get(dev, con_id, gflags);

​    if (!IS_ERR(retdesc)) {

​        dev_dbg(dev, "got GPIO from name %s\n", con_id);

​        return retdesc;

​    }

​    retdesc = devm_gpiod_get_index(dev, NULL, index, gflags);

​    if (!IS_ERR(retdesc)) {

​        dev_dbg(dev, "got GPIO from index %u\n", index);

​        return retdesc;

​    }

​    ret = PTR_ERR(retdesc);

​    /* FIXME: hack in the old code, is this really necessary? */

​    if (ret == -EINVAL)

​        retdesc = ERR_PTR(-EPROBE_DEFER);

​    /* This happens if the GPIO driver is not yet probed, let's defer */

​    if (ret == -ENOENT)

​        retdesc = ERR_PTR(-EPROBE_DEFER);



​    if (ret != -EPROBE_DEFER)

​        dev_err(dev, "error trying to get descriptor: %d\n", ret);



​    return retdesc;

}
```



//两个重要的函数

 devm_gpiod_get(dev, con_id, gflags);

devm_gpiod_get_index(dev, NULL, index, gflags);



所属路径：

- [**drivers/gpio/devres.c**, line 62 *(as a function)*](https://elixir.bootlin.com/linux/v4.7/source/drivers/gpio/devres.c#L62)

- [**drivers/gpio/devres.c**, line 68 *(as a variable)*](https://elixir.bootlin.com/linux/v4.7/source/drivers/gpio/devres.c#L68)

- [**include/linux/gpio/consumer.h**, line 209 *(as a function)*](https://elixir.bootlin.com/linux/v4.7/source/include/linux/gpio/consumer.h#L209)

- [**include/linux/gpio/consumer.h**, line 73 *(as a prototype)*](https://elixir.bootlin.com/linux/v4.7/source/include/linux/gpio/consumer.h#L73)

- 第一个中

  对devm_gpiod_get 定义如下：

  ```
  struct gpio_desc *__must_check devm_gpiod_get(struct device *dev,
  					      const char *con_id,
  					      enum gpiod_flags flags)
  {
  	return devm_gpiod_get_index(dev, con_id, 0, flags);
  }
  ```

返回值为devm_gpiod_get_index

devm_gpiod_get_index 方法如下：

```
struct gpio_desc *__must_check devm_gpiod_get_index(struct device *dev,
						    const char *con_id,
						    unsigned int idx,
						    enum gpiod_flags flags)
{
	struct gpio_desc **dr;
	struct gpio_desc *desc;

	dr = devres_alloc(devm_gpiod_release, sizeof(struct gpio_desc *),
			  GFP_KERNEL);
	if (!dr)
		return ERR_PTR(-ENOMEM);

	desc = gpiod_get_index(dev, con_id, idx, flags);
	if (IS_ERR(desc)) {
		devres_free(dr);
		return desc;
	}

	*dr = desc;
	devres_add(dev, dr);

	return desc;
}
EXPORT_SYMBOL(devm_gpiod_get_index);
```

返回值为desc结构体



位置为/driver/gpio/gpiolib.h

desc的结构体如下：

```
struct gpio_desc {
	struct gpio_device	*gdev;
	unsigned long		flags;
```



结构提gpio_device 如下：（也在gpiolib.h中)

```
struct gpio_device {
	int			id;
	struct device		dev;
	struct cdev		chrdev;
	struct device		*mockdev;
	struct module		*owner;
	struct gpio_chip	*chip;
	struct gpio_desc	*descs;
	int			base;
	u16			ngpio;
	char			*label;
	void			*data;
	struct list_head        list;

#ifdef CONFIG_PINCTRL
	/*
	 * If CONFIG_PINCTRL is enabled, then gpio controllers can optionally
	 * describe the actual pin range which they serve in an SoC. This
	 * information would be used by pinctrl subsystem to configure
	 * corresponding pins for gpio usage.
	 */
	struct list_head pin_ranges;
#endif
};
```

其中gpio_chip在/include/linux/gpio/driver.h下

```
struct gpio_chip {
	const char		*label;
	struct gpio_device	*gpiodev;
	struct device		*parent;
	struct module		*owner;

	int			(*request)(struct gpio_chip *chip,
						unsigned offset);
	void			(*free)(struct gpio_chip *chip,
						unsigned offset);
	int			(*get_direction)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_input)(struct gpio_chip *chip,
						unsigned offset);
	int			(*direction_output)(struct gpio_chip *chip,
						unsigned offset, int value);
	int			(*get)(struct gpio_chip *chip,
						unsigned offset);
	void			(*set)(struct gpio_chip *chip,
						unsigned offset, int value);
	void			(*set_multiple)(struct gpio_chip *chip,
						unsigned long *mask,
						unsigned long *bits);
	int			(*set_debounce)(struct gpio_chip *chip,
						unsigned offset,
						unsigned debounce);
	int			(*set_single_ended)(struct gpio_chip *chip,
						unsigned offset,
						enum single_ended_mode mode);

	int			(*to_irq)(struct gpio_chip *chip,
						unsigned offset);

	void			(*dbg_show)(struct seq_file *s,
						struct gpio_chip *chip);
	int			base;
	u16			ngpio;
	const char		*const *names;
	bool			can_sleep;
	bool			irq_not_threaded;

#if IS_ENABLED(CONFIG_GPIO_GENERIC)
	unsigned long (*read_reg)(void __iomem *reg);
	void (*write_reg)(void __iomem *reg, unsigned long data);
	unsigned long (*pin2mask)(struct gpio_chip *gc, unsigned int pin);
	void __iomem *reg_dat;
	void __iomem *reg_set;
	void __iomem *reg_clr;
	void __iomem *reg_dir;
	int bgpio_bits;
	spinlock_t bgpio_lock;
	unsigned long bgpio_data;
	unsigned long bgpio_dir;
#endif
```

gpio_chip 对/sys/class/gpio/gpiochip 有着对应关系

base代表基地址    cat /sys/class/gpio/gpiochipX/base

labe 代表标签

ngpio表示数量



//再看gpiod_get_index

```
struct gpio_desc *__must_check gpiod_get_index(struct device *dev,
					       const char *con_id,
					       unsigned int idx,
					       enum gpiod_flags flags)
{
	struct gpio_desc *desc = NULL;
	int status;
	enum gpio_lookup_flags lookupflags = 0;

	dev_dbg(dev, "GPIO lookup for consumer %s\n", con_id);

	if (dev) {
		/* Using device tree? */
		if (IS_ENABLED(CONFIG_OF) && dev->of_node) {
			dev_dbg(dev, "using device tree for GPIO lookup\n");
			desc = of_find_gpio(dev, con_id, idx, &lookupflags);
		} else if (ACPI_COMPANION(dev)) {
			dev_dbg(dev, "using ACPI for GPIO lookup\n");
			desc = acpi_find_gpio(dev, con_id, idx, flags, &lookupflags);
		}
	}

	/*
	 * Either we are not using DT or ACPI, or their lookup did not return
	 * a result. In that case, use platform lookup as a fallback.
	 */
	if (!desc || desc == ERR_PTR(-ENOENT)) {
		dev_dbg(dev, "using lookup tables for GPIO lookup\n");
		desc = gpiod_find(dev, con_id, idx, &lookupflags);
	}

	if (IS_ERR(desc)) {
		dev_dbg(dev, "lookup for GPIO %s failed\n", con_id);
		return desc;
	}

	status = gpiod_request(desc, con_id);
	if (status < 0)
		return ERR_PTR(status);
	
	//在这里调用gpiod_configure_flags 配置gpio口
	//如果成功就可以正常使用了，因此这里可以 作为修改点
	status = gpiod_configure_flags(desc, con_id, lookupflags, flags);
	if (status < 0) {
		dev_dbg(dev, "setup of GPIO %s failed\n", con_id);
		gpiod_put(desc);
		return ERR_PTR(status);
	}

	return desc;
}
EXPORT_SYMBOL_GPL(gpiod_get_index);

/**
```



再看gpiod_configure_flags

设置开漏与推流模式？ 初始化电平？ 

```
static int gpiod_configure_flags(struct gpio_desc *desc, const char *con_id,
		unsigned long lflags, enum gpiod_flags dflags)
{
	int status;

	if (lflags & GPIO_ACTIVE_LOW)
		set_bit(FLAG_ACTIVE_LOW, &desc->flags);
	if (lflags & GPIO_OPEN_DRAIN)
		set_bit(FLAG_OPEN_DRAIN, &desc->flags);
	if (lflags & GPIO_OPEN_SOURCE)
		set_bit(FLAG_OPEN_SOURCE, &desc->flags);

	/* No particular flag request, return here... */
	if (!(dflags & GPIOD_FLAGS_BIT_DIR_SET)) {
		pr_debug("no flags found for %s\n", con_id);
		return 0;
	}

	/* Process flags */
	if (dflags & GPIOD_FLAGS_BIT_DIR_OUT)
		status = gpiod_direction_output(desc,
					      dflags & GPIOD_FLAGS_BIT_DIR_VAL);
	else
		status = gpiod_direction_input(desc);

	return status;
}
```