在移动设备中，Codec的作用可以归结为4种，分别是：

对PCM等信号进行D/A转换，把数字的音频信号转换为模拟信号

对Mic、Linein或者其他输入源的模拟信号进行A/D转换，把模拟的声音信号转变CPU能够处理的数字信号

对音频通路进行控制，比如播放音乐，收听调频收音机，又或者接听电话时，音频信号在codec内的流通路线是不一样的

对音频信号做出相应的处理，例如音量控制，功率放大，EQ控制等等


kernel/msm-4.9/include/sound/soc.h

```
/* SoC Audio Codec device */
struct snd_soc_codec {
    struct device *dev;
    const struct snd_soc_codec_driver *driver;

    struct list_head list;
    struct list_head card_list;

    /* runtime */
    unsigned int cache_bypass:1; /* Suppress access to the cache */
    unsigned int suspended:1; /* Codec is in suspend PM state */
    unsigned int cache_init:1; /* codec cache has been initialized */

    /* codec IO */
    void *control_data; /* codec control (i2c/3wire) data */
    hw_write_t hw_write;
    void *reg_cache;

    /* component */
    struct snd_soc_component component;

#ifdef CONFIG_DEBUG_FS
    struct dentry *debugfs_reg;
#endif
};
```

```
/* codec driver */
struct snd_soc_codec_driver {

    /* driver ops */
    int (*probe)(struct snd_soc_codec *);
    int (*remove)(struct snd_soc_codec *);
    int (*suspend)(struct snd_soc_codec *);
    int (*resume)(struct snd_soc_codec *);
    struct snd_soc_component_driver component_driver;

    /* codec wide operations */
    int (*set_sysclk)(struct snd_soc_codec *codec,
              int clk_id, int source, unsigned int freq, int dir);
    int (*set_pll)(struct snd_soc_codec *codec, int pll_id, int source,
        unsigned int freq_in, unsigned int freq_out);

    /* codec IO */
    struct regmap *(*get_regmap)(struct device *);
    unsigned int (*read)(struct snd_soc_codec *, unsigned int);
    int (*write)(struct snd_soc_codec *, unsigned int, unsigned int);
    unsigned int reg_cache_size;
    short reg_cache_step;
    short reg_word_size;
    const void *reg_cache_default;

    /* codec bias level */
    int (*set_bias_level)(struct snd_soc_codec *,
                  enum snd_soc_bias_level level);
    bool idle_bias_off;
    bool suspend_bias_off;

    void (*seq_notifier)(struct snd_soc_dapm_context *,
                 enum snd_soc_dapm_type, int);

    bool ignore_pmdown_time;  /* Doesn't benefit from pmdown delay */
};
```

```
config/sdm450autoconf.h:54:#define CONFIG_SND_SOC_ANALOG_CDC 1
```

vendor/qcom/opensource/audio-kernel/asoc/codecs/sdm660_cdc$ vi msm-analog-cdc.c

```
static struct platform_driver msm_anlg_codec_driver = {
    .driver     = {  
        .owner          = THIS_MODULE,
        .name           = DRV_NAME,
        .of_match_table = of_match_ptr(sdm660_codec_of_match)
    },   
    .probe          = msm_anlg_cdc_probe,
    .remove         = msm_anlg_cdc_remove,
};
module_platform_driver(msm_anlg_codec_driver);
```

```
static int msm_anlg_cdc_probe(struct platform_device *pdev)
{
    int ret = 0;
    struct sdm660_cdc_priv *sdm660_cdc = NULL;
    struct sdm660_cdc_pdata *pdata;
    int adsp_state;
    const char *parent_dev = NULL;
    adsp_state = apr_get_subsys_state();
    if (adsp_state == APR_SUBSYS_DOWN ||
        !q6core_is_adsp_ready()) {
        dev_err(&pdev->dev, "Adsp is not loaded yet %d\n",
            adsp_state);
        return -EPROBE_DEFER;
    }
    device_init_wakeup(&pdev->dev, true);

    if (pdev->dev.of_node) {
        dev_dbg(&pdev->dev, "%s:Platform data from device tree\n",
            __func__);
        pdata = msm_anlg_cdc_populate_dt_pdata(&pdev->dev);                // 解析设备树
        pdev->dev.platform_data = pdata;
    } else {
        dev_dbg(&pdev->dev, "%s:Platform data from board file\n",
            __func__);
        pdata = pdev->dev.platform_data;
    }
    if (pdata == NULL) {
        dev_err(&pdev->dev, "%s:Platform data failed to populate\n",
            __func__);
        goto rtn;
    }
	    sdm660_cdc = devm_kzalloc(&pdev->dev, sizeof(struct sdm660_cdc_priv),
                     GFP_KERNEL);
    if (sdm660_cdc == NULL) {
        ret = -ENOMEM;
        goto rtn;
    }

    sdm660_cdc->dev = &pdev->dev;
    ret = msm_anlg_cdc_init_supplies(sdm660_cdc, pdata);
    if (ret) {
        dev_err(&pdev->dev, "%s: Fail to enable Codec supplies\n",
            __func__);
        goto rtn;
    }
    ret = msm_anlg_cdc_enable_static_supplies(sdm660_cdc, pdata);
    if (ret) {
        dev_err(&pdev->dev,
            "%s: Fail to enable Codec pre-reset supplies\n",
            __func__);
        goto rtn;
    }
    /* Allow supplies to be ready */
    usleep_range(5, 6);

    wcd9xxx_spmi_set_dev(pdev, 0);
    wcd9xxx_spmi_set_dev(pdev, 1);
    if (wcd9xxx_spmi_irq_init()) {
        dev_err(&pdev->dev,
            "%s: irq initialization failed\n", __func__);
    } else {
        dev_dbg(&pdev->dev,
            "%s: irq initialization passed\n", __func__);
			    dev_set_drvdata(&pdev->dev, sdm660_cdc);

    ret = snd_soc_register_codec(&pdev->dev,                            // 注册codec
                     &soc_codec_dev_sdm660_cdc,                        // snd_soc_codec_driver实例
                     msm_anlg_cdc_i2s_dai,                              // snd_soc_dai_driver实例
                     ARRAY_SIZE(msm_anlg_cdc_i2s_dai));
    if (ret) {
        dev_err(&pdev->dev,
            "%s:snd_soc_register_codec failed with error %d\n",
            __func__, ret);
        goto err_supplies;
    }
    BLOCKING_INIT_NOTIFIER_HEAD(&sdm660_cdc->notifier);
    BLOCKING_INIT_NOTIFIER_HEAD(&sdm660_cdc->notifier_mbhc);

    sdm660_cdc->dig_plat_data.handle = (void *) sdm660_cdc;
    sdm660_cdc->dig_plat_data.set_compander_mode = set_compander_mode;
    sdm660_cdc->dig_plat_data.update_clkdiv = update_clkdiv;
    sdm660_cdc->dig_plat_data.get_cdc_version = get_cdc_version;
    sdm660_cdc->dig_plat_data.register_notifier =
                    msm_anlg_cdc_dig_register_notifier;
    INIT_WORK(&sdm660_cdc->msm_anlg_add_child_devices_work,
          msm_anlg_add_child_devices);
    schedule_work(&sdm660_cdc->msm_anlg_add_child_devices_work);
    parent_dev = pdev->dev.parent->of_node->full_name;
    if (parent_dev) {
        snprintf(sdm660_cdc->pmic_analog, PMIC_ANOLOG_SIZE, "spmi0-0%s",
             parent_dev + strlen(parent_dev)-1);
        parent_dev = NULL;
    }
    return ret;
err_supplies:
    msm_anlg_cdc_disable_supplies(sdm660_cdc, pdata);
rtn:
    return ret;
}
```

```
static struct snd_soc_dai_driver msm_anlg_cdc_i2s_dai[] = {
    {
        .name = "msm_anlg_cdc_i2s_rx1",
        .id = AIF1_PB,
        .playback = {
            .stream_name = "PDM Playback",
            .rates = SDM660_CDC_RATES,
            .formats = SDM660_CDC_FORMATS,
            .rate_max = 192000,
            .rate_min = 8000,
            .channels_min = 1,
            .channels_max = 3,
        },
        .ops = &msm_anlg_cdc_dai_ops,
    },
    {
        .name = "msm_anlg_cdc_i2s_tx1",
        .id = AIF1_CAP,
        .capture = {
            .stream_name = "PDM Capture",
            .rates = SDM660_CDC_RATES,
            .formats = SDM660_CDC_FORMATS,
            .rate_max = 48000,
            .rate_min = 8000,
            .channels_min = 1,
            .channels_max = 4,
        },
        .ops = &msm_anlg_cdc_dai_ops,
    },
```

sound/soc/soc-core.c

```
int snd_soc_register_codec(struct device *dev,
               const struct snd_soc_codec_driver *codec_drv,
               struct snd_soc_dai_driver *dai_drv,
               int num_dai)
{
    struct snd_soc_dapm_context *dapm;
    struct snd_soc_codec *codec;
    struct snd_soc_dai *dai;
    int ret, i;

    dev_dbg(dev, "codec register %s\n", dev_name(dev));

    codec = kzalloc(sizeof(struct snd_soc_codec), GFP_KERNEL);      // 申请snd_soc_codec空间
    if (codec == NULL)
        return -ENOMEM;

    codec->component.codec = codec;

    ret = snd_soc_component_initialize(&codec->component,
            &codec_drv->component_driver, dev);
    if (ret)
        goto err_free;

    if (codec_drv->probe)                                         // 相关数据保存到component结构体中
        codec->component.probe = snd_soc_codec_drv_probe;
    if (codec_drv->remove)
        codec->component.remove = snd_soc_codec_drv_remove;
    if (codec_drv->write)
        codec->component.write = snd_soc_codec_drv_write;
    if (codec_drv->read)
        codec->component.read = snd_soc_codec_drv_read;
    codec->component.ignore_pmdown_time = codec_drv->ignore_pmdown_time;

    dapm = snd_soc_codec_get_dapm(codec);                        // 获取codec里面的dapm
    dapm->idle_bias_off = codec_drv->idle_bias_off;
    dapm->suspend_bias_off = codec_drv->suspend_bias_off;
    if (codec_drv->seq_notifier)
        dapm->seq_notifier = codec_drv->seq_notifier;
    if (codec_drv->set_bias_level)
        dapm->set_bias_level = snd_soc_codec_set_bias_level;
		    codec->dev = dev;
    codec->driver = codec_drv;
    codec->component.val_bytes = codec_drv->reg_word_size;

#ifdef CONFIG_DEBUG_FS
    codec->component.init_debugfs = soc_init_codec_debugfs;
    codec->component.debugfs_prefix = "codec";
#endif

    if (codec_drv->get_regmap)
        codec->component.regmap = codec_drv->get_regmap(dev);

    for (i = 0; i < num_dai; i++) {
        fixup_codec_formats(&dai_drv[i].playback);
        fixup_codec_formats(&dai_drv[i].capture);
    }

    ret = snd_soc_register_dais(&codec->component, dai_drv, num_dai, false);       // 对codec的dais进行注册
    if (ret < 0) {
        dev_err(dev, "ASoC: Failed to register DAIs: %d\n", ret);
        goto err_cleanup;
    }

    list_for_each_entry(dai, &codec->component.dai_list, list)
        dai->codec = codec;

    mutex_lock(&client_mutex);
    snd_
    list_add(&codec->list, &codec_list);                                            // 添加到链表上
    mutex_unlock(&client_mutex);

    dev_dbg(codec->dev, "ASoC: Registered codec '%s'\n",
        codec->component.name);
    return 0;

err_cleanup:
    snd_soc_component_cleanup(&codec->component);
err_free:
    kfree(codec);
    return ret;
}
EXPORT_SYMBOL_GPL(snd_soc_register_codec);
```

```
static int snd_soc_register_dais(struct snd_soc_component *component,
    struct snd_soc_dai_driver *dai_drv, size_t count,
    bool legacy_dai_naming)
{
    struct device *dev = component->dev;
    struct snd_soc_dai *dai;
    unsigned int i;
    int ret;

    dev_dbg(dev, "ASoC: dai register %s #%Zu\n", dev_name(dev), count);

    component->dai_drv = dai_drv;

    for (i = 0; i < count; i++) {

        dai = soc_add_dai(component, dai_drv + i,
                count == 1 && legacy_dai_naming);
        if (dai == NULL) {
            ret = -ENOMEM;
            goto err;
        }
    }

    return 0;

err:
    snd_soc_unregister_dais(component);

    return ret;
}
```