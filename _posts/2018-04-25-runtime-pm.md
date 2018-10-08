---
title:  "PM-runtime经验总结"
date:   2018-04-25 16:23:00
categories: text
---

## 问题
adb reboot挂死的问题，最终发现与PM Runtime及Generic PM Domain有关，因此我做了一下经验总结，在这里分享给大家，希望对大家有所帮助。

## 直接原因
reboot导致系统挂死的直接原因是display在resume过程中，disp的power domain被关闭了，导致CPU访问DISP的寄存器挂死。

## 背景
display驱动的suspend 和 resume操作都是对同一个device进行`pm_runtime_put_sync()`和`pm_runtime_get_sync()`操作，display的power domain寄存器操作是放在了标准的Generic PM Domain框架下的，内部自带引用计数，会随着该domain下关联的设备调用pm_runtime_get/put来决定是否开启/关闭power domain。


## 错误观念
1. pm_runtime_get/put函数内部有spin lock拿锁保护，因此对同一device的get和put操作在多线程中是互斥的，当get正在执行的时候，put是没有机会得到执行的
2. pm_runtime_get/put调用的callback函数就是我们platform driver中自定义的pm callback函数
3. generic pm domain只会在它关联的device执行pm_runtime_get/put的时候才会判断是否操作power domain

## 观念纠正
1. pm_runtime_get/put时，内部的确会拿锁，但是执行到rpm_resume/suspend时，内部会在多种情况下释放锁，尤其是在执行callback之前，会直接释放spin lock，等callback彻底执行完后，再重新lock住。这就导致了如果我们driver的callback函数内调用了如msleep()，schedule()这类线程调度的函数，那么当前正在执行pm_runtime_get_sync时，pm_runtime_put_sync仍然是能够得到执行的，这就会有风险。
2. pm_runtime_get/put内部调用的callback函数是有优先顺序的，如果当前调用的device有Generic PM Domain，那么会优先调用pm_domain的pm callback函数，其次是bus的pm callback函数，如果以上都没有，那么最后才会调用driver的pm callback函数。因为我们大部分驱动都是platform driver，都是挂在platform bus下的，因此会优先执行platform bus的pm callback函数，即`pm_generic_runtime_resume/suspend`函数。而该函数内部实则直接调用driver的`runtime_resume/suspend`接口。如果该device存在pm domain，那么会优先调用`pm_genpd_runtime_suspend()`，再通过`pm_genpd_default_save_state()`最终调用到driver的pm callback函数。
3. pm domain的power_on/off不仅会在它关联的device执行`pm_runtime_get/put`时被调用，还会在platform driver的`probe/remove/shutdown`时机被调用，具体为：在probe之前调用power_on，如果probe失败则调用power_off；在remove、shutdown之后调用power_off。

## 结论
1. 要保证设备驱动的suspend/resume操作彻底互斥，在driver的pm callback函数（即`runtime_suspend/resume`接口）中，需要驱动维护人员自己加锁来保证操作的互斥性；
2. 可以使用`pm_runtime_force_suspend`函数，该接口内部会调用`__pm_runtime_barrier()`，能够确保在该suspend执行之前，之前被挂起的suspend/resume操作继续得到执行，实现同步功能

## 遗留问题
1. pm domain的开关操作如何在多线程中与suspend/resume同步起来是个问题。
目前pm domain内部是以引用计数为判断条件，计数0->1则开启domain，计数1->0则关闭domain。但如果一个domain对应两个device，如gsp & dpu，当出现一个线程刚执行dpu的resume把domian打开，正在访问寄存器的时候，另一个线程开始执行gsp的shutdown把power domain关掉了，那么系统就会挂死。pm_runtime_put/get能够确保pm domain内部的引用计数配对，但是platform的shutdown操作再次去操作domain似乎打破了引用计数的平衡，这个我暂时还没有想出解决办法，可能是我对pm domain理解的还不够深，或者使用方法上还有问题，如果大家有感兴趣的，可以研究一下如何解决该问题，我们一起探讨学习。

## kernel专家baolin意见
1. Runtime PM的callbacks在runtime PM中是可以确保互斥的(除了runtime idle可以并行)，除了锁保护，runtime PM还提供众多标志位来确保互斥。可以参考`Documentation/power/runtime_pm.txt`中描述：
 > The callbacks are mutually exclusive (e.g. it is **forbidden** to execute ->runtime_suspend() in parallel with ->runtime_resume() or with another
    instance of ->runtime_suspend() for the same device) with the exception that  ->runtime_suspend() or ->runtime_resume() can be executed in parallel with  ->runtime_idle() (although ->runtime_idle() will not be started while any  of the other callbacks is being executed for the same device).

2. 对于频繁操作的设备，不建议使用`pm_runtime_put_sync()`，应该尽量多的去使用`pm_runtime_put()`异步操作，这样可以防止频繁的runtime suspend/resume。
2. runtime**不会**保证`pm_runtime_get_sync()/pm_runtime_put_sync()`函数级的互斥。
2. 首先**一定要检查**runtime pm的返回值，出错时要做相应的错误处理。其次减1也没关系，即意味着你的设备就算runtime resume完成，也会通过异步`runtime idle`再次进入`runtime suspend`状态，因为现在runtime usage计数是0。


## 关键代码
### PM Runtime流程

{% highlight c %}

int pm_generic_runtime_suspend(struct device *dev)
{
    const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;
    int ret;
    
    /*最终调用的还是driver的runtime_suspend*/
    ret = pm && pm->runtime_suspend ? pm->runtime_suspend(dev) : 0;
    
    return ret;
}

int pm_generic_runtime_resume(struct device *dev)
{
    const struct dev_pm_ops *pm = dev->driver ? dev->driver->pm : NULL;
    int ret;
    /*最终调用的还是driver的runtime_resume*/
    ret = pm && pm->runtime_resume ? pm->runtime_resume(dev) : 0;
    
    return ret;
}

static const struct dev_pm_ops platform_dev_pm_ops = {
    .runtime_suspend = pm_generic_runtime_suspend,
    .runtime_resume = pm_generic_runtime_resume,
    USE_PLATFORM_PM_SLEEP_OPS
};

struct bus_type platform_bus_type = {
    .name = "platform",
    .dev_groups = platform_dev_groups,
    .match = platform_match,
    .uevent = platform_uevent,
    .pm  = &platform_dev_pm_ops,
};

static pm_callback_t __rpm_get_callback(struct device *dev, size_t cb_offset)
{
    pm_callback_t cb;
    const struct dev_pm_ops *ops;
    
    if (dev->pm_domain)
        ops = &dev->pm_domain->ops;
    else if (dev->type && dev->type->pm)
        ops = dev->type->pm;
    else if (dev->class && dev->class->pm)
        ops = dev->class->pm;
    else if (dev->bus && dev->bus->pm)
        ops = dev->bus->pm;
    else
        ops = NULL;
    
    if (ops)
        cb = *(pm_callback_t *)((void *)ops + cb_offset);
    else
        cb = NULL;
    
    if (!cb && dev->driver && dev->driver->pm)
        cb = *(pm_callback_t *)((void *)dev->driver->pm + cb_offset);
    
    return cb;
}

static int rpm_suspend(struct device *dev, int rpmflags)
__releases(&dev->power.lock) __acquires(&dev->power.lock)
{
    ......
    
    /*这里拿到的callback一般都是dev->bus->pm的suspend，即pm_generic_runtime_suspend*/
    callback = RPM_GET_CALLBACK(dev, runtime_suspend);
    
    retval = rpm_callback(callback, dev);
    
    ......
}

static int __rpm_callback(int (*cb)(struct device *), struct device *dev)
__releases(&dev->power.lock) __acquires(&dev->power.lock)
{
    int retval;
    
    /*注意：回调之前，先解锁！*/
    if (dev->power.irq_safe)
        spin_unlock(&dev->power.lock);
    else
        spin_unlock_irq(&dev->power.lock);
    
    retval = cb(dev);
    
    /*回调之后再上锁*/
    if (dev->power.irq_safe)
        spin_lock(&dev->power.lock);
    else
        spin_lock_irq(&dev->power.lock);
    
    return retval;
}
{% endhighlight %}
--------------
### Generic PM Domains
{% highlight c %}
static int platform_drv_probe(struct device *_dev)
{
    struct platform_driver *drv = to_platform_driver(_dev->driver);
    struct platform_device *dev = to_platform_device(_dev);
    int ret;
    
    ret = of_clk_set_defaults(_dev->of_node, false);
    if (ret < 0)
        return ret;
    
    ret = dev_pm_domain_attach(_dev, true);
    if (ret != -EPROBE_DEFER) {
        if (drv->probe) {
            ret = drv->probe(dev);
            if (ret)
                dev_pm_domain_detach(_dev, true);
        } else {
            /* don't fail if just dev_pm_domain_attach failed */
            ret = 0;
        }
    }
    
    if (drv->prevent_deferred_probe && ret == -EPROBE_DEFER) {
        dev_warn(_dev, "probe deferral not supported\n");
        ret = -ENXIO;
    }
    
    return ret;
}


static void platform_drv_shutdown(struct device *_dev)
{
    struct platform_driver *drv = to_platform_driver(_dev->driver);
    struct platform_device *dev = to_platform_device(_dev);
    
    if (drv->shutdown)
        drv->shutdown(dev);
    dev_pm_domain_detach(_dev, true);
}
{% endhighlight %}
当调用pm_runtime_put_sync()时：
{% highlight c %}
static pm_callback_t __rpm_get_callback(struct device *dev, size_t cb_offset)
{
    pm_callback_t cb;
    const struct dev_pm_ops *ops;
    
    /*优先使用pm_domain的runtime_suspend*/
    if (dev->pm_domain)
        ops = &dev->pm_domain->ops;
    else if (dev->type && dev->type->pm)
        ops = dev->type->pm;
    else if (dev->class && dev->class->pm)
        ops = dev->class->pm;
    else if (dev->bus && dev->bus->pm)
        ops = dev->bus->pm;
    else
        ops = NULL;
    
    if (ops)
        cb = *(pm_callback_t *)((void *)ops + cb_offset);
    else
        cb = NULL;
    
    if (!cb && dev->driver && dev->driver->pm)
        cb = *(pm_callback_t *)((void *)dev->driver->pm + cb_offset);
    
    return cb;
}

static int rpm_suspend(struct device *dev, int rpmflags)
__releases(&dev->power.lock) __acquires(&dev->power.lock)
{
    ......
    
    /*这里拿到的callback是power domain的runtime_suspend，即pm_genpd_runtime_suspend*/
    callback = RPM_GET_CALLBACK(dev, runtime_suspend);
    
    retval = rpm_callback(callback, dev);
    
    ......
}

static int pm_genpd_runtime_suspend(struct device *dev)
{
    ......
    
    ret = genpd_save_dev(genpd, dev);
    if (ret)
        return ret;
    
    ret = genpd_stop_dev(genpd, dev);
    if (ret) {
        genpd_restore_dev(genpd, dev);
        return ret;
    }
    ......
    mutex_lock(&genpd->lock);
    genpd_poweroff(genpd, false);
    mutex_unlock(&genpd->lock);
    
    return 0;
}

static int genpd_save_dev(struct generic_pm_domain *genpd, struct device *dev)
{
    return GENPD_DEV_CALLBACK(genpd, int, save_state, dev);
}

void pm_genpd_init(struct generic_pm_domain *genpd,
                  struct dev_power_governor *gov, bool is_off)
{
    ......
    genpd->domain.ops.runtime_suspend = pm_genpd_runtime_suspend;
    genpd->domain.ops.runtime_resume = pm_genpd_runtime_resume;
    
    ......
    genpd->dev_ops.save_state = pm_genpd_default_save_state;
    genpd->dev_ops.restore_state = pm_genpd_default_restore_state;
    
    ......
}

static int pm_genpd_default_save_state(struct device *dev)
{
    int (*cb)(struct device *__dev);
    
    if (dev->type && dev->type->pm)
        cb = dev->type->pm->runtime_suspend;
    else if (dev->class && dev->class->pm)
        cb = dev->class->pm->runtime_suspend;
    else if (dev->bus && dev->bus->pm)
        cb = dev->bus->pm->runtime_suspend;
    else
        cb = NULL;
    
    if (!cb && dev->driver && dev->driver->pm)
        cb = dev->driver->pm->runtime_suspend;
    
    /*最终还是调用到platfrom bus的 runtime_suspend，即pm_generic_runtime_suspend */
    return cb ? cb(dev) : 0;
}
{% endhighlight %}
