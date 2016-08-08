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

###第二章：输出到屏幕

> Code: [tutorial02.c](http://dranger.com/ffmpeg/tutorial02.c)

####SDL 和 Video

我们使用 SDL 把图像输出到屏幕上。SDL(Simple Direct Layer)是一个优秀的跨平台多媒体库。你可以从它的官网上获取到你所用操作系统的开发库，你需要这个库来编译这个教程的代码。

SDL有很多绘制图像的方法，其中有一个是用来在屏幕上显示电影的 —— 我们叫它 YUV 视频叠加。YUV(YUV或者YCbCr)，一般来说，YUV是一个模拟格式，而YCbCr是一个数字格式。ffmpeg和SDL的代码和宏定义都是参考YCbCr和YUV，它跟RGB一样是一种用来存储原始图像数据的。简单的说，Y是亮度，U和V是色度。SDL的YUV覆盖（YUV overlay）拿到YUV的原始数据然后显示它。它可以接受4种不同的YUV格式，其中YV12是最快的。另一个YUV格式是YUV420P，它除了U和V数组是切换的，跟YV12格式是一样的。420指的是他的子样本的比例是 4:2:0，也就是说每四个亮度样本有一个色度样本，所以颜色信息是1/4。这样的方式很好的节省了带宽，同时人眼也无法分辨出其中的区别。YUV420P中的'P'是说格式是“二维”的 —— 简单的理解就是 Y, U, V分别在不同的数组中。ffmpeg可以将图像转换成YUV420P，而且有很多视频流已经是用YUV420P了，还有很多可以很容易转换成YUV420P。

那么我们接下来替换掉第一章中的 SaveFrame() 方法，把帧输出到屏幕上。但是，首先我们要先看一下怎么使用 SDL 库。
第一步：include 库，并 init SDL

    #include <SDL.h>
    #include <SDL_thread.h>
    
    if(SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
      fprintf(stderr, "Could not initialize SDL - %s\n", SDL_GetError());
      exit(1);
    }
    
SDL_Init() 本质上是告诉库，我们将要使用它。
SDL_GetError() 是为了方便调试bug。

####Creating a Display

现在我们需要在屏幕上有一个放东西的地方，这个用于显示图片的地方在SDL里叫做surface：

    SDL_Surface *screen;
    
    screen = SDL_SetVideoMode(pCodecCtx->width, pCodecCtx->height, 0, 0);
    if(!screen) {
      fprintf(stderr, "SDL: could not set video mode - exiting\n");
      exit(1);
    }

这里设置了屏幕的高和宽。接下来的选项是屏幕的位深度 - 0 是一个特殊的值，这个值的意思是“跟当前显示的一样”(这个在 OS X 中无效，可见源代码)

现在我们创建一个 YUV 覆盖(YUV overlay)，用来输出视频，并设置我们的 SWSContext 用来将图片数据转成 YUV420：

    SDL_Overlay     *bmp = NULL;
    struct SWSContext *sws_ctx = NULL;
    
    bmp = SDL_CreateYUVOverlay(pCodecCtx->width, pCodecCtx->height, SDL_YV12_OVERLAY, screen);
    
    // initialize SWS context for software scaling
    sws_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height,
			 pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height,
			 PIX_FMT_YUV420P, SWS_BILINEAR, NULL, NULL, NULL);

正如我们之前说的，我们使用YV12来显示图像，并从ffmpeg中取出YUV420数据。

####显示图片

很简单吧！现在我们只需要显示图片了，我们用显示代码替换掉 SaveFrame() 方法。显示图像，我们需要创建一个 AVPicture 结构，然后设置数据指针和linesize，并把它设置给我们的 YUV overlay：

      if(frameFinished) {
        SDL_LockYUVOverlay(bmp);
    
        AVPicture pict;
        pict.data[0] = bmp->pixels[0];
        pict.data[1] = bmp->pixels[2];
        pict.data[2] = bmp->pixels[1];
    
        pict.linesize[0] = bmp->pitches[0];
        pict.linesize[1] = bmp->pitches[2];
        pict.linesize[2] = bmp->pitches[1];
    
        // Convert the image into YUV format that SDL uses
        sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
    	      pFrame->linesize, 0, pCodecCtx->height,
    	      pict.data, pict.linesize);
        
        SDL_UnlockYUVOverlay(bmp);
      }

首先，因为我们需要写 overlay，所以我们先锁定它。这是一个保证之后不会产生问题的好习惯。AVPicture 结构，正如上面展示的，有一个指向4个指针的数组的指针。因为我们现在处理 YUV420P，我们只有3个通道，因为只有3组数据。其它格式可能有第四个指针用来做透明度或者其它的。linesize 跟YUV overlay中像素和球(pitches)变量类似（pitches 是SDL中用来指定宽度线的数据）。所以我们要做的是把 pict.data 中的三个数组放入我们的 overlay，那么当我们写入 pict 的时候，我们实际上把它写入到我们已经分配好空间的 overlay。同样的，我们得到了 linesize 数据，我们把它转换成 PIX_FMT_YUV420P 格式，然后像之前一样使用 sws_scale。

####绘制图像

但是我们仍需要告诉SDL，显示数据我们已经给了它。通常通过一个矩形来设置视频的宽高和缩放比例，并把它传递给方法，这样SDL可以进行缩放，并帮助图形处理器更快的处理缩放:

    SDL_Rect rect;
    
      if(frameFinished) {
        /* ... code ... */
        // Convert the image into YUV format that SDL uses
        sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
                  pFrame->linesize, 0, pCodecCtx->height,
        	      pict.data, pict.linesize);
        
        SDL_UnlockYUVOverlay(bmp);
    	rect.x = 0;
    	rect.y = 0;
    	rect.w = pCodecCtx->width;
    	rect.h = pCodecCtx->height;
    	SDL_DisplayYUVOverlay(bmp, &rect);
      }

现在，我们的视频显示出来了！！

接下来看看SDL的另一个功能：事件系统(Event)。比如在SDL应用中，移动鼠标，它可能发出一个信号，生成一个事件，然后应用程序可以检查这些事件，并处理相应的用户输入。程序可以自定义事件，用来扩充SDL的事件系统。尤其是在多线程编程的时候，事件系统特别有用。在第四章的时候我们会使用到它。在我们的程序中，我们在完成packet处理之后，将轮询事件，处理SDL_QUIT事件，并退出程序:

    SDL_Event       event;
    
        av_free_packet(&packet);
        SDL_PollEvent(&event);
        switch(event.type) {
        case SDL_QUIT:
          SDL_Quit();
          exit(0);
          break;
        default:
          break;
        }

现在我们编译一下，使用SDL最好的编译方式如下：

    gcc -o tutorial02 tutorial02.c -lavformat -lavcodec -lswscale -lz -lm \
    `sdl-config --cflags --libs`

如果gcc包含了正确的sdl的库，sdl-config会打印出正确的flags。在不同的系统上编译可能需要做一些不同的事情，可以查看SDL文档，然后编译运行它。

你可能需要做一些不同的事情来编译它在您的系统上,请检查SDL系统文档。一旦编译,然后运行它。

目前为止视频可以播放，但是它播放非常非常快，实际上我们没有对显示的帧的速度进行控制，目前我们先不控制，第五章的时候我们会做处理。接下来我们要处理很重要的东西：声音！
