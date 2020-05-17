记录msm8952 audio的machine驱动初始化过程。

在android Q上，audio与高通相关的代码移动到了vendor/qcom/opensource/audio-kernel目录。

SDM450平台的codec分为CPU内部的数字codec和外挂的模拟codec(PM8953)。本文只分析内部数字codec。

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

驱动在初始化过程中，会用到设备树的信息。msm8953-audio.dtsi

```
    // 内部codec
    int_codec: sound {
        status = "okay";
        compatible = "qcom,msm8952-audio-codec";
        qcom,model = "msm8953-snd-card-mtp";    // 声卡名称
        reg = <0xc051000 0x4>,      // 寄存器地址，已经寄存器长度
            <0xc051004 0x4>,
            <0xc055000 0x4>,
            <0xc052000 0x4>;
        reg-names = "csr_gp_io_mux_mic_ctl",    // 寄存器名称
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
    // 读取设备树中寄存器
    muxsel = platform_get_resource_byname(pdev, IORESOURCE_MEM,
            "csr_gp_io_mux_mic_ctl");
    if (!muxsel) {
        dev_err(&pdev->dev, "MUX addr invalid for MI2S\n");
        ret = -ENODEV;
        goto err1;
		    // 将物理地址映射成虚拟地址
    pdata->vaddr_gpio_mux_mic_ctl =
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
    // 获取设备树时钟
    ret = of_property_read_u32(pdev->dev.of_node, mclk, &id);
    if (ret) {
	        dev_err(&pdev->dev,
                "%s: missing %s in dt node\n", __func__, mclk);
        id = DEFAULT_MCLK_RATE;
    }
    pdata->mclk_freq = id;

    /*reading the gpio configurations from dtsi file*/
    // wsa为空, 所以不做处理
    num_strings = of_property_count_strings(pdev->dev.of_node,
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
    // 处理dai_links相关操作
    card = msm8952_populate_sndcard_dailinks(&pdev->dev);
    dev_dbg(&pdev->dev, "default codec configured\n");
    // qcom.msm-ext-pa，外部功率放大器,external power amplifier
    num_strings = of_property_count_strings(pdev->dev.of_node,
            ext_pa);
    if (num_strings < 0) {
        dev_err(&pdev->dev,
                "%s: missing %s in dt node or length is incorrect\n",
                __func__, ext_pa);
        goto err;
    }
    // 获取外部功放的信息
    for (i = 0; i < num_strings; i++) {
        ret = of_property_read_string_index(pdev->dev.of_node,
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
    // 外部喇叭功放为空
    pdata->spk_ext_pa_gpio = of_get_named_gpio(pdev->dev.of_node,
                            spk_ext_pa, 0);
    if (pdata->spk_ext_pa_gpio < 0) {
        dev_err(&pdev->dev, "%s: missing %s in dt node\n",
            __func__, spk_ext_pa);
    }

    pdata->spk_ext_pa_gpio_p = of_parse_phandle(pdev->dev.of_node,
                            spk_ext_pa, 0);
    // 查看美国标准，欧洲标准切换的gpio是否支持
    ret = is_us_eu_switch_gpio_support(pdev, pdata);
    if (ret < 0) {
        pr_err("%s: failed to is_us_eu_switch_gpio_support %d\n",
                __func__, ret);
        goto err;
    }
    // 不支持外部喇叭
	    ret = is_ext_spk_gpio_support(pdev, pdata);
    if (ret < 0)
        pr_err("%s:  doesn't support external speaker pa\n",
                __func__);
    // 没有外部功放
    enable_aw8738 = of_property_read_bool(pdev->dev.of_node , "action,enable-aw8738");
    if(enable_aw8738)
    {
        pr_err("msm8952_asoc_machine_probe enable_aw8738\n");
        gpio_request(pdata->spk_ext_pa_gpio, "AW8738_EN");
        gpio_direction_output(pdata->spk_ext_pa_gpio,0);
    }
    // 不支持外部功放
    enable_aw87519 = of_property_read_bool(pdev->dev.of_node , "action,enable-aw87519");
    pr_info("%s: enable_aw87519 = %d\n", __func__, enable_aw87519);
    // 获取cdc-comp-gpios
    pdata->comp_gpio_p = of_parse_phandle(pdev->dev.of_node,
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
    // 获取设备树MIC偏置电压类型
    ret = of_property_read_string(pdev->dev.of_node,
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
        // 使用内部偏置电压
        dev_err(&pdev->dev, "Headset is using internal micbias\n");
        mbhc_cfg.hs_ext_micbias = false;
    }
    // 获取AFE时钟
    ret = of_property_read_u32(pdev->dev.of_node,
                  "qcom,msm-afe-clk-ver", &val);
    if (ret)
        pdata->afe_clk_ver = AFE_CLK_VERSION_V2;
    else
        pdata->afe_clk_ver = val;
    /* initialize the mclk */
    // 初始化数字codec
    pdata->digital_cdc_clk.i2s_cfg_minor_version =
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
    // 数据保存到pdev中
    platform_set_drvdata(pdev, card);
    // 数据保存到card中
    snd_soc_card_set_drvdata(card, pdata);
    // 获取设备树声卡名称
    ret = snd_soc_of_parse_card_name(card, "qcom,model");
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
    // 获取设备树音频通路的信息
    ret = snd_soc_of_parse_audio_routing(card,
            "qcom,audio-routing");
    if (ret)
        goto err;

    ret = msm8952_populate_dai_link_component_of_node(card);
    if (ret) {
        ret = -EPROBE_DEFER;
        goto err;
    }
    // 注册声卡
    ret = devm_snd_soc_register_card(&pdev->dev, card);
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

查看对dai_link的处理

```
static struct snd_soc_card *msm8952_populate_sndcard_dailinks(
                        struct device *dev)
{
    struct snd_soc_card *card = &bear_card;
    struct snd_soc_dai_link *dailink;
    int len1;

    card->name = dev_name(dev);
    len1 = ARRAY_SIZE(msm8952_dai);     // 获取数组个数
    // 将msm8952_dai数组复制到msm8952_dai_links数组
    memcpy(msm8952_dai_links, msm8952_dai, sizeof(msm8952_dai));
    dailink = msm8952_dai_links;
    // 查看是否存在hdmi的dai_link, 本产品设备树中没有, 使用默认的
    if (of_property_read_bool(dev->of_node,
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
    // 查看是否存在qcom,split-a2dp的定义,也没有
    if (of_property_read_bool(dev->of_node,
                "qcom,split-a2dp")) {
        dev_dbg(dev, "%s(): split a2dp support present\n",
		                __func__);
        memcpy(dailink + len1, msm8952_split_a2dp_dai_link,
                sizeof(msm8952_split_a2dp_dai_link));
        len1 += ARRAY_SIZE(msm8952_split_a2dp_dai_link);
    }
    // dailink保存到snd_card结构体中
    card->dai_link = dailink;
    // 长度
    card->num_links = len1;
    return card;
}
```
重点是其中的msm8952_dai数组，里面定义了前端，后端的各种DAI链路。

CPU <---> 前端DAI接口 <---> 中间DSP <---> 后端DAI接口 <--->数字CODEC <--->模拟codec

```
// CPU <---> 前端DAI接口 <---> 中间DSP <---> 后端DAI接口 <--->数字CODEC <--->模拟codec
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
        .num_codecs = CODECS_MAX,
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

probe函数的其他内容看注释，跟代码就行了。最后是注册声卡。

kernel/msm-4.9/sound/soc/soc-devres.c
```
int devm_snd_soc_register_card(struct device *dev, struct snd_soc_card *card)
{
    struct snd_soc_card **ptr;
    int ret;

    ptr = devres_alloc(devm_card_release, sizeof(*ptr), GFP_KERNEL);
    if (!ptr)
        return -ENOMEM;

    ret = snd_soc_register_card(card);
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

    for (i = 0; i < card->num_links; i++) {
        struct snd_soc_dai_link *link = &card->dai_link[i];

        ret = soc_init_dai_link(card, link);
        if (ret) {
            dev_err(card->dev, "ASoC: failed to init link %s\n",
                link->name);
            return ret;
        }
    }
    // 设置声卡设备驱动信息
    dev_set_drvdata(card->dev, card);
    // 初始化声卡列表
    snd_soc_initialize_card_lists(card);

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
    // 初始化声卡
    ret = snd_soc_instantiate_card(card);
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
```