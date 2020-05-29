```  
// 一个pcm对应一个dai_link
machine文档跟到snd_soc_register_card

sound/soc/soc-core.c
snd_soc_register_card
  snd_soc_instantiate_card             // 声卡创建，pcm创建
    soc_probe_link_dais                

sound/soc/soc-pcm.c	  
	  soc_new_pcm
        snd_pcm_new

sound/core/pcm.c
snd_pcm_new
  _snd_pcm_new
  
 static int _snd_pcm_new(struct snd_card *card, const char *id, int device,
        int playback_count, int capture_count, bool internal,
        struct snd_pcm **rpcm)
{
    struct snd_pcm *pcm;
    int err; 
    static struct snd_device_ops ops = {
        .dev_free = snd_pcm_dev_free,
        .dev_register = snd_pcm_dev_register,                             // pcm设备节点创建
        .dev_disconnect = snd_pcm_dev_disconnect,
    };   

    if (snd_BUG_ON(!card))
        return -ENXIO;
    if (rpcm)
        *rpcm = NULL;
    pcm = kzalloc(sizeof(*pcm), GFP_KERNEL);
    if (!pcm)
        return -ENOMEM;
    pcm->card = card;
    pcm->device = device;
    pcm->internal = internal;
    mutex_init(&pcm->open_mutex);
    init_waitqueue_head(&pcm->open_wait);
    INIT_LIST_HEAD(&pcm->list);
    if (id) 
        strlcpy(pcm->id, id, sizeof(pcm->id));
    if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_PLAYBACK, playback_count)) < 0) { // 新建playback
        snd_pcm_free(pcm);
        return err; 
    }    
    if ((err = snd_pcm_new_stream(pcm, SNDRV_PCM_STREAM_CAPTURE, capture_count)) < 0) { // 新建capture
        snd_pcm_free(pcm);
        return err; 
    }    
    if ((err = snd_device_new(card, SNDRV_DEV_PCM, pcm, &ops)) < 0) {  // 新建snd_device，挂到snd_card上
        snd_pcm_free(pcm);
        return err; 
    }    
    if (rpcm)
	        *rpcm = pcm;
    return 0;
}


int snd_pcm_new_stream(struct snd_pcm *pcm, int stream, int substream_count)
{
    int idx, err;
    struct snd_pcm_str *pstr = &pcm->streams[stream];
    struct snd_pcm_substream *substream, *prev;

#if IS_ENABLED(CONFIG_SND_PCM_OSS)
    mutex_init(&pstr->oss.setup_mutex);
#endif
    pstr->stream = stream;
    pstr->pcm = pcm;
    pstr->substream_count = substream_count;
    if (!substream_count)
        return 0;

    snd_device_initialize(&pstr->dev, pcm->card);
    pstr->dev.groups = pcm_dev_attr_groups;
    dev_set_name(&pstr->dev, "pcmC%iD%i%c", pcm->card->number, pcm->device,       // 设置pcm设备的名称
             stream == SNDRV_PCM_STREAM_PLAYBACK ? 'p' : 'c');

    if (!pcm->internal) {
        err = snd_pcm_stream_proc_init(pstr);                                    // 创建 proc文件系统的文件 
        if (err < 0) {
            pcm_err(pcm, "Error in snd_pcm_stream_proc_init\n");
            return err;
        }
    }
    prev = NULL;
    for (idx = 0, prev = NULL; idx < substream_count; idx++) {
        substream = kzalloc(sizeof(*substream), GFP_KERNEL);
        if (!substream)
            return -ENOMEM;
        substream->pcm = pcm;
        substream->pstr = pstr;
        substream->number = idx;
        substream->stream = stream;
        sprintf(substream->name, "subdevice #%i", idx);
        substream->buffer_bytes_max = UINT_MAX;
		        if (prev == NULL)
            pstr->substream = substream;
        else
            prev->next = substream;

        if (!pcm->internal) {
            err = snd_pcm_substream_proc_init(substream);                 // proc/asound/card0/pcm0p/sub0目录和下面具体的文件
            if (err < 0) {
                pcm_err(pcm,
                    "Error in snd_pcm_stream_proc_init\n");
                if (prev == NULL)
                    pstr->substream = NULL;
                else
                    prev->next = NULL;
                kfree(substream);
                return err;
            }
        }
        substream->group = &substream->self_group;
        spin_lock_init(&substream->self_group.lock);
        spin_lock_init(&substream->runtime_lock);
        mutex_init(&substream->self_group.mutex);
        INIT_LIST_HEAD(&substream->self_group.substreams);
        list_add_tail(&substream->link_list, &substream->self_group.substreams);
        atomic_set(&substream->mmap_count, 0);
        prev = substream;
    }
    return 0;
}
EXPORT_SYMBOL(snd_pcm_new_stream);
```


```
static int snd_pcm_dev_register(struct snd_device *device)
{
    int cidx, err;
    struct snd_pcm_substream *substream;
    struct snd_pcm_notify *notify;
    struct snd_pcm *pcm;

    if (snd_BUG_ON(!device || !device->device_data))
        return -ENXIO;
    pcm = device->device_data;
    if (pcm->internal)
        return 0;

    mutex_lock(&register_mutex);
    err = snd_pcm_add(pcm);
    if (err)
        goto unlock;
    for (cidx = 0; cidx < 2; cidx++) {
        int devtype = -1;
        if (pcm->streams[cidx].substream == NULL)
            continue;
        switch (cidx) {
        case SNDRV_PCM_STREAM_PLAYBACK:                               // playback
            devtype = SNDRV_DEVICE_TYPE_PCM_PLAYBACK;
            break;
        case SNDRV_PCM_STREAM_CAPTURE:                                // capture
            devtype = SNDRV_DEVICE_TYPE_PCM_CAPTURE;
            break;
        }
        /* register pcm */
        err = snd_register_device(devtype, pcm->card, pcm->device,            // 注册pcm
                      &snd_pcm_f_ops[cidx], pcm,
                      &pcm->streams[cidx].dev);
        if (err < 0) {
            list_del_init(&pcm->list);
            goto unlock;
        }

        for (substream = pcm->streams[cidx].substream; substream; substream = substream->next)
            snd_pcm_timer_init(substream);
    }
	
	    list_for_each_entry(notify, &snd_pcm_notify_list, list)
        notify->n_register(pcm);

 unlock:
    mutex_unlock(&register_mutex);
    return err;
}
```

sound/soc/sound.c
```
int snd_register_device(int type, struct snd_card *card, int dev,
            const struct file_operations *f_ops,
            void *private_data, struct device *device)
{
    int minor;
    int err = 0;
    struct snd_minor *preg;

    if (snd_BUG_ON(!device))
        return -EINVAL;

    preg = kmalloc(sizeof *preg, GFP_KERNEL);
    if (preg == NULL)
        return -ENOMEM;
    preg->type = type;
    preg->card = card ? card->number : -1; 
    preg->device = dev;
    preg->f_ops = f_ops;
    preg->private_data = private_data;
    preg->card_ptr = card;
    mutex_lock(&sound_mutex);
    minor = snd_find_free_minor(type, card, dev);
    if (minor < 0) {
        err = minor;
        goto error;
    }   

    preg->dev = device;
    device->devt = MKDEV(major, minor);
    err = device_add(device);                // 创建设备节点
    if (err < 0)
        goto error;

    snd_minors[minor] = preg;
 error:
    mutex_unlock(&sound_mutex);
    if (err < 0)
        kfree(preg);
    return err;
}
EXPORT_SYMBOL(snd_register_device);
```

```
const struct file_operations snd_pcm_f_ops[2] = {
    {    
        .owner =        THIS_MODULE,
        .write =        snd_pcm_write,
        .write_iter =       snd_pcm_writev,
        .open =         snd_pcm_playback_open,              // playback open函数
        .release =      snd_pcm_release,
        .llseek =       no_llseek,
        .poll =         snd_pcm_playback_poll,
        .unlocked_ioctl =   snd_pcm_playback_ioctl,
        .compat_ioctl =     snd_pcm_ioctl_compat,
        .mmap =         snd_pcm_mmap,
        .fasync =       snd_pcm_fasync,
        .get_unmapped_area =    snd_pcm_get_unmapped_area,
    },   
    {    
        .owner =        THIS_MODULE,
        .read =         snd_pcm_read,
        .read_iter =        snd_pcm_readv,
        .open =         snd_pcm_capture_open,              // capture open函数
        .release =      snd_pcm_release,
        .llseek =       no_llseek,
        .poll =         snd_pcm_capture_poll,
        .unlocked_ioctl =   snd_pcm_capture_ioctl,
        .compat_ioctl =     snd_pcm_ioctl_compat,
        .mmap =         snd_pcm_mmap,
        .fasync =       snd_pcm_fasync,
        .get_unmapped_area =    snd_pcm_get_unmapped_area,
    }    
};
```

dev的设备节点
```
ls /dev/snd
comprC0D24 comprC0D40 hwC0D14   hwC0D46 pcmC0D0c  pcmC0D13p pcmC0D17p pcmC0D22c pcmC0D34c pcmC0D3c  timer
comprC0D28 comprC0D9  hwC0D15   hwC0D5  pcmC0D0p  pcmC0D14c pcmC0D18c pcmC0D23c pcmC0D34p pcmC0D3p
comprC0D29 controlC0  hwC0D16   hwC0D52 pcmC0D10c pcmC0D14p pcmC0D18p pcmC0D25p pcmC0D35c pcmC0D41p
comprC0D30 hwC0D10    hwC0D3033 hwC0D53 pcmC0D10p pcmC0D15c pcmC0D19c pcmC0D26c pcmC0D35p pcmC0D4p
comprC0D31 hwC0D1000  hwC0D39   hwC0D6  pcmC0D11c pcmC0D15p pcmC0D1c  pcmC0D27p pcmC0D36c pcmC0D5p
comprC0D32 hwC0D11    hwC0D40   hwC0D7  pcmC0D12c pcmC0D16c pcmC0D1p  pcmC0D2c  pcmC0D36p pcmC0D6c
comprC0D38 hwC0D12    hwC0D41   hwC0D8  pcmC0D12p pcmC0D16p pcmC0D20c pcmC0D2p  pcmC0D37c pcmC0D7p
comprC0D39 hwC0D13    hwC0D43   hwC0D9  pcmC0D13c pcmC0D17c pcmC0D21c pcmC0D33p pcmC0D37p pcmC0D8c
```