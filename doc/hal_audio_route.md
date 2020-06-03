参考一下连接就可以：

https://blog.csdn.net/azloong/article/details/79383323

顺便记录一下上层的参数是如何写入到底层的。

```
frameworks/av/media/libaudiohal/impl/StreamHalLocal.cpp
status_t StreamHalLocal::setParameters(const String8& kvPairs) {
    return mStream->set_parameters(mStream, kvPairs.string());
}

audio_stream_t mStream;

hardware/libhardware/include/hardware/audio.h
typedef struct audio_stream audio_stream_t

struct stream_out {
    struct audio_stream_out stream;
struct audio_stream_out {
    /** 
     * Common methods of the audio stream out.  This *must* be the first member of audio_stream_out
     * as users of this structure will cast a audio_stream to audio_stream_out pointer in contexts
     * where it's known the audio_stream references an audio_stream_out.
     */
    struct audio_stream common;
	
hardware/libhardware/include/hardware/audio.h
struct audio_stream {

    /**
     * set/get audio stream parameters. The function accepts a list of
     * parameter key value pairs in the form: key1=value1;key2=value2;...
     *
     * Some keys are reserved for standard parameters (See AudioParameter class)
     *
     * If the implementation does not accept a parameter change while
     * the output is active but the parameter is acceptable otherwise, it must
     * return -ENOSYS.
     *
     * The audio flinger will put the stream in standby and then change the
     * parameter value.
     */
    int (*set_parameters)(struct audio_stream *stream, const char *kv_pairs);

hardware/qcom/audio/hal/audio_hw.c:6670:    out->stream.common.set_parameters = out_set_parameters;

最后framework调用StreamHalLocal::setParameters，就是调用out_set_parameters函数。
```