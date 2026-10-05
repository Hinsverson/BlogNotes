# FFMPEG 重要类解析

[【iOS】FFMpeg SDK 开发手册 - 简书](https://www.jianshu.com/p/c79fcda6bb1c)
[音视频专辑 - 专题 - 简书](https://www.jianshu.com/c/5cef16ba6351)

# 类
## AVFormatContext
这个结构体描述了一个媒体文件或媒体流的构成和基本信息
这是FFMpeg中最为基本的一个结构，是其他所有结构的根，是一个多媒体文件或流的根本抽象。其中:
- nb_streams和streams所表示的AVStream结构指针数组包含了所有内嵌媒体流的描述。
- iformat和oformat指向对应的demuxer和muxer指针；
- pb则指向一个控制底层数据读写的ByteIOContext结构。
- start_time和duration是从streams数组的各个AVStream中推断出的多媒体文件的起始时间和长度，以微妙为单位。

通常，这个结构由av_open_input_file在内部创建并以缺省值初始化部分成员。但是，如果调用者希望自己创建该结构，则需要显式为该结构的一些成员置缺省值——如果没有缺省值的话，会导致之后的动作产生异常。以下成员需要被关注：
probesize
mux_rate
packet_size
flags
max_analyze_duration
key
max_index_size
max_picture_buffer
max_delay


## AVStream
该结构体描述一个媒体流，主要域的释义如下，其中大部分域的值可以由av_open_input_file根据文件头的信息确定，缺少的信息需要通过调用av_find_stream_info读帧及软解码进一步获取：

Index/id：index对应流的索引，这个数字是自动生成的，根据index可以从AVFormatContext::streams表中索引到该流；而id则是流的标识，依赖于具体的容器格式。比如对于MPEG TS格式，id就是pid。
time_base：流的时间基准，是一个实数，该流中媒体数据的pts和dts都将以这个时间基准为粒度。通常，使用av_rescale/av_rescale_q可以实现不同时间基准的转换。
start_time：流的起始时间，以流的时间基准为单位，通常是该流中第一个帧的pts。
Duration：流的总时间，以流的时间基准为单位。
need_parsing：对该流parsing过程的控制域。
nb_frames：流内的帧数目。
r_frame_rate/framerate/avg_frame_rate：帧率相关。
codec：指向该流对应的AVCodecContext结构，调用av_open_input_file时生成。
parser：指向该流对应的AVCodecParserContext结构，调用av_find_stream_info时生成。

[FFmpeg中AVStream重要参数分析|入门笔记](https://www.rumenz.com/rumenbiji/ffmpeg-avstream.html)
``` c
int index; //在AVFormatContext中的索引，这个数字是自动生成的，可以通过这个数字从AVFormatContext::streams表中索引到该流。
int id;//流的标识，依赖于具体的容器格式。解码：由libavformat设置。编码：由用户设置，如果未设置则由libavformat替换。
AVCodecContext *codec;//已经废除,由codecpar代替
AVRational time_base;//这是表示帧时间戳的基本时间单位（以秒为单位）。该流中媒体数据的pts和dts都将以这个时间基准为粒度。
int64_t start_time;//流的起始时间，以流的时间基准为单位。如需设置，100％确保你设置它的值真的是第一帧的pts。
int64_t duration;//解码：流的持续时间。如果源文件未指定持续时间，但指定了比特率，则将根据比特率和文件大小估计该值。
int64_t nb_frames; //此流中的帧数（如果已知）或0。
enum AVDiscard discard;//选择哪些数据包可以随意丢弃，不需要去demux。
AVRational sample_aspect_ratio;//样本长宽比（如果未知，则为0）。
AVDictionary *metadata;//元数据信息。
AVRational avg_frame_rate;//平均帧速率。解封装：可以在创建流时设置为libavformat，也可以在avformat_find_stream_info（）中设置。封装：可以由调用者在avformat_write_header（）之前设置。
AVPacket attached_pic;//附带的图片。比如说一些MP3，AAC音频文件附带的专辑封面。
int probe_packets;//编解码器用于probe的包的个数。
int codec_info_nb_frames;//在av_find_stream_info（）期间已经解封装的帧数。
int request_probe;//流探测状态，1表示探测完成，0表示没有探测请求，rest 执行探测。
int skip_to_keyframe;//表示应丢弃直到下一个关键帧的所有内容。
int skip_samples;//在从下一个数据包解码的帧开始时要跳过的采样数。
int64_t start_skip_samples;//如果不是0，则应该从流的开始跳过的采样的数目。
int64_t first_discard_sample;//如果不是0，则应该从流中丢弃第一个音频样本。

int64_t pts_reorder_error[MAX_REORDER_DELAY+1];
uint8_t pts_reorder_error_count[MAX_REORDER_DELAY+1];//内部数据，从pts生成dts。

int64_t last_dts_for_order_check;
uint8_t dts_ordered;
uint8_t dts_misordered;//内部数据，用于分析dts和检测故障mpeg流。
AVRational display_aspect_ratio;//显示宽高比。
AVCodecParameters *codecpar;　　// 包含音视频参数的结构体。很重要，可以用来获取音视频参数中的宽度、高度、采样率、编码格式等信息。
```

## AVPacket
对于视频而言, 它通常包含一个压缩帧,对音频而言,可能包含多个压缩帧,该结构体类型通过av_malloc()函数分配内存,通过av_packet_ref()函数拷贝,通过av_packet_unref().函数释放内存.

# AVCodecParameters

``` c
enum AVMediaType codec_type; 　　// 编码类型。说明这段流数据究竟是音频还是视频。
enum AVCodecID codec_id     　　　// 编码格式。说明这段流的编码格式，h264，MPEG4, MJPEG，etc...
uint32_t  codecTag;              //  一般不用
int format;                      //  格式。对于视频来说指的就是像素格式(YUV420,YUV422...)，对于音频来说，指的就是音频的采样格式。
int width, int height;           // 视频的宽高，只有视频有
uint64_t channel_layout;         // 取默认值即可
int channels;                    // 声道数
int sample_rate;                 // 样本率
int frame_size;                  // 只针对音频，一帧音频的大小
```


## SwrContext


# 函数
### avformat_open_input
``` c
int avformat_open_input(AVFormatContext **ps, const char *url, ff_const59 AVInputFormat *fmt, AVDictionary **options);
```
[FFmpeg源码分析：avformat_open_input_小薇子的博客-CSDN博客](https://blog.csdn.net/qq_36391075/article/details/90514721)
该函数用于打开一个输入的封装器。在调用该函数之前，须确保av_register_all()和avformat_network_init()已调用。
参数说明：
AVFormatContext **ps, 格式化的上下文。要注意，如果传入的是一个AVFormatContext*的指针，则该空间须自己手动清理，若传入的指针为空，则FFmpeg会内部自己创建。
const char *url, 传入的地址。支持http,RTSP,以及普通的本地文件。地址最终会存入到AVFormatContext结构体当中。
AVInputFormat *fmt, 指定输入的封装格式。一般传NULL，由FFmpeg自行探测。
AVDictionary **options, 其它参数设置。它是一个字典，用于参数传递，不传则写NULL。参见：libavformat/options_table.h,其中包含了它支持的参数设置。

### avformat_find_stream_info
``` c
int avformat_find_stream_info(AVFormatContext *ic, AVDictionary **options);
```
检索、读取媒体文件的数据包以获取流信息。有一些文件格式没有头，比如说MPEG格式的，这个时候，这个函数就很有用，因为它可以从读取到的包中获得到流的信息。

### av_find_best_stream
``` c
int av_find_best_stream(AVFormatContext *ic,
                        enum AVMediaType type, //要选择的流类型
                        int wanted_stream_nb, //目标流索引
                        int related_stream, //参考流索引
                        AVCodec **decoder_ret,
                        int flags);
```
寻找特定流（video、audio、subtitle）并获取流索引：
如果指定了正确的wanted_stream_nb，一般情况都是直接返回该指定流，即用户选择的流。如果指定了参考流，且未指定目标流的情况，会在参考流的同一个节目中查找所需类型的流，但一般结果，都是返回该类型第一个流。

### av_read_frame
``` c
int av_read_frame(AVFormatContext *s, AVPacket *pkt);
// 文件格式上下文，输入的AVFormatContext
```
返回流的下一帧。
*此函数返回存储在文件中的内容，但不验证解码器是否有有效帧。
它将把文件中存储的内容拆分为帧，并为每个调用返回一个帧。
它不会省略有效帧之间的无效数据，以便给解码器最大可能的解码信息。
如果pkt->buf为NULL，那么直到下一个av_read_frame()或直到avformat_close_input()，包都是有效的。
否则数据包将无限期有效。在这两种情况下，当不再需要包时，必须使用av_free_packet释放包。
对于视频，数据包只包含一帧。
对于音频，如果每个帧具有已知的固定大小(例如PCM或ADPCM数据)，则它包含整数帧数。
如果音频帧有一个可变的大小(例如MPEG音频)，那么它包含一帧。
在AVStream中，pkt->pts、pkt->dts和pkt->持续时间总是被设置为恰当的值。
time_base单元(猜测格式是否不能提供它们)。
如果视频格式为B-frames，pkt->pts可以是AV_NOPTS_VALUE，所以如果不解压缩有效负载，最好依赖pkt->dts。
参数说明：
AVFormatContext *s 　　// 文件格式上下文，输入的AVFormatContext
AVPacket *pkt 　 // 这个值不能传NULL，必须是一个空间，输出的AVPacket

### av_seek_frame
``` c
int av_seek_frame(AVFormatContext *s, int stream_index, int64_t timestamp,
                  int flags);
```
Timebase指的是时间戳，对应pts时间戳，如果index是-1，则使用AV_TIMEBASE作为timebase并由ffmpeg自动转换成默认时间戳， 如果指定了stream那么就要使用相应的stream的timebase来计算pts了。这里注意的是比如seek到32s不能简单的直接32*AV_TIMEBASE来计算时间戳，因为pts不一定是从0开始的，所以要加上起始的pts。
stream_index是选择针对哪一条媒体流来做seek
flag用来指定寻找寻找的I帧和指定点之间的位置关系，因为seek过去的时间点不一定就处在I帧的地方，解码需要依赖于I帧，所以这时候就得选择一个附近的I帧，flag表明要seek到当前帧的前面一个I帧还是后面一个I帧。


### sws_getCachedContext
```c
struct SwsContext *sws_getCachedContext(struct SwsContext *context,
                                        int srcW, int srcH, enum AVPixelFormat srcFormat,
                                        int dstW, int dstH, enum AVPixelFormat dstFormat,
                                        int flags, SwsFilter *srcFilter,
                                        SwsFilter *dstFilter, const double *param);
```

[SwrContext重采样结构体_Stoneshen的博客-CSDN博客](https://blog.csdn.net/u011003120/article/details/81542347)

### av_dict_set_int

### av_log_set_callback
