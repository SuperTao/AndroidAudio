一般是指某一个SoC平台，比如pxaxxx,s3cxxxx,omapxxx等等，与音频相关的通常包含该SoC中的时钟、DMA、I2S、PCM等等，

只要指定了SoC，那么我们可以认为它会有一个对应的Platform，它只与SoC相关，与Machine无关，这样我们就可以把Platform抽象出来，

使得同一款SoC不用做任何的改动，就可以用在不同的Machine中。实际上，把Platform认为是某个SoC更好理解。

### snd_soc_platform和snd_soc_platform_driver结构体定义

```
struct snd_soc_platform {
    struct device *dev;
    const struct snd_soc_platform_driver *driver;

    struct list_head list;

    struct snd_soc_component component;
};
```

```
/* SoC platform interface */
struct snd_soc_platform_driver {

    int (*probe)(struct snd_soc_platform *);
    int (*remove)(struct snd_soc_platform *);
    struct snd_soc_component_driver component_driver;

    /* pcm creation and destruction */
    int (*pcm_new)(struct snd_soc_pcm_runtime *);
    void (*pcm_free)(struct snd_pcm *);

    /*
     * For platform caused delay reporting.
     * Optional.
     */
    snd_pcm_sframes_t (*delay)(struct snd_pcm_substream *,
        struct snd_soc_dai *);

    /*
     * For platform-caused delay reporting, where the thread blocks waiting
     * for the delay amount to be determined.  Defining this will cause the
     * ASoC core to skip calling the delay callbacks for all components in
     * the runtime.
     * Optional.
     */
    snd_pcm_sframes_t (*delay_blk)(struct snd_pcm_substream *,
        struct snd_soc_dai *);

    /* platform stream pcm ops */
    const struct snd_pcm_ops *ops;

    /* platform stream compress ops */
    const struct snd_compr_ops *compr_ops;

    int (*bespoke_trigger)(struct snd_pcm_substream *, int);
};
```

### snd_soc_component和snd_soc_component_driver

```
struct snd_soc_component {
    const char *name;
    int id;
    const char *name_prefix;
    struct device *dev;
    struct snd_soc_card *card;                                // 指向snd_soc_card

    unsigned int active;

    unsigned int ignore_pmdown_time:1; /* pmdown_time is ignored at stop */
    unsigned int registered_as_component:1;

    struct list_head list;
    struct list_head list_aux; /* for auxiliary component of the card */

    struct snd_soc_dai_driver *dai_drv;                      // 指向snd_soc_dai_driver
    int num_dai;

    const struct snd_soc_component_driver *driver;           // 指向snd_soc_component_driver

    struct list_head dai_list;

    int (*read)(struct snd_soc_component *, unsigned int, unsigned int *);
    int (*write)(struct snd_soc_component *, unsigned int, unsigned int);

    struct regmap *regmap;
    int val_bytes;

    struct mutex io_mutex;

    /* attached dynamic objects */
    struct list_head dobj_list;

#ifdef CONFIG_DEBUG_FS
    struct dentry *debugfs_root;
#endif

    /*
    * DO NOT use any of the fields below in drivers, they are temporary and
    * are going to be removed again soon. If you use them in driver code the
    * driver will be marked as BROKEN when these fields are removed.
    */
	/* Don't use these, use snd_soc_component_get_dapm() */
    struct snd_soc_dapm_context dapm;

    const struct snd_kcontrol_new *controls;                   // snd_kcontrol_new
    unsigned int num_controls;
    const struct snd_soc_dapm_widget *dapm_widgets;
    unsigned int num_dapm_widgets;
    const struct snd_soc_dapm_route *dapm_routes;
    unsigned int num_dapm_routes;
    struct snd_soc_codec *codec;

    int (*probe)(struct snd_soc_component *);
    void (*remove)(struct snd_soc_component *);

    /* machine specific init */
    int (*init)(struct snd_soc_component *component);

#ifdef CONFIG_DEBUG_FS
    void (*init_debugfs)(struct snd_soc_component *component);
    const char *debugfs_prefix;
#endif
};	
```

```
/* component interface */
struct snd_soc_component_driver {
    const char *name;

    /* Default control and setup, added after probe() is run */
    const struct snd_kcontrol_new *controls;
    unsigned int num_controls;
    const struct snd_soc_dapm_widget *dapm_widgets;
    unsigned int num_dapm_widgets;
    const struct snd_soc_dapm_route *dapm_routes;
    unsigned int num_dapm_routes;

    int (*probe)(struct snd_soc_component *);
    void (*remove)(struct snd_soc_component *);

    /* DT */
    int (*of_xlate_dai_name)(struct snd_soc_component *component,
                 struct of_phandle_args *args,
                 const char **dai_name);
    void (*seq_notifier)(struct snd_soc_component *, enum snd_soc_dapm_type,
        int subseq);
    int (*stream_event)(struct snd_soc_component *, int event);

    /* probe ordering - for components with runtime dependencies */
    int probe_order;
    int remove_order;
};
```

代码中有很多snd_soc_register_platform的函数调用，是针对不同的类型，不同的应用场景注册的。

选其中一个进行举例

vendor/qcom/opensource/audio-kernel/asoc/msm-pcm-q6-v2.c

```
int __init msm_pcm_dsp_init(void)
{
    init_waitqueue_head(&the_locks.enable_wait);
    init_waitqueue_head(&the_locks.eos_wait);
    init_waitqueue_head(&the_locks.write_wait);
    init_waitqueue_head(&the_locks.read_wait);

    return platform_driver_register(&msm_pcm_driver);
}


static struct platform_driver msm_pcm_driver = {
    .driver = {
        .name = "msm-pcm-dsp",
        .owner = THIS_MODULE,
        .of_match_table = msm_pcm_dt_match,
    },   
    .probe = msm_pcm_probe,
    .remove = msm_pcm_remove,
};


static int msm_pcm_probe(struct platform_device *pdev)
{
    int rc;
    int id;
    struct msm_plat_data *pdata;
    const char *latency_level;

    rc = of_property_read_u32(pdev->dev.of_node,                // 获取DSP的ID
                "qcom,msm-pcm-dsp-id", &id);
    if (rc) {
        dev_err(&pdev->dev, "%s: qcom,msm-pcm-dsp-id missing in DT node\n",
                    __func__);
        return rc;
    }

    pdata = kzalloc(sizeof(struct msm_plat_data), GFP_KERNEL);
    if (!pdata)
        return -ENOMEM;

    if (of_property_read_bool(pdev->dev.of_node,                // 查找属性是否存在
                "qcom,msm-pcm-low-latency")) {

        pdata->perf_mode = LOW_LATENCY_PCM_MODE;
        rc = of_property_read_string(pdev->dev.of_node,         // 读取设备树低延时的等级
            "qcom,latency-level", &latency_level);
        if (!rc) {                                              // 设置延时的等级
		            if (!strcmp(latency_level, "ultra"))        // latency : ultra
                pdata->perf_mode = ULTRA_LOW_LATENCY_PCM_MODE;
            else if (!strcmp(latency_level, "ull-pp"))          // latency : ull-pp
                pdata->perf_mode =
                    ULL_POST_PROCESSING_PCM_MODE;
        }
    } else {
        pdata->perf_mode = LEGACY_PCM_MODE;                     // latency : regular
    }

    mutex_init(&pdata->lock);
    dev_set_drvdata(&pdev->dev, pdata);                         // 保存私有数据


    dev_dbg(&pdev->dev, "%s: dev name %s\n",
                __func__, dev_name(&pdev->dev));
    return snd_soc_register_platform(&pdev->dev,               // 注册声卡的platform
                   &msm_soc_platform);                         // 其中包括了声卡的操作函数。
}


static struct snd_soc_platform_driver msm_soc_platform = {       // platform 操作函数
    .ops        = &msm_pcm_ops,
    .pcm_new    = msm_asoc_pcm_new,
    .delay_blk      = msm_pcm_delay_blk,
};

static const struct snd_pcm_ops msm_pcm_ops = {
    .open           = msm_pcm_open,
    .copy       = msm_pcm_copy,
    .hw_params  = msm_pcm_hw_params,
    .close          = msm_pcm_close,
    .ioctl          = snd_pcm_lib_ioctl,
    .prepare        = msm_pcm_prepare,
    .trigger        = msm_pcm_trigger,
    .pointer        = msm_pcm_pointer,
    .mmap       = msm_pcm_mmap,
};
```

### snd_soc_register_platform

kernel/msm-4.9/sound/soc/soc-core.c

```
int snd_soc_register_platform(struct device *dev,
        const struct snd_soc_platform_driver *platform_drv)
{
    struct snd_soc_platform *platform;
    int ret; 

    dev_dbg(dev, "ASoC: platform register %s\n", dev_name(dev));
    
    platform = kzalloc(sizeof(struct snd_soc_platform), GFP_KERNEL);    // 申请内存
    if (platform == NULL)
        return -ENOMEM;

    ret = snd_soc_add_platform(dev, platform, platform_drv);
    if (ret)
        kfree(platform);

    return ret; 
}
EXPORT_SYMBOL_GPL(snd_soc_register_platform);

int snd_soc_add_platform(struct device *dev, struct snd_soc_platform *platform,
        const struct snd_soc_platform_driver *platform_drv)
{
    int ret;

    ret = snd_soc_component_initialize(&platform->component,
            &platform_drv->component_driver, dev);
    if (ret)
        return ret;

    platform->dev = dev;
    platform->driver = platform_drv;

    if (platform_drv->probe)
        platform->component.probe = snd_soc_platform_drv_probe;       // 在machine初始化时，创建声卡是调用
    if (platform_drv->remove)
        platform->component.remove = snd_soc_platform_drv_remove;

#ifdef CONFIG_DEBUG_FS
    platform->component.debugfs_prefix = "platform";
#endif

    mutex_lock(&client_mutex);
    snd_soc_component_add_unlocked(&platform->component);
    list_add(&platform->list, &platform_list);                  // 添加到platform_list链表中
    mutex_unlock(&client_mutex);

    dev_dbg(dev, "ASoC: Registered platform '%s'\n",
        platform->component.name);

    return 0;
}


// snd_soc_component_driver数据赋值给snd_soc_component
static int snd_soc_component_initialize(struct snd_soc_component *component,
    const struct snd_soc_component_driver *driver, struct device *dev)
{
    struct snd_soc_dapm_context *dapm;

    component->name = fmt_single_name(dev, &component->id);
    if (!component->name) {
        dev_err(dev, "ASoC: Failed to allocate name\n");
        return -ENOMEM;
    }

    component->dev = dev;
    component->driver = driver;
    component->probe = component->driver->probe;
    component->remove = component->driver->remove;

    dapm = &component->dapm;
    dapm->dev = dev;
    dapm->component = component;
    dapm->bias_level = SND_SOC_BIAS_OFF;
    dapm->idle_bias_off = true;
    if (driver->seq_notifier)
        dapm->seq_notifier = snd_soc_component_seq_notifier;
    if (driver->stream_event)
        dapm->stream_event = snd_soc_component_stream_event;

    component->controls = driver->controls;
    component->num_controls = driver->num_controls;
    component->dapm_widgets = driver->dapm_widgets;
    component->num_dapm_widgets = driver->num_dapm_widgets;
    component->dapm_routes = driver->dapm_routes;
	
    INIT_LIST_HEAD(&component->dai_list);
    mutex_init(&component->io_mutex);

    return 0;
}
```

### dai driver初始化

vendor/qcom/opensource/audio-kernel/asoc/msm-dai-q6-v2.c

```
int __init msm_dai_q6_init(void)
{
    int rc;

    rc = platform_driver_register(&msm_auxpcm_dev_driver);
    if (rc) {
        pr_err("%s: fail to register auxpcm dev driver", __func__);
        goto fail;
    }    

    rc = platform_driver_register(&msm_dai_q6);
    if (rc) {
        pr_err("%s: fail to register dai q6 driver", __func__);
        goto dai_q6_fail;
    }    

    rc = platform_driver_register(&msm_dai_q6_dev);
    if (rc) {
        pr_err("%s: fail to register dai q6 dev driver", __func__);
        goto dai_q6_dev_fail;
    }    

    rc = platform_driver_register(&msm_dai_q6_mi2s_driver);
    if (rc) {
        pr_err("%s: fail to register dai MI2S dev drv\n", __func__);
        goto dai_q6_mi2s_drv_fail;
    }    
	
```

```
static struct platform_driver msm_dai_q6_mi2s_driver = {
    .probe  = msm_dai_q6_mi2s_dev_probe,
    .remove  = msm_dai_q6_mi2s_dev_remove,
    .driver = {
        .name = "msm-dai-q6-mi2s",
        .owner = THIS_MODULE,
        .of_match_table = msm_dai_q6_mi2s_dev_dt_match,
    },
};
```

```
static int msm_dai_q6_mi2s_dev_probe(struct platform_device *pdev)
{
    struct msm_dai_q6_mi2s_dai_data *dai_data;
    const char *q6_mi2s_dev_id = "qcom,msm-dai-q6-mi2s-dev-id";
    u32 tx_line = 0;
    u32  rx_line = 0;
    u32 mi2s_intf = 0;
    struct msm_mi2s_pdata *mi2s_pdata;
    int rc;

    rc = of_property_read_u32(pdev->dev.of_node, q6_mi2s_dev_id,            // 读取设备树id
                  &mi2s_intf);
    if (rc) {
        dev_err(&pdev->dev,
            "%s: missing 0x%x in dt node\n", __func__, mi2s_intf);
        goto rtn;
    }

    dev_dbg(&pdev->dev, "dev name %s dev id 0x%x\n", dev_name(&pdev->dev),
        mi2s_intf);

    if ((mi2s_intf < MSM_MI2S_MIN || mi2s_intf > MSM_MI2S_MAX)            // 判断ID是否有效
        || (mi2s_intf >= ARRAY_SIZE(msm_dai_q6_mi2s_dai))) {
        dev_err(&pdev->dev,
            "%s: Invalid MI2S ID %u from Device Tree\n",
            __func__, mi2s_intf);
        rc = -ENXIO;
        goto rtn;
    }

    pdev->id = mi2s_intf;

    mi2s_pdata = kzalloc(sizeof(struct msm_mi2s_pdata), GFP_KERNEL);
    if (!mi2s_pdata) {
        rc = -ENOMEM;
        goto rtn;
    }

    rc = of_property_read_u32(pdev->dev.of_node, "qcom,msm-mi2s-rx-lines",  // 读取设备树mi2s接口rx line数量
                  &rx_line);
    if (rc) {
        dev_err(&pdev->dev, "%s: Rx line from DT file %s\n", __func__,
            "qcom,msm-mi2s-rx-lines");
        goto free_pdata;
    }

    rc = of_property_read_u32(pdev->dev.of_node, "qcom,msm-mi2s-tx-lines", // 读取设备树mi2s接口tx line数量
                  &tx_line);
    if (rc) {
        dev_err(&pdev->dev, "%s: Tx line from DT file %s\n", __func__,
            "qcom,msm-mi2s-tx-lines");
        goto free_pdata;
    }
    dev_dbg(&pdev->dev, "dev name %s Rx line 0x%x , Tx ine 0x%x\n",
        dev_name(&pdev->dev), rx_line, tx_line);
    mi2s_pdata->rx_sd_lines = rx_line;
    mi2s_pdata->tx_sd_lines = tx_line;
    mi2s_pdata->intf_id = mi2s_intf;

    dai_data = kzalloc(sizeof(struct msm_dai_q6_mi2s_dai_data),
               GFP_KERNEL);
    if (!dai_data) {
        rc = -ENOMEM;
        goto free_pdata;
    } else
        dev_set_drvdata(&pdev->dev, dai_data);             // 保存数据

    pdev->dev.platform_data = mi2s_pdata;

    rc = msm_dai_q6_mi2s_platform_data_validation(pdev,    // 判断数据的有效性
            &msm_dai_q6_mi2s_dai[mi2s_intf]);
    if (rc < 0)
        goto free_dai_data;

    rc = snd_soc_register_component(&pdev->dev, &msm_q6_mi2s_dai_component,     // 注册component, 这里一个dai当做一>
个component
    &msm_dai_q6_mi2s_dai[mi2s_intf], 1);
    if (rc < 0)
        goto err_register;
    return 0;

err_register:
    dev_err(&pdev->dev, "fail to msm_dai_q6_mi2s_dev_probe\n");
free_dai_data:
    kfree(dai_data);
free_pdata:
    kfree(mi2s_pdata);
rtn:
    return rc;
}
```

```
static struct snd_soc_dai_driver msm_dai_q6_mi2s_dai[] = {
    {
        .playback = {                                                // 声音播放
            .stream_name = "Primary MI2S Playback",                  // 流的名称
            .aif_name = "PRI_MI2S_RX",                               // 接口名称
            .rates = SNDRV_PCM_RATE_8000 | SNDRV_PCM_RATE_11025 |    // 采样率
                 SNDRV_PCM_RATE_16000 | SNDRV_PCM_RATE_22050 |
                 SNDRV_PCM_RATE_32000 | SNDRV_PCM_RATE_44100 |
                 SNDRV_PCM_RATE_48000 | SNDRV_PCM_RATE_96000 |
                 SNDRV_PCM_RATE_192000,
            .formats = (SNDRV_PCM_FMTBIT_S16_LE|                     // PCM格式，16/24/32位
                        SNDRV_PCM_FMTBIT_S24_LE |
                        SNDRV_PCM_FMTBIT_S24_3LE |
                        SNDRV_PCM_FMTBIT_S32_LE ),
            .rate_min =     8000,                                   // 最小采样率
            .rate_max =     192000,                                 // 最大采样率
        },
        .capture = {                                               // 录音
            .stream_name = "Primary MI2S Capture",
            .aif_name = "PRI_MI2S_TX",
            .rates = SNDRV_PCM_RATE_8000 | SNDRV_PCM_RATE_11025 |
                 SNDRV_PCM_RATE_16000 | SNDRV_PCM_RATE_22050 |
                 SNDRV_PCM_RATE_32000 | SNDRV_PCM_RATE_44100 |
                 SNDRV_PCM_RATE_48000 | SNDRV_PCM_RATE_96000 |
                 SNDRV_PCM_RATE_192000,
            .formats = SNDRV_PCM_FMTBIT_S16_LE,
            .rate_min =     8000,
            .rate_max =     192000,
        },
        .ops = &msm_dai_q6_mi2s_ops,                              // 操作函数
        .id = MSM_PRIM_MI2S,
        .probe = msm_dai_q6_dai_mi2s_probe,                       // probe函数，什么时候调用呢？

        .remove = msm_dai_q6_dai_mi2s_remove,
    },
    {
        .playback = {
            .stream_name = "Secondary MI2S Playback",
            .aif_name = "SEC_MI2S_RX",
            .rates = SNDRV_PCM_RATE_8000 | SNDRV_PCM_RATE_11025 |
                 SNDRV_PCM_RATE_16000 | SNDRV_PCM_RATE_22050 |
                 SNDRV_PCM_RATE_32000 | SNDRV_PCM_RATE_44100 |
                 SNDRV_PCM_RATE_48000 | SNDRV_PCM_RATE_96000 |
                 SNDRV_PCM_RATE_192000,
            .formats = SNDRV_PCM_FMTBIT_S16_LE,
            .rate_min =     8000,
            .rate_max =     192000,
        },
        .capture = {
            .stream_name = "Secondary MI2S Capture",
            .aif_name = "SEC_MI2S_TX",
            .rates = SNDRV_PCM_RATE_8000 | SNDRV_PCM_RATE_11025 |
                 SNDRV_PCM_RATE_16000 | SNDRV_PCM_RATE_22050 |
                 SNDRV_PCM_RATE_32000 | SNDRV_PCM_RATE_44100 |
                 SNDRV_PCM_RATE_48000 | SNDRV_PCM_RATE_96000 |
                 SNDRV_PCM_RATE_192000,
            .formats = SNDRV_PCM_FMTBIT_S16_LE,
            .rate_min =     8000,
            .rate_max =     192000,
        },
        .ops = &msm_dai_q6_mi2s_ops,
        .id = MSM_SEC_MI2S,
        .probe = msm_dai_q6_dai_mi2s_probe,
        .remove = msm_dai_q6_dai_mi2s_remove,
    },
```