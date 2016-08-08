原文：http://dranger.com/ffmpeg/ffmpeg.html

ffmpeg是一个非常好的库，可以用来创建视频应用。它帮我们做了很多困难的事情，比如编解码，复用，解复用。使用它可以非常简单的创建多媒体应用。
它非常简单，用C语言写成，高效，并且几乎能解码所有的视频。

但唯一的问题是，它几乎没有什么文档。这是一个展示ffmpeg基础知识的教程。那么，当我决定学习ffmpeg和数字音视频是如何工作的，我决定将这个过程做成一个教程。

ffplay是一个使用ffmpeg的示例程序。它是一个简单的C程序，使用ffmpeg实现了一个完整的视频播放器。这个教程是基于ffplay.c的，每一章，将介绍一个功能并解释它是如何实现的，并会有一个C文件，你可以下载并编译它。这些源文件会告诉你真正的程序是如何工作的，我们如何处理，以及技术细节。当这个教程结束，我们将会有一个少于1000行代码的视频播放器。

这个播放器，我们使用SDL进行音视频的输出。SDL是一个优秀的跨平台多媒体库，可以做播放器，仿真器或者游戏。在教程开始之前需要先下载安装SDL开发库。

本教程针对有一定编程基础的。至少需要了解C语言，知道一些队列，互斥锁这样的东西。还应该知道一些多媒体的基础知识，比如波形，但是不需要知道很多。

下面开始进入教程

###第一章：屏幕捕捉

> Code: [tutorial01.c](http://dranger.com/ffmpeg/tutorial01.c)

####综述

视频文件有一些基本部分。首先，文件本身就是一个容器，这个容器的类型决定了文件中的信息。比如：AVI，Quicktime就是容器。接下来，可以获取到一些流数据，比如通常一个视频文件中会包含一个音频流和一个视频流（其实流就是随时间变化的连续的数据）。流中的数据被称作帧(frame)。每一个流都被不同的编解码器(codec)编码，编解码器定义了如何去进行编码和解码。比如：DivX 和 MP3。数据包(Packets)是从流中读出来的。从数据包中拿到的数据可以被解码器解码成原始帧，那么这些原始帧就可以被应用使用。

在底层，处理音视频流是很容易的：

    10 OPEN video_stream FROM video.avi
    20 READ packet FROM video_stream INTO frame
    30 IF frame NOT COMPLETE GOTO 20
    40 DO SOMETHING WITH frame
    50 GOTO 20


尽管有的程序有一些复杂的处理流程，但用ffmpeg处理多媒体是非常简单的。在这个教程里，我们会打开一个文件，从文件中读取视频流，并把它的帧写到一个PPM文件中。

####打开文件

首先我们看如何打开一个文件，使用ffmpeg之前，必须先初始化这个库（av_register_all()）。

    #include <libavcodec/avcodec.h>
    #include <libavformat/avformat.h>
    #include <ffmpeg/swscale.h>
    ...
    int main(int argc, charg *argv[]) {
        av_register_all();

这个操作可以注册库里所有的封装格式和编解码器，当打开一个文件的时候，它会自动使用相应的封装格式和编解码器(format/codec)。
注意，av_register_all()函数只需要调用一次。

接下来我们真正的打开一个文件:

    AVFormatContext *pFormatCtx = NULL;
    
    // Open video file
    if(avformat_open_input(&pFormatCtx, argv[1], NULL, 0, NULL)!=0)
      return -1; // Couldn't open file

从第一个参数中获取文件名，这个函数读取了文件头，并将它的格式信息存储到 AVFormatContext 结构中。最后三个参数用于指定文件类型，buffer大小和格式选项，但将其设置成 NULL 和 0 时，libavformat会自动设置这些。

这个方法只是查看头部信息，接下来我们要从文件中取出流信息:

    // Retrieve stream information
    if(avformat_find_stream_info(pFormatCtx, NULL)<0)
      return -1; // Couldn't find stream information

这个方法用于获取流信息(pFormatCtx->streams)

    // Dump information about file onto standard error
    av_dump_format(pFormatCtx, 0, argv[1], 0);

pFormatCtx->streams 是一个指针数组， 数组的大小是 pFormatCtx->nb_streams，接下来让我们从这里面找到一个视频流

    int i;
    AVCodecContext *pCodecCtxOrig = NULL;
    AVCodecContext *pCodecCtx = NULL;
    
    // Find the first video stream
    videoStream=-1;
    for(i=0; i<pFormatCtx->nb_streams; i++)
      if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO) {
        videoStream=i;
        break;
      }
    if(videoStream==-1)
      return -1; // Didn't find a video stream
    
    // Get a pointer to the codec context for the video stream
    pCodecCtx=pFormatCtx->streams[videoStream]->codec;

这个流的编解码器(codec)信息存在"codec context"中，里面包含这个容器内所有流的编解码器，刚刚我们拿到了它的指针，但是我们仍然需要去找到并打开真正的编解码器。

    AVCodec *pCodec = NULL;
    
    // Find the decoder for the video stream
    pCodec=avcodec_find_decoder(pCodecCtx->codec_id);
    if(pCodec==NULL) {
      fprintf(stderr, "Unsupported codec!\n");
      return -1; // Codec not found
    }
    // Copy context
    pCodecCtx = avcodec_alloc_context3(pCodec);
    if(avcodec_copy_context(pCodecCtx, pCodecCtxOrig) != 0) {
      fprintf(stderr, "Couldn't copy codec context");
      return -1; // Error copying codec context
    }
    // Open codec
    if(avcodec_open2(pCodecCtx, pCodec)<0)
      return -1; // Could not open codec

需要注意的是我们不能直接使用视频流中的 AVCodecContext，所以，我们需要使用 avcodec_copy_context() 把 context 复制到一个新的位置（在分配新内存之后）。

####存储数据

现在我们需要一个地方来存储帧:

    AVFrame *pFrame = NULL;
    
    // Allocate video frame
    pFrame=av_frame_alloc();

现在我们输出24位RGB的 PPM 文件，用ffmpeg把帧转成RGB的原生格式。大多数处理的时候，我们都是想把原格式转换成一个特定的格式，那么我们来分配一个用来转换帧的帧。

    // Allocate an AVFrame structure
    pFrameRGB=av_frame_alloc();
    if(pFrameRGB==NULL)
      return -1;

即使我们分配了帧，当我们转码帧的时候仍需要一个空间，所以我们需要 avpicture_get_size 来获取大小，并分配相应大小的空间。

    uint8_t *buffer = NULL;
    int numBytes;
    // Determine required buffer size and allocate buffer
    numBytes=avpicture_get_size(PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height);
    buffer=(uint8_t *)av_malloc(numBytes*sizeof(uint8_t));

av_malloc是ffmpeg的malloc,它包装malloc以确保内存地址对齐。但它不保证内存泄漏,双释放,或其他malloc问题。

现在，我们使用avpicture_fill来把帧填充到我们新的缓冲区。
AVPicture：AVPicture结构是AVFrame结构的子集，最开始的时候，AVFrame和AVPicture是一样的。

    // Assign appropriate parts of buffer to image planes in pFrameRGB
    // Note that pFrameRGB is an AVFrame, but AVFrame is a superset
    // of AVPicture
    avpicture_fill((AVPicture *)pFrameRGB, buffer, PIX_FMT_RGB24, pCodecCtx->width, pCodecCtx->height);

最后，我们准备读取流数据。

####读取数据

我们现在要做的是读取整个视频流，读取packet，把它解码成帧，得到帧之后把它转换并保存。

    struct SwsContext *sws_ctx = NULL;
    int frameFinished;
    AVPacket packet;
    // initialize SWS context for software scaling
    sws_ctx = sws_getContext(pCodecCtx->width,
        pCodecCtx->height,
        pCodecCtx->pix_fmt,
        pCodecCtx->width,
        pCodecCtx->height,
        PIX_FMT_RGB24,
        SWS_BILINEAR,
        NULL,
        NULL,
        NULL
        );
    
    i=0;
    while(av_read_frame(pFormatCtx, &packet)>=0) {
      // Is this a packet from the video stream?
      if(packet.stream_index==videoStream) {
    	// Decode video frame
        avcodec_decode_video2(pCodecCtx, pFrame, &frameFinished, &packet);
        
        // Did we get a video frame?
        if(frameFinished) {
        // Convert the image from its native format to RGB
            sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
    		  pFrame->linesize, 0, pCodecCtx->height,
    		  pFrameRGB->data, pFrameRGB->linesize);
    	
            // Save the frame to disk
            if(++i<=5)
              SaveFrame(pFrameRGB, pCodecCtx->width, 
                        pCodecCtx->height, i);
        }
      }
        
      // Free the packet that was allocated by av_read_frame
      av_free_packet(&packet);
    }

备注：
packet 理论上可以包含一部分帧和一些其它数据，但是ffmpeg的解析器保证了，我们会获得完整的一个或多个帧。

简单说一下这个过程：av_read_frame()读取packet并把它存储到 AVPacket 结构中。注意，我们只分配packet结构，而ffmpeg为我们分配内部数据，用packet.data获取。之后由 av_free_packet() 进行释放。avcodec_decode_video()把packet转换成帧。然而，解码后可能没有我们需要的全部信息，所以如果有下一帧，则使用avcodec_decode_video()设置frameFinished。最后，使用sws_scale()去把原生格式(native format (pCodecCtx->pix_fmt))转成RGB数据。记住，我们可以把一个AVFrame指针指向一个AVPicture指针。最后，通过帧、高和宽的信息调用我们的 SaveFrame 方法。

现在我们来写 SaveFrame 方法，用它把 RGB 数据写入一个 PPM 文件。

    void SaveFrame(AVFrame *pFrame, int width, int height, int iFrame) {
      FILE *pFile;
      char szFilename[32];
      int  y;
      
      // Open file
      sprintf(szFilename, "frame%d.ppm", iFrame);
      pFile=fopen(szFilename, "wb");
      if(pFile==NULL)
        return;
      
      // Write header
      fprintf(pFile, "P6\n%d %d\n255\n", width, height);
      
      // Write pixel data
      for(y=0; y<height; y++)
        fwrite(pFrame->data[0]+y*pFrame->linesize[0], 1, width*3, pFile);
      
      // Close file
      fclose(pFile);
    }


先做一些标准的文件处理工作，比如打开等等，然后写入 RGB 数据。一次写入文件的一行。PPM文件只是一个把RGB信息写成一个长字符串的文件。如果用过HTML的颜色，就会知道一个红色的屏幕，就是把每个像素都设置成 #ff0000#ff0000......。（没有分割符的二进制存储）头部中包含宽度、高度和RGB的最大大小。

现在回到我们的main函数，一旦我们读完视频流，我们需要清空所有的东西:

    // Free the RGB image
    av_free(buffer);
    av_free(pFrameRGB);
    
    // Free the YUV frame
    av_free(pFrame);
    
    // Close the codecs
    avcodec_close(pCodecCtx);
    avcodec_close(pCodecCtxOrig);
    
    // Close the video file
    avformat_close_input(&pFormatCtx);
    
    return 0;

avcode_alloc_frame 和 av_malloc 分配的内存，我们使用 av_free 进行清理。

现在我们来运行它(linux 或者类型平台)：

    gcc -o tutorial01 tutorial01.c -lavutil -lavformat -lavcodec -lz -lavutil -lm

如果是旧版本的ffmpeg，可能需要去掉 -lavutil 选项运行：

    gcc -o tutorial01 tutorial01.c -lavformat -lavcodec -lz -lm

大部分的图像工具都可以打开 PPM 文件，可以测试一些电影文件。
