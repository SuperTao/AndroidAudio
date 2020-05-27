记录msm8952 audio的machine驱动初始化过程。

Machine 是指某一款机器，可以是某款设备，某款开发板，又或者是某款智能手机，由此可以看出Machine几乎是不可重用的，

每个Machine上的硬件实现可能都不一样，CPU不一样，Codec不一样，音频的输入、输出设备也不一样，Machine为CPU、Codec、输入输出设备

提供了一个载体，用于描述一块电路板, 它指明此块电路板上用的是哪个Platform和哪个Codec, 由电路板商负责编写此部分代码。

绑定platform driver和codec driver。ASoC的一切都从Machine驱动开始，包括声卡的注册，绑定Platform和Codec驱动等等。

machine驱动主要功能：

* 注册声卡

* 调用codec_dai的probe函数

* 调用platform的probe函数

在android Q上，audio与高通相关的代码移动到了vendor/qcom/opensource/audio-kernel目录。

SDM450平台的codec分为CPU内部的数字codec和外挂的模拟codec(PM8953)。本文只分析内部数字codec。

#### snd_soc_card和snd_card
---

```
struct snd_soc_card {
    const char *name;
    const char *long_name;
    const char *driver_name;
    struct device *dev;
    struct snd_card *snd_card;                                              // 指向snd_card结构体
    struct module *owner;

    struct mutex mutex;
    struct mutex dapm_mutex;
    struct mutex dapm_power_mutex;

    bool instantiated;

    int (*probe)(struct snd_soc_card *card);                              // probe函数，还不知道在哪里赋值
    int (*late_probe)(struct snd_soc_card *card);
    int (*remove)(struct snd_soc_card *card);

    /* the pre and post PM functions are used to do any PM work before and
     * after the codec and DAI's do any PM work. */
    int (*suspend_pre)(struct snd_soc_card *card);
    int (*suspend_post)(struct snd_soc_card *card);
    int (*resume_pre)(struct snd_soc_card *card);
    int (*resume_post)(struct snd_soc_card *card);

    /* callbacks */
    int (*set_bias_level)(struct snd_soc_card *,
                  struct snd_soc_dapm_context *dapm,
                  enum snd_soc_bias_level level);
    int (*set_bias_level_post)(struct snd_soc_card *,
                   struct snd_soc_dapm_context *dapm,
                   enum snd_soc_bias_level level);

    int (*add_dai_link)(struct snd_soc_card *,
                struct snd_soc_dai_link *link);
    void (*remove_dai_link)(struct snd_soc_card *,
                struct snd_soc_dai_link *link);

    long pmdown_time;
    /* CPU <--> Codec DAI links  */
    struct snd_soc_dai_link *dai_link;  /* predefined links only */      // 指向所有的dai链路数据结构体
    int num_links;  /* predefined links only */
    struct list_head dai_link_list; /* all links */
    int num_dai_links;

    struct list_head rtd_list;
    int num_rtd;

    /* optional codec specific configuration */
    struct snd_soc_codec_conf *codec_conf;
    int num_configs;

    /*
     * optional auxiliary devices such as amplifiers or codecs with DAI
     * link unused
     */
    struct snd_soc_aux_dev *aux_dev;
    int num_aux_devs;
    struct list_head aux_comp_list;

    const struct snd_kcontrol_new *controls;
    int num_controls;

    /*
     * Card-specific routes and widgets.
     * Note: of_dapm_xxx for Device Tree; Otherwise for driver build-in.
     */
    const struct snd_soc_dapm_widget *dapm_widgets;
    int num_dapm_widgets;
    const struct snd_soc_dapm_route *dapm_routes;
    int num_dapm_routes;
    const struct snd_soc_dapm_widget *of_dapm_widgets;
    int num_of_dapm_widgets;
    const struct snd_soc_dapm_route *of_dapm_routes;
    int num_of_dapm_routes;
    bool fully_routed;

    struct work_struct deferred_resume_work;

    /* lists of probed devices belonging to this card */
    struct list_head codec_dev_list;
	struct list_head widgets;
    struct list_head paths;
    struct list_head dapm_list;
    struct list_head dapm_dirty;

    /* attached dynamic objects */
    struct list_head dobj_list;

    /* Generic DAPM context for the card */
    struct snd_soc_dapm_context dapm;
    struct snd_soc_dapm_stats dapm_stats;
    struct snd_soc_dapm_update *update;

#ifdef CONFIG_DEBUG_FS
    struct dentry *debugfs_card_root;
    struct dentry *debugfs_pop_time;
#endif
    u32 pop_time;

    void *drvdata;
};
```

#### 设备树定义
---

驱动在初始化过程中，会用到设备树的信息。msm8953-audio.dtsi

```
    // 内部codec
    int_codec: sound {
        status = "okay";
        compatible = "qcom,msm8952-audio-codec";
        qcom,model = "msm8953-snd-card-mtp";            // 声卡名称
        reg = <0xc051000 0x4>,                          // 寄存器地址，已经寄存器长度
            <0xc051004 0x4>,
            <0xc055000 0x4>,
            <0xc052000 0x4>;
        reg-names = "csr_gp_io_mux_mic_ctl",             // 寄存器名称
            "csr_gp_io_mux_spkr_ctl",
            "csr_gp_io_lpaif_pri_pcm_pri_mode_muxsel",
            "csr_gp_io_mux_quin_ctl";

        qcom,msm-ext-pa = "primary";
        qcom,msm-mclk-freq = <9600000>;
        qcom,msm-mbhc-hphl-swh = <0>;
        qcom,msm-mbhc-gnd-swh = <0>;
        qcom,msm-hs-micbias-type = "internal";  // 选择内部电压作为MIC偏置电压
        qcom,msm-micbias1-ext-cap;              // 选择外部电容

        qcom,audio-routing =
                "RX_BIAS", "MCLK",
                "SPK_RX_BIAS", "MCLK",
                "INT_LDO_H", "MCLK",
                "RX_I2S_CLK", "MCLK",
                "TX_I2S_CLK", "MCLK",
                "MIC BIAS Internal1", "Handset Mic",
                "MIC BIAS External2", "Headset Mic",
                "MIC BIAS Internal3", "Secondary Mic",
                "AMIC1", "MIC BIAS Internal1",
                "AMIC2", "MIC BIAS Internal2",
				                "AMIC3", "MIC BIAS Internal3",
                "ADC1_IN", "ADC1_OUT",
                "ADC2_IN", "ADC2_OUT",
                "ADC3_IN", "ADC3_OUT",
                "PDM_IN_RX1", "PDM_OUT_RX1",
                "PDM_IN_RX2", "PDM_OUT_RX2",
                "PDM_IN_RX3", "PDM_OUT_RX3",
                "WSA_SPK OUT", "VDD_WSA_SWITCH",
                "SpkrMono WSA_IN", "WSA_SPK OUT";

        qcom,cdc-us-euro-gpios = <&tlmm 63 0>;
        qcom,cdc-us-eu-gpios = <&cdc_us_euro_sw>;
        qcom,cdc-comp-gpios = <&cdc_comp_gpios>;
        qcom,pri-mi2s-gpios = <&cdc_pri_mi2s_gpios>;
        /*qcom,quin-mi2s-gpios = <&cdc_quin_mi2s_gpios>;*/

        qcom,afe-rxtx-lb;

        asoc-platform = <&pcm0>, <&pcm1>, <&pcm2>, <&voip>, <&voice>,
                <&loopback>, <&compress>, <&hostless>,
                <&afe>, <&lsm>, <&routing>, <&pcm_noirq>;
        asoc-platform-names = "msm-pcm-dsp.0", "msm-pcm-dsp.1",
                "msm-pcm-dsp.2", "msm-voip-dsp",
                "msm-pcm-voice", "msm-pcm-loopback",
                "msm-compress-dsp", "msm-pcm-hostless",
                "msm-pcm-afe", "msm-lsm-client",
                "msm-pcm-routing", "msm-pcm-dsp-noirq";
        asoc-cpu = <&dai_pri_auxpcm>,
            <&dai_mi2s0>, <&dai_mi2s1>,
            <&dai_mi2s2>, <&dai_mi2s3>,
            <&dai_mi2s4>, <&dai_mi2s5>,
            <&sb_0_rx>, <&sb_0_tx>, <&sb_1_rx>, <&sb_1_tx>,
			            <&sb_3_rx>, <&sb_3_tx>, <&sb_4_rx>, <&sb_4_tx>,
            <&bt_sco_rx>, <&bt_sco_tx>,
            <&int_fm_rx>, <&int_fm_tx>,
            <&afe_pcm_rx>, <&afe_pcm_tx>,
            <&afe_proxy_rx>, <&afe_proxy_tx>,
            <&incall_record_rx>, <&incall_record_tx>,
            <&incall_music_rx>, <&incall_music_2_rx>,
            <&afe_loopback_tx>;

        asoc-cpu-names = "msm-dai-q6-auxpcm.1",
                "msm-dai-q6-mi2s.0", "msm-dai-q6-mi2s.1",
                "msm-dai-q6-mi2s.2", "msm-dai-q6-mi2s.3",
                "msm-dai-q6-mi2s.4", "msm-dai-q6-mi2s.6",
                "msm-dai-q6-dev.16384", "msmdai-q6-dev.16385",
                "msm-dai-q6-dev.16386", "msm-dai-q6-dev.16387",
                "msm-dai-q6-dev.16390", "msm-dai-q6-dev.16391",
                "msm-dai-q6-dev.16392", "msm-dai-q6-dev.16393",
                "msm-dai-q6-dev.12288", "msm-dai-q6-dev.12289",
                "msm-dai-q6-dev.12292", "msm-dai-q6-dev.12293",
                "msm-dai-q6-dev.224", "msm-dai-q6-dev.225",
                "msm-dai-q6-dev.241", "msm-dai-q6-dev.240",
                "msm-dai-q6-dev.32771", "msm-dai-q6-dev.32772",
                "msm-dai-q6-dev.32773", "msm-dai-q6-dev.32770",
                "msm-dai-q6-dev.24577";

        asoc-codec = <&stub_codec>, <&msm_digital_codec>,
                <&pmic_analog_codec>;
        asoc-codec-names = "msm-stub-codec.1", "msm-dig-codec",
                    "analog-codec";
        asoc-wsa-codec-names = "wsa881x-i2c-codec.2-000f";
        asoc-wsa-codec-prefixes = "SpkrMono";
        msm-vdd-wsa-switch-supply = <&pm8953_l5>;
		        qcom,msm-vdd-wsa-switch-voltage = <1800000>;
        qcom,msm-vdd-wsa-switch-current = <10000>;
    };
```

#### 确定machine文件
---

通过宏定义可以知道哪些文件进行了编译，如下：
```
config/sdm450autoconf.h:32:#define CONFIG_SND_SOC_SDM450 1

config/sdm450autoconf.h:33:#define CONFIG_SND_SOC_EXT_CODEC_SDM450 1
```

通过asoc/Kbuild
```
# for SDM450 internal codec sound card driver
ifdef CONFIG_SND_SOC_SDM450
    MACHINE_OBJS += msm8952.o
endif

# for SDM450 external codec sound card driver
ifdef CONFIG_SND_SOC_EXT_CODEC_SDM450
    MACHINE_EXT_OBJS += msm8952-slimbus.o
    MACHINE_EXT_OBJS += msm8952-dai-links.o
endif
```

#### machine初始化
---

通过Kbuild，确定数字codec初始化文件：asoc/msm8952.c
```
模块初始化
static int __init msm8952_machine_init(void)
{
    return platform_driver_register(&msm8952_asoc_machine_driver);
}

static const struct of_device_id msm8952_asoc_machine_of_match[]  = {  
    { .compatible = "qcom,msm8952-audio-codec", },
    {},  
};

static struct platform_driver msm8952_asoc_machine_driver = {
    .driver = {
        .name = DRV_NAME,
        .owner = THIS_MODULE,
        .pm = &snd_soc_pm_ops,
        .of_match_table = msm8952_asoc_machine_of_match,
    },   
    .probe = msm8952_asoc_machine_probe,		// 驱动匹配上，调用Probe函数。	
    .remove = msm8952_asoc_machine_remove,
};
```

```
static struct snd_soc_card bear_card = {
    /* snd_soc_card_msm8952 */
    .name       = "msm8952-snd-card",
    .dai_link   = msm8952_dai,
    .num_links  = ARRAY_SIZE(msm8952_dai),
};
```

通过匹配设备树节点，调用probe函数。

```
static int msm8952_asoc_machine_probe(struct platform_device *pdev)
{
    struct snd_soc_card *card;
    struct msm_asoc_mach_data *pdata = NULL;
    const char *hs_micbias_type = "qcom,msm-hs-micbias-type";
    const char *ext_pa = "qcom,msm-ext-pa";
    const char *mclk = "qcom,msm-mclk-freq";
    const char *wsa = "asoc-wsa-codec-names";
    const char *wsa_prefix = "asoc-wsa-codec-prefixes";
    const char *type = NULL;
    const char *ext_pa_str = NULL;
    const char *wsa_str = NULL;
    const char *wsa_prefix_str = NULL;
    const char *spk_ext_pa = "  ";
    int num_strings;
    int id, i, val;
    int ret = 0;
    struct resource *muxsel;
    char *temp_str = NULL;

    pdata = devm_kzalloc(&pdev->dev,
                sizeof(struct msm_asoc_mach_data),
                GFP_KERNEL);
    if (!pdata)
        return -ENOMEM;
    
    muxsel = platform_get_resource_byname(pdev, IORESOURCE_MEM,	   // 读取设备树中寄存器
            "csr_gp_io_mux_mic_ctl");
    if (!muxsel) {
        dev_err(&pdev->dev, "MUX addr invalid for MI2S\n");
        ret = -ENODEV;
        goto err1;
		    
    pdata->vaddr_gpio_mux_mic_ctl =	                           // 将物理地址映射成虚拟地址
        ioremap(muxsel->start, resource_size(muxsel));
    if (pdata->vaddr_gpio_mux_mic_ctl == NULL) {
        pr_err("%s ioremap failure for muxsel virt addr\n",
                __func__);
        ret = -ENOMEM;
        goto err1;
    }

    muxsel = platform_get_resource_byname(pdev, IORESOURCE_MEM,
            "csr_gp_io_mux_spkr_ctl");
    if (!muxsel) {
        dev_err(&pdev->dev, "MUX addr invalid for MI2S\n");
        ret = -ENODEV;
        goto err;
    }
    pdata->vaddr_gpio_mux_spkr_ctl =
        ioremap(muxsel->start, resource_size(muxsel));
    if (pdata->vaddr_gpio_mux_spkr_ctl == NULL) {
        pr_err("%s ioremap failure for muxsel virt addr\n",
                __func__);
        ret = -ENOMEM;
        goto err;
    }

    muxsel = platform_get_resource_byname(pdev, IORESOURCE_MEM,
            "csr_gp_io_lpaif_pri_pcm_pri_mode_muxsel");
			    if (!muxsel) {
        dev_err(&pdev->dev, "MUX addr invalid for MI2S\n");
        ret = -ENODEV;
        goto err;
    }
    pdata->vaddr_gpio_mux_pcm_ctl =
        ioremap(muxsel->start, resource_size(muxsel));
    if (pdata->vaddr_gpio_mux_pcm_ctl == NULL) {
        pr_err("%s ioremap failure for muxsel virt addr\n",
                __func__);
        ret = -ENOMEM;
        goto err;
    }

    muxsel = platform_get_resource_byname(pdev, IORESOURCE_MEM,
            "csr_gp_io_mux_quin_ctl");
    if (!muxsel) {
        dev_dbg(&pdev->dev, "MUX addr invalid for MI2S\n");
        goto parse_mclk_freq;
    }
    pdata->vaddr_gpio_mux_quin_ctl =
        ioremap(muxsel->start, resource_size(muxsel));
    if (pdata->vaddr_gpio_mux_quin_ctl == NULL) {
        pr_err("%s ioremap failure for muxsel virt addr\n",
                __func__);
        ret = -ENOMEM;
        goto err;
    }
parse_mclk_freq:
    
    ret = of_property_read_u32(pdev->dev.of_node, mclk, &id);           // 获取设备树时钟
    if (ret) {
	        dev_err(&pdev->dev,
                "%s: missing %s in dt node\n", __func__, mclk);
        id = DEFAULT_MCLK_RATE;
    }
    pdata->mclk_freq = id;

    /*reading the gpio configurations from dtsi file*/
    
    num_strings = of_property_count_strings(pdev->dev.of_node,          // wsa为空, 所以不做处理
            wsa);
    if (num_strings > 0) {
        if (wsa881x_get_probing_count() < 2) {
            ret = -EPROBE_DEFER;
            goto err;
        } else if (wsa881x_get_presence_count() == num_strings) {
            bear_card.aux_dev = msm8952_aux_dev;
            bear_card.num_aux_devs = num_strings;
            bear_card.codec_conf = msm8952_codec_conf;
            bear_card.num_configs = num_strings;

            for (i = 0; i < num_strings; i++) {
                ret = of_property_read_string_index(
                        pdev->dev.of_node, wsa,
                        i, &wsa_str);
                if (ret) {
                    dev_err(&pdev->dev,
                        "%s:of read string %s i %d error %d\n",
                        __func__, wsa, i, ret);
                    goto err;
                }
                temp_str = kstrdup(wsa_str, GFP_KERNEL);
                if (!temp_str) {
				                    ret = -ENOMEM;
                    goto err;
                }
                msm8952_aux_dev[i].codec_name = temp_str;
                temp_str = NULL;

                temp_str = kstrdup(wsa_str, GFP_KERNEL);
                if (!temp_str) {
                    ret = -ENOMEM;
                    goto err;
                }
                msm8952_codec_conf[i].dev_name = temp_str;
                temp_str = NULL;

                ret = of_property_read_string_index(
                        pdev->dev.of_node, wsa_prefix,
                        i, &wsa_prefix_str);
                if (ret) {
                    dev_err(&pdev->dev,
                        "%s:of read string %s i %d error %d\n",
                        __func__, wsa_prefix, i, ret);
                    goto err;
                }
                temp_str = kstrdup(wsa_prefix_str, GFP_KERNEL);
                if (!temp_str) {
                    ret = -ENOMEM;
                    goto err;
                }
                msm8952_codec_conf[i].name_prefix = temp_str;
                temp_str = NULL;
            }
			            ret = msm8952_init_wsa_switch_supply(pdev, pdata);
            if (ret < 0) {
                pr_err("%s: failed to init wsa_switch vdd supply %d\n",
                        __func__, ret);
                goto err;
            }
            wsa881x_set_mclk_callback(msm8952_enable_wsa_mclk);
            /* update the internal speaker boost usage */
            msm_anlg_cdc_update_int_spk_boost(false);
        }
    }
    
    card = msm8952_populate_sndcard_dailinks(&pdev->dev);                // 处理dai_links相关操作，后面分析
    dev_dbg(&pdev->dev, "default codec configured\n");
    num_strings = of_property_count_strings(pdev->dev.of_node,          // qcom.msm-ext-pa，外部功率放大器,external power amplifier
            ext_pa);
    if (num_strings < 0) {
        dev_err(&pdev->dev,
                "%s: missing %s in dt node or length is incorrect\n",
                __func__, ext_pa);
        goto err;
    }
    
    for (i = 0; i < num_strings; i++) {
        ret = of_property_read_string_index(pdev->dev.of_node,        // 获取外部功放的信息
                ext_pa, i, &ext_pa_str);
        if (ret) {
            dev_err(&pdev->dev, "%s:of read string %s i %d error %d\n",
                    __func__, ext_pa, i, ret);
            goto err;
        }
		        dev_err(&pdev->dev, "%s:of read string %s i %d ret %d\n",
                    __func__, ext_pa, i, ret);
        if (!strcmp(ext_pa_str, "primary"))
            pdata->ext_pa = (pdata->ext_pa | PRI_MI2S_ID);
        else if (!strcmp(ext_pa_str, "secondary"))
            pdata->ext_pa = (pdata->ext_pa | SEC_MI2S_ID);
        else if (!strcmp(ext_pa_str, "tertiary"))
            pdata->ext_pa = (pdata->ext_pa | TER_MI2S_ID);
        else if (!strcmp(ext_pa_str, "quaternary"))
            pdata->ext_pa = (pdata->ext_pa | QUAT_MI2S_ID);
        else if (!strcmp(ext_pa_str, "quinary"))
            pdata->ext_pa = (pdata->ext_pa | QUIN_MI2S_ID);
    }
    pr_debug("%s: ext_pa = %d\n", __func__, pdata->ext_pa);
    
    pdata->spk_ext_pa_gpio = of_get_named_gpio(pdev->dev.of_node,          // 外部喇叭功放为空
                            spk_ext_pa, 0);
    if (pdata->spk_ext_pa_gpio < 0) {
        dev_err(&pdev->dev, "%s: missing %s in dt node\n",
            __func__, spk_ext_pa);
    }

    pdata->spk_ext_pa_gpio_p = of_parse_phandle(pdev->dev.of_node,
                            spk_ext_pa, 0);
    
    ret = is_us_eu_switch_gpio_support(pdev, pdata);                    // 查看美国标准，欧洲标准切换的gpio是否支持
    if (ret < 0) {
        pr_err("%s: failed to is_us_eu_switch_gpio_support %d\n",
                __func__, ret);
        goto err;
    }
    
	ret = is_ext_spk_gpio_support(pdev, pdata);	                        // 不支持外部喇叭
    if (ret < 0)
        pr_err("%s:  doesn't support external speaker pa\n",
                __func__);
    
    enable_aw8738 = of_property_read_bool(pdev->dev.of_node , "action,enable-aw8738");        // 没有外部功放
    if(enable_aw8738)
    {
        pr_err("msm8952_asoc_machine_probe enable_aw8738\n");
        gpio_request(pdata->spk_ext_pa_gpio, "AW8738_EN");
        gpio_direction_output(pdata->spk_ext_pa_gpio,0);
    }
    
    enable_aw87519 = of_property_read_bool(pdev->dev.of_node , "action,enable-aw87519");      // 不支持外部功放
    pr_info("%s: enable_aw87519 = %d\n", __func__, enable_aw87519);
    
    pdata->comp_gpio_p = of_parse_phandle(pdev->dev.of_node,               // 获取cdc-comp-gpios
                    "qcom,cdc-comp-gpios", 0);

    pdata->mi2s_gpio_p[PRIM_MI2S] = of_parse_phandle(pdev->dev.of_node,
                    "qcom,pri-mi2s-gpios", 0);
    pdata->mi2s_gpio_p[SEC_MI2S] = of_parse_phandle(pdev->dev.of_node,
                    "qcom,sec-mi2s-gpios", 0);
    pdata->mi2s_gpio_p[TERT_MI2S] = of_parse_phandle(pdev->dev.of_node,
                    "qcom,tert-mi2s-gpios", 0);
    pdata->mi2s_gpio_p[QUAT_MI2S] = of_parse_phandle(pdev->dev.of_node,
                    "qcom,quat-mi2s-gpios", 0);
    pdata->mi2s_gpio_p[QUIN_MI2S] = of_parse_phandle(pdev->dev.of_node,
                    "qcom,quin-mi2s-gpios", 0);
    
    ret = of_property_read_string(pdev->dev.of_node,                      // 获取设备树MIC偏置电压类型
        hs_micbias_type, &type);
		    if (ret) {
        dev_err(&pdev->dev, "%s: missing %s in dt node\n",
            __func__, hs_micbias_type);
        goto err;
    }
    if (!strcmp(type, "external")) {
        dev_err(&pdev->dev, "Headset is using external micbias\n");
        mbhc_cfg.hs_ext_micbias = true;
    } else {
        
        dev_err(&pdev->dev, "Headset is using internal micbias\n");       // 使用内部偏置电压
        mbhc_cfg.hs_ext_micbias = false;
    }
    
    ret = of_property_read_u32(pdev->dev.of_node,                  // 获取AFE时钟
                  "qcom,msm-afe-clk-ver", &val);
    if (ret)
        pdata->afe_clk_ver = AFE_CLK_VERSION_V2;
    else
        pdata->afe_clk_ver = val;
    /* initialize the mclk */
    
    pdata->digital_cdc_clk.i2s_cfg_minor_version =                 // 初始化数字codec
                    AFE_API_VERSION_I2S_CONFIG;
    pdata->digital_cdc_clk.clk_val = pdata->mclk_freq;
    pdata->digital_cdc_clk.clk_root = 5;
    pdata->digital_cdc_clk.reserved = 0;
    /* initialize the digital codec core clk */
    pdata->digital_cdc_core_clk.clk_set_minor_version =
            AFE_API_VERSION_I2S_CONFIG;
    pdata->digital_cdc_core_clk.clk_id =
            Q6AFE_LPASS_CLK_ID_INTERNAL_DIGITAL_CODEC_CORE;
    pdata->digital_cdc_core_clk.clk_freq_in_hz =
            pdata->mclk_freq;
    pdata->digital_cdc_core_clk.clk_attri =
            Q6AFE_LPASS_CLK_ATTRIBUTE_COUPLE_NO;
    pdata->digital_cdc_core_clk.clk_root =
            Q6AFE_LPASS_CLK_ROOT_DEFAULT;
    pdata->digital_cdc_core_clk.enable = 1;

    /* Initialize loopback mode to false */
    pdata->lb_mode = false;
    msm8952_dt_parse_cap_info(pdev, pdata);

    card->dev = &pdev->dev;
    platform_set_drvdata(pdev, card);                           // 数据保存到pdev中
    snd_soc_card_set_drvdata(card, pdata);                     // 数据保存到card中
    ret = snd_soc_of_parse_card_name(card, "qcom,model");      // 获取设备树声卡名称
    if (ret)
        goto err;
    /* initialize timer */
    INIT_DELAYED_WORK(&pdata->disable_int_mclk0_work, msm8952_disable_mclk);
    mutex_init(&pdata->cdc_int_mclk0_mutex);
    atomic_set(&pdata->int_mclk0_rsc_ref, 0);
    if (card->aux_dev) {
        mutex_init(&pdata->wsa_mclk_mutex);
        atomic_set(&pdata->wsa_int_mclk0_rsc_ref, 0);
    }
    atomic_set(&pdata->int_mclk0_enabled, false);
	    atomic_set(&quat_mi2s_clk_ref, 0);
    atomic_set(&quin_mi2s_clk_ref, 0);
    atomic_set(&auxpcm_mi2s_clk_ref, 0);
    ret = snd_soc_of_parse_audio_routing(card,                  // 获取设备树音频通路的信息
            "qcom,audio-routing");
    if (ret)
        goto err;

    ret = msm8952_populate_dai_link_component_of_node(card);
    if (ret) {
        ret = -EPROBE_DEFER;
        goto err;
    }
    
    ret = devm_snd_soc_register_card(&pdev->dev, card);        // 注册声卡
    if (ret) {
        dev_err(&pdev->dev, "snd_soc_register_card failed (%d)\n",
            ret);
        goto err;
    }

    return 0;
err:
    if (pdata->vaddr_gpio_mux_spkr_ctl)
        iounmap(pdata->vaddr_gpio_mux_spkr_ctl);
    if (pdata->vaddr_gpio_mux_mic_ctl)
        iounmap(pdata->vaddr_gpio_mux_mic_ctl);
    if (pdata->vaddr_gpio_mux_pcm_ctl)
        iounmap(pdata->vaddr_gpio_mux_pcm_ctl);
    if (pdata->vaddr_gpio_mux_quin_ctl)
        iounmap(pdata->vaddr_gpio_mux_quin_ctl);
		    if (bear_card.num_aux_devs > 0) {
        for (i = 0; i < bear_card.num_aux_devs; i++) {
            kfree(msm8952_aux_dev[i].codec_name);
            kfree(msm8952_codec_conf[i].dev_name);
            kfree(msm8952_codec_conf[i].name_prefix);
        }
    }
err1:
    devm_kfree(&pdev->dev, pdata);
    return ret;
}
```

#### snd_soc_dai_link处理
---

查看对dai_link的处理，保存所有的dai链路地址到snd_soc_card结构体中。

```
static struct snd_soc_card *msm8952_populate_sndcard_dailinks(
                        struct device *dev)
{
    struct snd_soc_card *card = &bear_card;
    struct snd_soc_dai_link *dailink;
    int len1;

    card->name = dev_name(dev);
    len1 = ARRAY_SIZE(msm8952_dai);     								// 获取数组个数
    
    memcpy(msm8952_dai_links, msm8952_dai, sizeof(msm8952_dai));		// 将msm8952_dai数组复制到msm8952_dai_links数组
    dailink = msm8952_dai_links;
    
    if (of_property_read_bool(dev->of_node,	                           // 查看是否存在hdmi的dai_link, 本产品设备树中没有, 使用默认的
                "qcom,hdmi-dba-codec-rx")) {
        dev_dbg(dev, "%s(): hdmi audio support present\n",
                __func__);
        memcpy(dailink + len1, msm8952_hdmi_dba_dai_link,
                sizeof(msm8952_hdmi_dba_dai_link));
        len1 += ARRAY_SIZE(msm8952_hdmi_dba_dai_link);
    } else {
        dev_dbg(dev, "%s(): No hdmi dba present, add quin dai\n",
                __func__);
        memcpy(dailink + len1, msm8952_quin_dai_link,
                sizeof(msm8952_quin_dai_link));
        len1 += ARRAY_SIZE(msm8952_quin_dai_link);
    }
    
    if (of_property_read_bool(dev->of_node,							// 查看是否存在qcom,split-a2dp的定义,也没有
                "qcom,split-a2dp")) {
        dev_dbg(dev, "%s(): split a2dp support present\n",
		                __func__);
        memcpy(dailink + len1, msm8952_split_a2dp_dai_link,
                sizeof(msm8952_split_a2dp_dai_link));
        len1 += ARRAY_SIZE(msm8952_split_a2dp_dai_link);
    }
    
    card->dai_link = dailink;										// dailink保存到snd_soc_card结构体中
    // 长度
    card->num_links = len1;
    return card;
}
```

重点是其中的msm8952_dai数组，里面定义了前端，后端的各种DAI链路。

CPU <---> 前端DAI接口 <---> 中间DSP <---> 后端DAI接口 <--->数字CODEC <--->模拟codec <---> 音频设备

```
// CPU <---> 前端DAI接口 <---> 中间DSP <---> 后端DAI接口 <--->数字CODEC <--->模拟codec <---> 音频设备
/* Digital audio interface glue - connects codec <---> CPU */
static struct snd_soc_dai_link msm8952_dai[] = {
/*********************  前端  ***************************/	
    /* FrontEnd DAI Links */
    {/* hw:x,0 */
        .name = "MSM8952 Media1",
        .stream_name = "MultiMedia1",
        .cpu_dai_name   = "MultiMedia1",
        .platform_name  = "msm-pcm-dsp.0",
        .dynamic = 1,
        .async_ops = ASYNC_DPCM_SND_SOC_PREPARE,
        .dpcm_playback = 1,
        .dpcm_capture = 1,
        .trigger = {SND_SOC_DPCM_TRIGGER_POST,
            SND_SOC_DPCM_TRIGGER_POST},
        .codec_dai_name = "snd-soc-dummy-dai",
        .codec_name = "snd-soc-dummy",
        .ignore_suspend = 1,
        /* this dainlink has playback support */
        .ignore_pmdown_time = 1,
        .id = MSM_FRONTEND_DAI_MULTIMEDIA1
    },
    {/* hw:x,1 */
        .name = "MSM8952 Media2",
        .stream_name = "MultiMedia2",
        .cpu_dai_name   = "MultiMedia2",
        .platform_name  = "msm-pcm-dsp.0",
        .dynamic = 1,
        .dpcm_playback = 1,
        .dpcm_capture = 1,
        .codec_dai_name = "snd-soc-dummy-dai",
        .codec_name = "snd-soc-dummy",
		        .trigger = {SND_SOC_DPCM_TRIGGER_POST,
            SND_SOC_DPCM_TRIGGER_POST},
        .ignore_suspend = 1,
        /* this dainlink has playback support */
        .ignore_pmdown_time = 1,
        .id = MSM_FRONTEND_DAI_MULTIMEDIA2,
    },
    {/* hw:x,2 */
        .name = "Circuit-Switch Voice",
        .stream_name = "CS-Voice",
        .cpu_dai_name   = "VoiceMMode1",
        .platform_name  = "msm-pcm-voice",
        .dynamic = 1,
        .dpcm_playback = 1,
        .dpcm_capture = 1,
        .trigger = {SND_SOC_DPCM_TRIGGER_POST,
            SND_SOC_DPCM_TRIGGER_POST},
        .no_host_mode = SND_SOC_DAI_LINK_NO_HOST,
        .ignore_suspend = 1,
        /* this dainlink has playback support */
        .ignore_pmdown_time = 1,
        .id = MSM_FRONTEND_DAI_CS_VOICE,
        .codec_dai_name = "snd-soc-dummy-dai",
        .codec_name = "snd-soc-dummy",
    },
	
/*********************  后端  ***************************/	
	    /* Backend I2S DAI Links */
    {
        .name = LPASS_BE_PRI_MI2S_RX,
        .stream_name = "Primary MI2S Playback",
        .cpu_dai_name = "msm-dai-q6-mi2s.0",
        .platform_name = "msm-pcm-routing",
        .codecs = dlc_rx1,
        .num_codecs = CODECS_MAX,                                          // 2个codec, 一个DIG_CDC, 一个ANA_CDC
        .no_pcm = 1,
        .dpcm_playback = 1,
        .async_ops = ASYNC_DPCM_SND_SOC_PREPARE |
            ASYNC_DPCM_SND_SOC_HW_PARAMS,
        .id = MSM_BACKEND_DAI_PRI_MI2S_RX,
        .init = &msm_audrx_init,
        .be_hw_params_fixup = msm_mi2s_rx_be_hw_params_fixup,
        .ops = &msm8952_mi2s_be_ops,
        .ignore_suspend = 1,
    },
    {
        .name = LPASS_BE_SEC_MI2S_RX,
        .stream_name = "Secondary MI2S Playback",
        .cpu_dai_name = "msm-dai-q6-mi2s.1",
        .platform_name = "msm-pcm-routing",
        .codec_name = "msm-stub-codec.1",
        .codec_dai_name = "msm-stub-rx",
        .no_pcm = 1,
        .dpcm_playback = 1,
        .id = MSM_BACKEND_DAI_SECONDARY_MI2S_RX,
        .be_hw_params_fixup = msm_mi2s_rx_be_hw_params_fixup,
        .ops = &msm8952_sec_mi2s_be_ops,
        .ignore_suspend = 1,
    },
	
```

#### 声卡注册
---

probe函数的其他内容看注释，跟代码就行了。最后是注册声卡。

kernel/msm-4.9/sound/soc/soc-devres.c

```
int devm_snd_soc_register_card(struct device *dev, struct snd_soc_card *card)
{
    struct snd_soc_card **ptr;
    int ret;

    ptr = devres_alloc(devm_card_release, sizeof(*ptr), GFP_KERNEL);                 // 申请空间
    if (!ptr)
        return -ENOMEM;

    ret = snd_soc_register_card(card);                                              // 注册声卡
    if (ret == 0) {
        *ptr = card;
        devres_add(dev, ptr);
    } else {
        devres_free(ptr);
    }

    return ret;
}
EXPORT_SYMBOL_GPL(devm_snd_soc_register_card);
```

kernel/msm-4.9/sound/soc/soc-core.c

```
int snd_soc_register_card(struct snd_soc_card *card)
{
    int i, ret;
    struct snd_soc_pcm_runtime *rtd;

    if (!card->name || !card->dev)
        return -EINVAL;

    for (i = 0; i < card->num_links; i++) {                  // num_links就是前面列出的msm8952_populate_sndcard_dailinks函数里面初始化的，数组元素
很多
        struct snd_soc_dai_link *link = &card->dai_link[i];

        ret = soc_init_dai_link(card, link);                // 每一个snd_soc_dai_link初始化 
        if (ret) {
            dev_err(card->dev, "ASoC: failed to init link %s\n",
                link->name);
            return ret;
        }
    }

    dev_set_drvdata(card->dev, card);

    snd_soc_initialize_card_lists(card);                   // 初始化链表头

    INIT_LIST_HEAD(&card->dai_link_list);
    card->num_dai_links = 0; 

    INIT_LIST_HEAD(&card->rtd_list);
    card->num_rtd = 0; 

    INIT_LIST_HEAD(&card->dapm_dirty);
    INIT_LIST_HEAD(&card->dobj_list);
    card->instantiated = 0; 
    mutex_init(&card->mutex);
    mutex_init(&card->dapm_mutex);
    mutex_init(&card->dapm_power_mutex);

    ret = snd_soc_instantiate_card(card);                 // 声卡创建
    if (ret != 0)
        return ret;
		
    arch_setup_dma_ops(card->dev, 0, 0, NULL, 0);

    /* deactivate pins to sleep state */
    list_for_each_entry(rtd, &card->rtd_list, list)  {
        struct snd_soc_dai *cpu_dai = rtd->cpu_dai;
        int j;

        for (j = 0; j < rtd->num_codecs; j++) {
            struct snd_soc_dai *codec_dai = rtd->codec_dais[j];
            if (!codec_dai->active)
                pinctrl_pm_select_sleep_state(codec_dai->dev);
        }

        if (!cpu_dai->active)
            pinctrl_pm_select_sleep_state(cpu_dai->dev);
    }

    return ret;
}
EXPORT_SYMBOL_GPL(snd_soc_register_card);
```

#### snd_soc_initialize_card_lists

为每一个snd_soc_card的成员初始化一个链表

```
static inline void snd_soc_initialize_card_lists(struct snd_soc_card *card)
{
    INIT_LIST_HEAD(&card->codec_dev_list);                   //  codec_dev_list
    INIT_LIST_HEAD(&card->widgets);                          // widgets
    INIT_LIST_HEAD(&card->paths);                            // paths
    INIT_LIST_HEAD(&card->dapm_list);                        // dapm_list
    INIT_LIST_HEAD(&card->aux_comp_list);                    // aux_comp_list
}
```

#### snd_soc_pcm_runtime定义
---

```
/* SoC machine DAI configuration, glues a codec and cpu DAI together */
struct snd_soc_pcm_runtime {
    struct device *dev;
    struct snd_soc_card *card;
    struct snd_soc_dai_link *dai_link;
    struct mutex pcm_mutex;
    enum snd_soc_pcm_subclass pcm_subclass;
    struct snd_pcm_ops ops; 

    unsigned int dev_registered:1;

    /* Dynamic PCM BE runtime data */
    struct snd_soc_dpcm_runtime dpcm[2];
    int fe_compr;

    long pmdown_time;
    unsigned char pop_wait:1;

    /* err in case of ops failed */
    int err_ops;
    /* runtime devices */
    struct snd_pcm *pcm;
    struct snd_compr *compr;
    struct snd_soc_codec *codec;
    struct snd_soc_platform *platform;
    struct snd_soc_dai *codec_dai;
    struct snd_soc_dai *cpu_dai;
    struct snd_soc_component *component; /* Only valid for AUX dev rtds */

    struct snd_soc_dai **codec_dais;
    unsigned int num_codecs;

    struct delayed_work delayed_work;
#ifdef CONFIG_DEBUG_FS
    struct dentry *debugfs_dpcm_root;
    struct dentry *debugfs_dpcm_state;
#endif

    unsigned int num; /* 0-based and monotonic increasing */
    struct list_head list; /* rtd list of the soc card */
};
```

```
struct snd_soc_dai {
    const char *name;
    int id; 
    struct device *dev;

    /* driver ops */
    struct snd_soc_dai_driver *driver;

    /* DAI runtime info */
    unsigned int capture_active;        /* stream is in use */
    unsigned int playback_active;       /* stream is in use */
    unsigned int symmetric_rates:1;
    unsigned int symmetric_channels:1;
    unsigned int symmetric_samplebits:1;
    unsigned int active;
    unsigned char probed:1;

    struct snd_soc_dapm_widget *playback_widget;
    struct snd_soc_dapm_widget *capture_widget;

    /* DAI DMA data */
    void *playback_dma_data;
    void *capture_dma_data;

    /* Symmetry data - only valid if symmetry is being enforced */
    unsigned int rate;
    unsigned int channels;
    unsigned int sample_bits;

    /* parent platform/codec */
    struct snd_soc_codec *codec;
    struct snd_soc_component *component;

    /* CODEC TDM slot masks and params (for fixup) */
    unsigned int tx_mask;
    unsigned int rx_mask;

    struct list_head list;
};
```

#### soc_init_dai_link

```
// 这个函数的主要功能，对snd_soc_dai_link_component赋值，然后进行校验
static int soc_init_dai_link(struct snd_soc_card *card,
                   struct snd_soc_dai_link *link)
{
    int i, ret;

    ret = snd_soc_init_multicodec(card, link);                        // 初始snd_soc_dai_link_component结构体
    if (ret) {
        dev_err(card->dev, "ASoC: failed to init multicodec\n");
        return ret;
    }

    for (i = 0; i < link->num_codecs; i++) {                         // 有两个codec，一个digital codec,一个analog codec
        /*
         * Codec must be specified by 1 of name or OF node,
         * not both or neither.
         */
        if (!!link->codecs[i].name ==
            !!link->codecs[i].of_node) {
            dev_err(card->dev, "ASoC: Neither/both codec name/of_node are set for %s\n",
                link->name);
            return -EINVAL;
        }
        /* Codec DAI name must be specified */
        if (!link->codecs[i].dai_name) {
            dev_err(card->dev, "ASoC: codec_dai_name not set for %s\n",
                link->name);
            return -EINVAL;
        }
    }

    /*
     * Platform may be specified by either name or OF node, but
     * can be left unspecified, and a dummy platform will be used.
     */
    if (link->platform_name && link->platform_of_node) {
        dev_err(card->dev,
            "ASoC: Both platform name/of_node are set for %s\n",
            link->name);
        return -EINVAL;
    }
	    /*
     * CPU device may be specified by either name or OF node, but
     * can be left unspecified, and will be matched based on DAI
     * name alone..
     */
    if (link->cpu_name && link->cpu_of_node) {
        dev_err(card->dev,
            "ASoC: Neither/both cpu name/of_node are set for %s\n",
            link->name);
        return -EINVAL;
    }
    /*
     * At least one of CPU DAI name or CPU device name/node must be
     * specified
     */
    if (!link->cpu_dai_name &&
        !(link->cpu_name || link->cpu_of_node)) {
        dev_err(card->dev,
            "ASoC: Neither cpu_dai_name nor cpu_name/of_node are set for %s\n",
            link->name);
        return -EINVAL;
    }

    return 0;
}
```

#### snd_soc_instantiate_card

绑定cpu_dai、codec_dai、platform 到rtd中

```
static int snd_soc_instantiate_card(struct snd_soc_card *card)
{
    struct snd_soc_codec *codec;
    struct snd_soc_pcm_runtime *rtd;
    struct snd_soc_dai_link *dai_link;
    int ret, i, order;

    mutex_lock(&client_mutex);
    mutex_lock_nested(&card->mutex, SND_SOC_CARD_CLASS_INIT);

    /* bind DAIs */
    for (i = 0; i < card->num_links; i++) {
        ret = soc_bind_dai_link(card, &card->dai_link[i]);                // rtd和codec_dai关联
        if (ret != 0)
            goto base_error;
    }

    /* bind aux_devs too */
    for (i = 0; i < card->num_aux_devs; i++) {                            // 绑定rtd_aux
        ret = soc_bind_aux_dev(card, i);
        if (ret != 0)
            goto base_error;
    }

    /* add predefined DAI links to the list */
    for (i = 0; i < card->num_links; i++)
        snd_soc_add_dai_link(card, card->dai_link+i);

    /* initialize the register cache for each available codec */
    list_for_each_entry(codec, &codec_list, list) {
        if (codec->cache_init)
            continue;
        ret = snd_soc_init_codec_cache(codec);                           // 初始化codec cache
        if (ret < 0)
            goto base_error;
    }

    /* card bind complete so register a sound card */
    ret = snd_card_new(card->dev, SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1,  // 注册声卡
            card->owner, 0, &card->snd_card);
    if (ret < 0) {
        dev_err(card->dev,
            "ASoC: can't create sound card for card %s: %d\n",
			            card->name, ret);
        goto base_error;
    }

    soc_init_card_debugfs(card);                                         // 创建debugfs

    card->dapm.bias_level = SND_SOC_BIAS_OFF;                            // dapm电源配置
    card->dapm.dev = card->dev;
    card->dapm.card = card;
    list_add(&card->dapm.list, &card->dapm_list);

#ifdef CONFIG_DEBUG_FS
    snd_soc_dapm_debugfs_init(&card->dapm, card->debugfs_card_root);
#endif

#ifdef CONFIG_PM_SLEEP
    /* deferred resume work */
    INIT_WORK(&card->deferred_resume_work, soc_resume_deferred);
#endif

    if (card->dapm_widgets)                                             // dapm
        snd_soc_dapm_new_controls(&card->dapm, card->dapm_widgets,
                      card->num_dapm_widgets);

    if (card->of_dapm_widgets)
        snd_soc_dapm_new_controls(&card->dapm, card->of_dapm_widgets,
                      card->num_of_dapm_widgets);

    /* initialise the sound card only once */
    if (card->probe) {
        ret = card->probe(card);                                       // 调用probe函数,这是哪个probe函数啊？
        if (ret < 0)
            goto card_probe_error;
    }

    /* probe all components used by DAI links on this card */
    for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
            order++) {
        list_for_each_entry(rtd, &card->rtd_list, list) {
            ret = soc_probe_link_components(card, rtd, order);        // 这里也调用platform, codec_dai的probe函数
            if (ret < 0) {
                dev_err(card->dev,
                    "ASoC: failed to instantiate card %d\n",
					                    ret);
                goto probe_dai_err;
            }
        }
    }

    /* probe auxiliary components */
    ret = soc_probe_aux_devices(card);                                // 这里面调用probe函数
    if (ret < 0)
        goto probe_dai_err;

    /* Find new DAI links added during probing components and bind them.
     * Components with topology may bring new DAIs and DAI links.
     */
    list_for_each_entry(dai_link, &card->dai_link_list, list) {
        if (soc_is_dai_link_bound(card, dai_link))
            continue;

        ret = soc_init_dai_link(card, dai_link);
        if (ret)
            goto probe_dai_err;
        ret = soc_bind_dai_link(card, dai_link);
        if (ret)
            goto probe_dai_err;
    }

    /* probe all DAI links on this card */
    for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
            order++) {
        list_for_each_entry(rtd, &card->rtd_list, list) {
            ret = soc_probe_link_dais(card, rtd, order);
            if (ret < 0) {
                dev_err(card->dev,
                    "ASoC: failed to instantiate card %d\n",
                    ret);
                goto probe_dai_err;
            }
        }
    }

    snd_soc_dapm_link_dai_widgets(card);
    snd_soc_dapm_connect_dai_link_widgets(card);
	    if (card->controls)
        snd_soc_add_card_controls(card, card->controls, card->num_controls);

    if (card->dapm_routes)
        snd_soc_dapm_add_routes(&card->dapm, card->dapm_routes,
                    card->num_dapm_routes);

    if (card->of_dapm_routes)
        snd_soc_dapm_add_routes(&card->dapm, card->of_dapm_routes,
                    card->num_of_dapm_routes);

    snprintf(card->snd_card->shortname, sizeof(card->snd_card->shortname),
         "%s", card->name);
    snprintf(card->snd_card->longname, sizeof(card->snd_card->longname),
         "%s", card->long_name ? card->long_name : card->name);
    snprintf(card->snd_card->driver, sizeof(card->snd_card->driver),
         "%s", card->driver_name ? card->driver_name : card->name);
    for (i = 0; i < ARRAY_SIZE(card->snd_card->driver); i++) {
        switch (card->snd_card->driver[i]) {
        case '_':
        case '-':
        case '\0':
            break;
        default:
            if (!isalnum(card->snd_card->driver[i]))
                card->snd_card->driver[i] = '_';
            break;
        }
    }

    if (card->late_probe) {
        ret = card->late_probe(card);
        if (ret < 0) {
            dev_err(card->dev, "ASoC: %s late_probe() failed: %d\n",
                card->name, ret);
            goto probe_aux_dev_err;
        }
    }

    snd_soc_dapm_new_widgets(card);

    ret = snd_card_register(card->snd_card);
        if (ret < 0) {
        dev_err(card->dev, "ASoC: failed to register soundcard %d\n",
                ret);
        goto probe_aux_dev_err;
    }

    card->instantiated = 1;
    dapm_mark_endpoints_dirty(card);
    snd_soc_dapm_sync(&card->dapm);
    mutex_unlock(&card->mutex);
    mutex_unlock(&client_mutex);

    return 0;

probe_aux_dev_err:
    soc_remove_aux_devices(card);

probe_dai_err:
    soc_remove_dai_links(card);

card_probe_error:
    if (card->remove)
        card->remove(card);

    snd_soc_dapm_free(&card->dapm);
    soc_cleanup_card_debugfs(card);
    snd_card_free(card->snd_card);

base_error:
    soc_remove_pcm_runtimes(card);
    mutex_unlock(&card->mutex);
    mutex_unlock(&client_mutex);

    return ret;
}
```

#### soc_bind_dai_link

主要用于调用codec_dai和platform的probe函数。

```
static int soc_bind_dai_link(struct snd_soc_card *card,
    struct snd_soc_dai_link *dai_link)
{
    struct snd_soc_pcm_runtime *rtd;
    struct snd_soc_dai_link_component *codecs = dai_link->codecs;
    struct snd_soc_dai_link_component cpu_dai_component;
    struct snd_soc_dai **codec_dais;
    struct snd_soc_platform *platform;
    const char *platform_name;
    int i;

    dev_dbg(card->dev, "ASoC: binding %s\n", dai_link->name);

    if (soc_is_dai_link_bound(card, dai_link)) {                  // 查看是否已经绑定
        dev_dbg(card->dev, "ASoC: dai link %s already bound\n",
            dai_link->name);
        return 0;
    }

    rtd = soc_new_pcm_runtime(card, dai_link);                   // 初始化snd_soc_pcm_runtime结构体
    if (!rtd)
        return -ENOMEM;

    cpu_dai_component.name = dai_link->cpu_name;
    cpu_dai_component.of_node = dai_link->cpu_of_node;
    cpu_dai_component.dai_name = dai_link->cpu_dai_name;
    rtd->cpu_dai = snd_soc_find_dai(&cpu_dai_component);        // snd_soc_pcm_runtime和snd_soc_dai关联
    if (!rtd->cpu_dai) {
        dev_err(card->dev, "ASoC: CPU DAI %s not registered\n",
            dai_link->cpu_dai_name);
        goto _err_defer;
    }

    rtd->num_codecs = dai_link->num_codecs;                    // 两个codec

    /* Find CODEC from registered CODECs */
    codec_dais = rtd->codec_dais;
    for (i = 0; i < rtd->num_codecs; i++) {
        codec_dais[i] = snd_soc_find_dai(&codecs[i]);                  // 查看DAI是否已经注册
        if (!codec_dais[i]) {
            dev_err(card->dev, "ASoC: CODEC DAI %s not registered\n",
                codecs[i].dai_name);
            goto _err_defer;
			        }
    }

    /* Single codec links expect codec and codec_dai in runtime data */
    rtd->codec_dai = codec_dais[0];
    rtd->codec = rtd->codec_dai->codec;

    /* if there's no platform we match on the empty platform */
    platform_name = dai_link->platform_name;
    if (!platform_name && !dai_link->platform_of_node)
        platform_name = "snd-soc-dummy";

    /* find one from the set of registered platforms */
    list_for_each_entry(platform, &platform_list, list) {
        if (dai_link->platform_of_node) {
            if (platform->dev->of_node !=
                dai_link->platform_of_node)
                continue;
        } else {
            if (strcmp(platform->component.name, platform_name))
                continue;
        }

        rtd->platform = platform;
    }
    if (!rtd->platform) {
        dev_err(card->dev, "ASoC: platform %s not registered\n",
            ((platform_name) ? platform_name :
              dai_link->platform_of_node->full_name));
        goto _err_defer;
    }

    soc_add_pcm_runtime(card, rtd);                  // 添加到链表中
    return 0;

_err_defer:
    soc_free_pcm_runtime(rtd);
    return  -EPROBE_DEFER;
}
```

#### soc_probe_link_components
---

查看probe函数是合适调用的。
```
static int soc_probe_link_components(struct snd_soc_card *card,
            struct snd_soc_pcm_runtime *rtd,
                     int order)
{
    struct snd_soc_platform *platform = rtd->platform;
    struct snd_soc_component *component;
    int i, ret;

    /* probe the CPU-side component, if it is a CODEC */
    component = rtd->cpu_dai->component;
    if (component->driver->probe_order == order) {
        ret = soc_probe_component(card, component);
        if (ret < 0)
            return ret;
    }

    /* probe the CODEC-side components */
    for (i = 0; i < rtd->num_codecs; i++) {
        component = rtd->codec_dais[i]->component;
        if (component->driver->probe_order == order) {
            ret = soc_probe_component(card, component);            // 调用codec_dai的probe函数
            if (ret < 0)
                return ret;
        }
    }

    /* probe the platform */
    if (platform->component.driver->probe_order == order) {
        ret = soc_probe_component(card, &platform->component);     // 调用platform的probe函数
        if (ret < 0)
            return ret;
    }

    return 0;
}
```
	