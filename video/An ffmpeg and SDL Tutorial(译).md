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

    gcc -o tutorial02 tutorial02.c -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`

如果gcc包含了正确的sdl的库，sdl-config会打印出正确的flags。在不同的系统上编译可能需要做一些不同的事情，可以查看SDL文档，然后编译运行它。

你可能需要做一些不同的事情来编译它在您的系统上,请检查SDL系统文档。一旦编译,然后运行它。

目前为止视频可以播放，但是它播放非常非常快，实际上我们没有对显示的帧的速度进行控制，目前我们先不控制，第五章的时候我们会做处理。接下来我们要处理很重要的东西：声音！

###第三章：播放音频

> Code: [tutorial03.c](http://dranger.com/ffmpeg/tutorial03.c)

####Audio

现在我们想播放声音。SDL也提供了播放声音的方法。SDL_OpenAudio() 函数是用来打开音频设备的。它需要一个 SDL_AudioSpec 结构体的参数，其中包含所有我们将要输出的音频信息。

在讲解如何设置它之前，我们先解释一下电脑是如何处理音频的，数字音频是由一长串连续的采样构成的。每一个样本代表音频波形中的一个值。音频被记录在某一个特定的采样率，简单的说就是每秒播放的样本数。举个例子，采样率是 22050 和 44100，分别是无线电和CD的采样率。另外，大多数音频有多个声道用来播放环绕立体声，举个例子，如果在立体声中采样，样本在同一时间将会是两个。当我们从视频文件中获取数据的时候，我们并不知道会得到多少样本，但是ffmpeg不会只给我们部分样本，这也意味着它不会分离一个立体声样本。

SDL播放音频的方法：
设置音频选项：采样率（SDL的结构中被称作"freq"），声道的数量等等，通常我们还会设置一个回调函数和用户数据。当我们开始播放音频，SDL将会不断的调用这个回调函数，让它往音频缓冲区填充一定数量字节的数据。在我们给 SDL_AudioSpec 放完这些数据后，我们调用 SDL_OpenAudio() 函数，它将打开音频设备，并给我们返回另一个 AudioSpec 结构体。这些规格会被我们使用 —— 我们无法保证得到我们所要的。

####设置音频

让这些保留在你大脑里一会儿，因为我们没有真正的获取任何有关于音频流的数据。现在我们回到我们获取视频流的代码中，在这找到我们需要的音频流。

    // Find the first video stream
    videoStream=-1;
    audioStream=-1;
    for(i=0; i < pFormatCtx->nb_streams; i++) {
      if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO && videoStream < 0) {
        videoStream=i;
      }
      if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_AUDIO && audioStream < 0) {
        audioStream=i;
      }
    }
    if(videoStream==-1)
      return -1; // Didn't find a video stream
    if(audioStream==-1)
      return -1;

从流中的AVCodecContext可以得到我们想要的所有信息，就像从视频流中获取一样:

    AVCodecContext *aCodecCtxOrig;
    AVCodecContext *aCodecCtx;
    
    aCodecCtxOrig = pFormatCtx->streams[audioStream]->codec;

如果你还记得之前的教程，我们仍然需要打开音频的 codec。这很简单:

    AVCodec         *aCodec;
    
    aCodec = avcodec_find_decoder(aCodecCtx->codec_id);
    if(!aCodec) {
      fprintf(stderr, "Unsupported codec!\n");
      return -1;
    }
    // Copy context
    aCodecCtx = avcodec_alloc_context3(aCodec);
    if(avcodec_copy_context(aCodecCtx, aCodecCtxOrig) != 0) {
      fprintf(stderr, "Couldn't copy codec context");
      return -1; // Error copying codec context
    }
    /* set up SDL Audio here */
    
    avcodec_open2(aCodecCtx, aCodec, NULL);

编解码器上下文中包含所有信息，我们需要设置我们的音频:

    wanted_spec.freq = aCodecCtx->sample_rate;
    wanted_spec.format = AUDIO_S16SYS;
    wanted_spec.channels = aCodecCtx->channels;
    wanted_spec.silence = 0;
    wanted_spec.samples = SDL_AUDIO_BUFFER_SIZE;
    wanted_spec.callback = audio_callback;
    wanted_spec.userdata = aCodecCtx;
    
    if(SDL_OpenAudio(&wanted_spec, &spec) < 0) {
      fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
      return -1;
    }

参数解释:
 - freq: 采样率。
 - format: 传给SDL的格式。"S16SYS" 中 "S" 代表签名(signed)，"16"代表每一个样本长度为16位(bits)，"SYS"表示endian-order将取决于你的系统。这是 avcodec_decode_audio2 将给我们的音频格式。
 - channels: 声道个数。
 - silence: 消除噪音。通常这个值为0。
 - samples: 音频缓冲区大小。一般在 512 到 8192 之间，ffplay 使用的是 1024。
 - callback: 回调函数。稍后我们会讨论更多关于回调函数的话题。
 - userdata: SDL调用回调函数时，会给我们一个指向用户数据的空指针（void pointer），我们希望它知道我们的 codec context。 

最后我们用 SDL_OpenAudio 打开音频。

####队列

现在我们准备好从流中获取音频数据流了。但我们能对这些数据做什么呢？我们将从视频文件中获取连续的packets，但是同时SDL会调用回调函数。解决方案是创建一种我们能通过audio_callback填充音频包的全局的结构体。所以我们要创建一个packets的队列。ffmpeg 提供了一个结构来帮我们处理这个：AVPacketList，就是packet的linked list，在这就是我们的队列结构：

    typedef struct PacketQueue {
      AVPacketList *first_pkt, *last_pkt;
      int nb_packets;
      int size;
      SDL_mutex *mutex;
      SDL_cond *cond;
    } PacketQueue;

我们要知道 nb_packets 与 packet->size 的大小是不一样的。在这我们有一个互斥锁和一个状态变量。这是因为SDL的音频处理是一个单独运行的线程。如果我们不锁定队列，我们可能把数据弄混乱。我们将看到如何实现队列。每个程序员都应该知道如何写一个队列，但是我们包括它，你从SDL的函数中学习。

首先我们创建一个初始化队列的函数:

    void packet_queue_init(PacketQueue *q) {
      memset(q, 0, sizeof(PacketQueue));
      q->mutex = SDL_CreateMutex();
      q->cond = SDL_CreateCond();
    }

接下来创建一个填充队列的方法:

    int packet_queue_put(PacketQueue *q, AVPacket *pkt) {
    
        AVPacketList *pkt1;
        if(av_dup_packet(pkt) < 0) {
          return -1;
        }
        pkt1 = av_malloc(sizeof(AVPacketList));
        if (!pkt1)
          return -1;
        pkt1->pkt = *pkt;
        pkt1->next = NULL;
    
        SDL_LockMutex(q->mutex);
    
        if (!q->last_pkt)
            q->first_pkt = pkt1;
        else
            q->last_pkt->next = pkt1;
    
        q->last_pkt = pkt1;
        q->nb_packets++;
        q->size += pkt1->pkt.size;
        SDL_CondSignal(q->cond);
    
        SDL_UnlockMutex(q->mutex);
        return 0;
    }

SDL_LockMutex() 锁住队列，然后可以向队列中添加数据，然后 SDL_CondSignal() 发出一个信号来通知我们的函数(如果它正在等待)，通过条件变量告诉它有数据可以处理，然后解锁。

下面是相应的函数，注意怎么使用 SDL_CondWait() 函数块（等待获取数据）。

    int quit = 0;
    
    static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block) {
      AVPacketList *pkt1;
      int ret;
      
      SDL_LockMutex(q->mutex);
      
      for(;;) {
        
        if(quit) {
          ret = -1;
          break;
        }

        pkt1 = q->first_pkt;
        if (pkt1) {
          q->first_pkt = pkt1->next;
          if (!q->first_pkt)
	    q->last_pkt = NULL;
          
          q->nb_packets--;
          q->size -= pkt1->pkt.size;
          *pkt = pkt1->pkt;
          av_free(pkt1);
          ret = 1;
          break;
        } else if (!block) {
          ret = 0;
          break;
        } else {
          SDL_CondWait(q->cond, q->mutex);
        }
      }
      SDL_UnlockMutex(q->mutex);
      return ret;
    }

我们封装了一个无限循环的函数，如果想停止它，需要获取到数据。我们避免通过SDL的 SDL_CondWait() 进行无限循环。基本上，所有的等待条件都是等待来自SDL_CondSignal() (或者 SDL_CondBroadcast())的一个信号，然后继续。然而，看起来我们被困在我们的锁里 —— 如果我们保持锁，我们的put方法将不能往队列中put任何东西。无论如何，SDL_CondWait() 总是解锁，处理，然后在尝试加锁。

####In Case of Fire

我们还有一个全局退出的变量，我们需要确保设置退出信号，否则线程将会永远持续，直到kill -9。

    SDL_PollEvent(&event);
    switch(event.type) {
        case SDL_QUIT:
            quit = 1;

我们将退出flag设置成1。

####读取Packets

接下来设置我们的队列:

    PacketQueue audioq;
    main() {
        ...
        avcodec_open2(aCodecCtx, aCodec, NULL);
    
        packet_queue_init(&audioq);
        SDL_PauseAudio(0);

SDL_PauseAudio()最后启动音频设备。如果没有数据（或者是出错），则他是静音状态。

所以，我们设置了队列，现在我们开始给队列数据包，进到我们的数据包读取循环:

    while(av_read_frame(pFormatCtx, &packet)>=0) {
      // Is this a packet from the video stream?
      if(packet.stream_index==videoStream) {
        // Decode video frame
        ....
      } else if(packet.stream_index==audioStream) {
        packet_queue_put(&audioq, &packet);
      } else {
        av_free_packet(&packet);
      }
    }

注意，packet放到队列后不释放，在我们解码后进行释放。

####获取Packets

现在我们在队列中创建一个audio_callback函数来获取packets。这个回调函数的结构是 void callback(void *userdata, Uint8 *stream, int len)，把userdata指针传给SDL，stream是我们写入音频数据的缓冲区，len 是这个缓冲区的大小，下面是代码:

    void audio_callback(void *userdata, Uint8 *stream, int len) {
    
      AVCodecContext *aCodecCtx = (AVCodecContext *)userdata;
      int len1, audio_size;
    
      static uint8_t audio_buf[(AVCODEC_MAX_AUDIO_FRAME_SIZE * 3) / 2];
      static unsigned int audio_buf_size = 0;
      static unsigned int audio_buf_index = 0;
    
      while(len > 0) {
        if(audio_buf_index >= audio_buf_size) {
          /* We have already sent all our data; get more */
          audio_size = audio_decode_frame(aCodecCtx, audio_buf, sizeof(audio_buf));
          if(audio_size < 0) {
	    /* If error, output silence */
	    audio_buf_size = 1024;
	    memset(audio_buf, 0, audio_buf_size);
          } else {
	    audio_buf_size = audio_size;
          }
          audio_buf_index = 0;
        }
        len1 = audio_buf_size - audio_buf_index;
        if(len1 > len)
          len1 = len;
        memcpy(stream, (uint8_t *)audio_buf + audio_buf_index, len1);
        len -= len1;
        stream += len1;
        audio_buf_index += len1;
      }
    }

这个循环从我们写的另一个方法中获取数据，audio_decode_frame()，在中介缓冲区中存储结果，尝试写入 len 字节长度的流，如果没有足够的数据，则获取更多的数据，audio_buf的大小是ffmpeg获取到的最大音频真的大小的1.5倍，这样给了我们一个很好的缓冲。

####解码音频

最后是音频解码:

    int audio_decode_frame(AVCodecContext *aCodecCtx, uint8_t *audio_buf, int buf_size) {
    
      static AVPacket pkt;
      static uint8_t *audio_pkt_data = NULL;
      static int audio_pkt_size = 0;
      static AVFrame frame;
    
      int len1, data_size = 0;
    
      for(;;) {
        while(audio_pkt_size > 0) {
          int got_frame = 0;
          len1 = avcodec_decode_audio4(aCodecCtx, &frame, &got_frame, &pkt);
          if(len1 < 0) {
	    /* if error, skip frame */
	    audio_pkt_size = 0;
	    break;
          }
          audio_pkt_data += len1;
          audio_pkt_size -= len1;
          data_size = 0;
          if(got_frame) {
	    data_size = av_samples_get_buffer_size(NULL, aCodecCtx->channels, frame.nb_samples, aCodecCtx->sample_fmt, 1);
	    assert(data_size <= buf_size);
	    memcpy(audio_buf, frame.data[0], data_size);
          }
          if(data_size <= 0) {
	    /* No data yet, get more frames */
	    continue;
          }
          /* We have data, return it and come back for more later */
          return data_size;
        }
        if(pkt.data)
          av_free_packet(&pkt);
    
        if(quit) {
          return -1;
        }
    
        if(packet_queue_get(&audioq, &pkt, 1) < 0) {
          return -1;
        }
        audio_pkt_data = pkt.data;
        audio_pkt_size = pkt.size;
      }
    }

在这整个函数的最后，我们调用 packet_queue_get()。从队列中获取packet，并保存其信息。一旦我们获取了packet，就可以调用 avcodec_decode_audio4() 函数，就像 avcodec_decode_video() 函数一样，除了这种情况，一个packet钟可能包含多个frame，所以需要多次调用它，来从packet中获取data。一旦我们得到了 frame，拷贝它到音频缓冲区，并保证 data_size 小于我们的音频缓冲区。记得计算 audio_buf，因为SDL给了一个8位的整数缓冲区(int buffer)，同时ffmpeg给了一个16位的整数缓冲区(int buffer)。还应该注意 len1 和 data_size 的不同，len1是我们使用的packet大小，而data_size返回的是原始数据的总大小。

当我们得到一些数据，如果我们仍然要从队列中获取数据，或者已经获取完了，我们要立即返回。如果我们仍然有很多的packet需要处理，我们需要保存它给以后用。如果我们已经完成了一个packet，我们最终要释放它。

编译：

    gcc -o tutorial03 tutorial03.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`

Hooray! 现在视频播放仍然很快，但是音频的播放很正常。

我们几乎已经准备好开始同步音频和视频了，但我们需要先做一个小的重构。音频的队列和播放使用不同的线程处理会更好：它保证了代码的可管理性和模块化。在我们开始音视频同步之前，需要先让我们的代码更容易处理。下一章：多线程

###第四章：多线程

> Code: [tutorial04.c](http://dranger.com/ffmpeg/tutorial04.c)

####综述

上一章我们利用SDL的音频功能添加了音频支持。SDL启动一个线程，每次需要音频时回调我们定义的函数。现在，我们对视频做的同样的事情。这使代码更加模块化，也更容易使用（当我们做同步的时候就会知道了），好，现在开始。

首先看看我们的main函数都做了什么：通过事件循环运行，读取packets，视频解码。那么我们可以对它进行一些拆分：一个负责解码packets的线程，这些packets被添加到队列，并被相应的音视频线程读取，音频线程我们已经处理好了，因为要显示视频，所以视频线程处理相对负责一些。我们要在主循环中添加显示代码，而不仅仅是在每次循环中显示视频，我们要把视频显示整合到事件循环中。一个思路是解码视频，并把它的帧存到另一个队列中，然后生成一个自定义事件(FF_REFRESH_EVENT)，把它添加到事件系统中，然后当我们在事件循环遇到这个事件的时候，我们就把队列中的下一帧视频显示出来。流程图如下:

     ________ audio  _______      _____
    |        | pkts |       |    |     | to spkr
    | DECODE |----->| AUDIO |--->| SDL |-->
    |________|      |_______|    |_____|
        |  video     _______
        |   pkts    |       |
        +---------->| VIDEO |
     ________       |_______|   _______
    |       |          |       |       |
    | EVENT |          +------>| VIDEO | to mon.
    | LOOP  |----------------->| DISP. |-->
    |_______|<---FF_REFRESH----|_______|


用事件循环控制视频显示的目的是使用 SDL_Delay 线程。这样我们就可以精确的控制下一帧显示在屏幕上的事件。那么我们就可以让视频在正确的时间上显示正确的画面。

####简化代码

我们有音频和视频编解码器的信息，以及相应的添加队列和缓冲区，以及其它的东西，其实这就是对视频的描述，所以我们创建一个大的结构体（VideoState）

    typedef struct VideoState {
    
      AVFormatContext *pFormatCtx;
      int             videoStream, audioStream;
      AVStream        *audio_st;
      AVCodecContext  *audio_ctx;
      PacketQueue     audioq;
      uint8_t         audio_buf[(AVCODEC_MAX_AUDIO_FRAME_SIZE * 3) / 2];
      unsigned int    audio_buf_size;
      unsigned int    audio_buf_index;
      AVPacket        audio_pkt;
      uint8_t         *audio_pkt_data;
      int             audio_pkt_size;
      AVStream        *video_st;
      AVCodecContext  *video_ctx;
      PacketQueue     videoq;
    
      VideoPicture    pictq[VIDEO_PICTURE_QUEUE_SIZE];
      int             pictq_size, pictq_rindex, pictq_windex;
      SDL_mutex       *pictq_mutex;
      SDL_cond        *pictq_cond;
      
      SDL_Thread      *parse_tid;
      SDL_Thread      *video_tid;
    
      char            filename[1024];
      int             quit;
    } VideoState;

它包括：格式上下文(format context)、音视频流的索引，以及相应的AVStream对象，音频的缓冲区，用于存储解码后视频帧的队列和缓冲区，VideoPicture自定义的结构体，两个额外的线程指针，退出flag和文件名。

现在回到主函数，看看怎么用VideoState结构体:

    int main(int argc, char *argv[]) {
    
      SDL_Event       event;
    
      VideoState      *is;
    
      is = av_mallocz(sizeof(VideoState));

av_mallocz() 为我们分配内存并把它置零。

然后初始化显示缓冲区(pictq)的锁，直到事件循环调用我们的显示函数，才从pictq中拉取预解码的帧。同时，我们的视频解码器会给它放入信息，我们并不知道它们处理的先后顺序。这是一个典型的竞争条件。所以我们要在启动任何线程之前，分配它。我们也把文件名复制到 VideoState 结构体中。

    av_strlcpy(is->filename, argv[1], sizeof(is->filename));
    
    is->pictq_mutex = SDL_CreateMutex();
    is->pictq_cond = SDL_CreateCond();

av_strlcpy 是ffmpeg的一个函数，它比 strncpy 多了一些边界判断。

####第一个线程

现在开始我们的线程，看看它怎么工作:

    schedule_refresh(is, 40);
    
    is->parse_tid = SDL_CreateThread(decode_thread, is);
    if(!is->parse_tid) {
      av_free(is);
      return -1;
    }

schedule_refresh是后面定义的一个函数。它的基本工作是在指定毫秒后给系统发送一个FF_REFRESH_EVENT事件。然后事件循环在遇到它的时候，进行视频显示刷新。但现在，我们先来看 SDL_CreateThread() 函数。

SDL_CreateThread() 创建一个能访问原处理中所有内存的线程，并在我们给定的函数中运行线程。这里我们调用decode_thread()，并与VideoState结构关联。该函数的上半部分就是打开文件，找到音视频流的索引，把格式上下文(format context)放到VideoState中。得到流索引之后，调用我们定义的另一个方法（stream_component_open()），因为音视频编解码器设置很像，所以这样我们可以重用一些代码。

在 stream_component_open() 函数中找到编解码器，设置音频选项，将重要的信息保存到VideoState中，并运行音视频线程。这里，我们也会放入一些其它选项，比如不自动获取codec，而是强制指定codec。下面是代码:

    int stream_component_open(VideoState *is, int stream_index) {
    
      AVFormatContext *pFormatCtx = is->pFormatCtx;
      AVCodecContext *codecCtx;
      AVCodec *codec;
      SDL_AudioSpec wanted_spec, spec;
    
      if(stream_index < 0 || stream_index >= pFormatCtx->nb_streams) {
        return -1;
      }
    
      codec = avcodec_find_decoder(pFormatCtx->streams[stream_index]->codec->codec_id);
      if(!codec) {
        fprintf(stderr, "Unsupported codec!\n");
        return -1;
      }
    
      codecCtx = avcodec_alloc_context3(codec);
      if(avcodec_copy_context(codecCtx, pFormatCtx->streams[stream_index]->codec) != 0) {
        fprintf(stderr, "Couldn't copy codec context");
        return -1; // Error copying codec context
      }
    
      if(codecCtx->codec_type == AVMEDIA_TYPE_AUDIO) {
        // Set audio settings from codec info
        wanted_spec.freq = codecCtx->sample_rate;
        /* ...etc... */
        wanted_spec.callback = audio_callback;
        wanted_spec.userdata = is;
        
        if(SDL_OpenAudio(&wanted_spec, &spec) < 0) {
          fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
          return -1;
        }
      }
      if(avcodec_open2(codecCtx, codec, NULL) < 0) {
        fprintf(stderr, "Unsupported codec!\n");
        return -1;
      }
    
      switch(codecCtx->codec_type) {
      case AVMEDIA_TYPE_AUDIO:
        is->audioStream = stream_index;
        is->audio_st = pFormatCtx->streams[stream_index];
        is->audio_ctx = codecCtx;
        is->audio_buf_size = 0;
        is->audio_buf_index = 0;
        memset(&is->audio_pkt, 0, sizeof(is->audio_pkt));
        packet_queue_init(&is->audioq);
        SDL_PauseAudio(0);
        break;
      case AVMEDIA_TYPE_VIDEO:
        is->videoStream = stream_index;
        is->video_st = pFormatCtx->streams[stream_index];
        is->video_ctx = codecCtx;
        
        packet_queue_init(&is->videoq);
        is->video_tid = SDL_CreateThread(video_thread, is);
        is->sws_ctx = sws_getContext(is->video_st->codec->width, is->video_st->codec->height,
				 is->video_st->codec->pix_fmt, is->video_st->codec->width,
				 is->video_st->codec->height, PIX_FMT_YUV420P,
				 SWS_BILINEAR, NULL, NULL, NULL
				 );
        break;
      default:
        break;
      }
    }

这跟之前的代码基本一样，只是它包含了音频和视频。注意不是aCodecCtx，我们为音频回调在VideoState中设置了userdata。还把音视频流保存在 audio_st 和 video_st 中。创建一个跟音频队列一样的视频队列:

    SDL_PauseAudio(0);
    break;

    /* ...... */

    is->video_tid = SDL_CreateThread(video_thread, is);

SDL_CreateThread()的用法跟前面的 SDL_PauseAudio() 一样。接下来，我们回到video_thread()函数。

在此之前，我们先看decode_thread()。它是一个for循环，读取 packet 并把它存到正确的队列中:

      for(;;) {
        if(is->quit) {
          break;
        }
        // seek stuff goes here
        if(is->audioq.size > MAX_AUDIOQ_SIZE ||
           is->videoq.size > MAX_VIDEOQ_SIZE) {
          SDL_Delay(10);
          continue;
        }
        if(av_read_frame(is->pFormatCtx, packet) < 0) {
          if((is->pFormatCtx->pb->error) == 0) {
    	    SDL_Delay(100); /* no error; wait for user input */
    	    continue;
          } else {
    	    break;
          }
        }
        // Is this a packet from the video stream?
        if(packet->stream_index == is->videoStream) {
          packet_queue_put(&is->videoq, packet);
        } else if(packet->stream_index == is->audioStream) {
          packet_queue_put(&is->audioq, packet);
        } else {
          av_free_packet(packet);
        }
      }

现在我们有了音视频队列的最大size，我们增加了对读取错误的确认。格式上下文（format context）包含了一个ByteIOContext结构体，称作 pb。ByteIOContext结构体中保持了所有低级别的文件信息。

在循环之后，我们有等待程序结束或者通知它已经结束的代码。这段代码说明了如何推送事件，之后视频显示我们也会用到它。

      while(!is->quit) {
        SDL_Delay(100);
      }
    
     fail:
      if(1){
        SDL_Event event;
        event.type = FF_QUIT_EVENT;
        event.user.data1 = is;
        SDL_PushEvent(&event);
      }
      return 0;

使用SDL的SDL_USEREVENT自定义用户事件，第一个用户事件应该是SDL_USEREVENT，下一个就是SDL_USEREVENT+1，以此类推。FF_QUIT_EVENT我们用SDL_USEREVENT+1定义。我们也可以传递用户数据，如果我们愿意，也可以把传递的指针指向VideoState结构体。最后调用 SDL_PushEvent()。

####获取帧: video_thread

编解码器准备好之后，我们开始视频线程。这个线程从视频队列中读取packets，视频解码成帧，然后调用queue_picture函数把处理好的帧放到图像队列上（picture queue）:

    int video_thread(void *arg) {
      VideoState *is = (VideoState *)arg;
      AVPacket pkt1, *packet = &pkt1;
      int frameFinished;
      AVFrame *pFrame;
    
      pFrame = av_frame_alloc();
    
      for(;;) {
        if(packet_queue_get(&is->videoq, packet, 1) < 0) {
          // means we quit getting packets
          break;
        }
        // Decode video frame
        avcodec_decode_video2(is->video_st->codec, pFrame, &frameFinished, packet);
    
        // Did we get a video frame?
        if(frameFinished) {
          if(queue_picture(is, pFrame) < 0) {
    	    break;
          }
        }
        av_free_packet(packet);
      }
      av_free(pFrame);
      return 0;
    }

我们把avcodec_decode_video2函数替换到这，只需要替换一些参数。例如，我们有AVStream存储我们的VideoState，所以我们从这获取codec。我们持续的从视频队列中获取packets，只到退出或者出错。

####帧队列

让我们看看存储解码后的帧，P帧到图像队列中的函数。图像队列是一个SDL覆盖（overlay），我们需要在这里面转换帧。我们做了一个用来把数据存储到图像队列的结构体:

    typedef struct VideoPicture {
      SDL_Overlay *bmp;
      int width, height; /* source height & width */
      int allocated;
    } VideoPicture;

VideoState中含有缓冲区，我们需要分配SDL_Overlay。

使用这个队列，我们有两个指针（写索引和读索引）。我们在缓冲区中跟踪一些图片。在写入队列之前，我们要先等缓冲区清空，那样才有空间用来存储VideoPicture。然后检查我们是否已经给我们的写索引分配好覆盖（overlay），如果没有我们需要分配一定的空间。如果窗口大小改变，我们也需要重新分配缓冲区。

    int queue_picture(VideoState *is, AVFrame *pFrame) {
    
      VideoPicture *vp;
      int dst_pix_fmt;
      AVPicture pict;
    
      /* wait until we have space for a new pic */
      SDL_LockMutex(is->pictq_mutex);
      while(is->pictq_size >= VIDEO_PICTURE_QUEUE_SIZE && !is->quit) {
        SDL_CondWait(is->pictq_cond, is->pictq_mutex);
      }
      SDL_UnlockMutex(is->pictq_mutex);
    
      if(is->quit)
        return -1;
    
      // windex is set to 0 initially
      vp = &is->pictq[is->pictq_windex];
    
      /* allocate or resize the buffer! */
      if(!vp->bmp ||
         vp->width != is->video_st->codec->width ||
         vp->height != is->video_st->codec->height) {
        SDL_Event event;
    
        vp->allocated = 0;
        alloc_picture(is);
        if(is->quit) {
          return -1;
        }
      }

下面看看 alloc_picture() 函数:

    void alloc_picture(void *userdata) {
    
      VideoState *is = (VideoState *)userdata;
      VideoPicture *vp;
    
      vp = &is->pictq[is->pictq_windex];
      if(vp->bmp) {
        // we already have one make another, bigger/smaller
        SDL_FreeYUVOverlay(vp->bmp);
      }
      // Allocate a place to put our YUV image on that screen
      SDL_LockMutex(screen_mutex);
      vp->bmp = SDL_CreateYUVOverlay(is->video_st->codec->width,
				 is->video_st->codec->height,
				 SDL_YV12_OVERLAY,
				 screen);
      SDL_UnlockMutex(screen_mutex);
      vp->width = is->video_st->codec->width;
      vp->height = is->video_st->codec->height;  
      vp->allocated = 1;
    }

你可能看到了 SDL_CreateYUVOverlay 函数，我们把它从主循环中拿了出来。现在这段代码的含义很明显。因为两个线程不能同时往屏幕上输出信息，所以我们用互斥锁给它加锁。我们在VideoPicture结构体中保存了宽度和高度，是为了确保视频大小不被改变。

现在我们分配好了YUV覆盖（overlay），准备接受图片了。我们回到 queue_picture，看看怎么用代码把帧复制到覆盖（overlay）中:

    int queue_picture(VideoState *is, AVFrame *pFrame) {
    
      /* Allocate a frame if we need it... */
      /* ... */
      /* We have a place to put our picture on the queue */
    
      if(vp->bmp) {
    
        SDL_LockYUVOverlay(vp->bmp);
        
        dst_pix_fmt = PIX_FMT_YUV420P;
        /* point pict at the queue */
    
        pict.data[0] = vp->bmp->pixels[0];
        pict.data[1] = vp->bmp->pixels[2];
        pict.data[2] = vp->bmp->pixels[1];
        
        pict.linesize[0] = vp->bmp->pitches[0];
        pict.linesize[1] = vp->bmp->pitches[2];
        pict.linesize[2] = vp->bmp->pitches[1];
        
        // Convert the image into YUV format that SDL uses
        sws_scale(is->sws_ctx, (uint8_t const * const *)pFrame->data,
	      pFrame->linesize, 0, is->video_st->codec->height,
	      pict.data, pict.linesize);
        
        SDL_UnlockYUVOverlay(vp->bmp);
        /* now we inform our display thread that we have a pic ready */
        if(++is->pictq_windex == VIDEO_PICTURE_QUEUE_SIZE) {
          is->pictq_windex = 0;
        }
        SDL_LockMutex(is->pictq_mutex);
        is->pictq_size++;
        SDL_UnlockMutex(is->pictq_mutex);
      }
      return 0;
    }

####显示视频

这就是我们的视频线程。除了 schedule_refresh() 函数，我们已经处理完了，下面我们看看 schedule_refresh():

    /* schedule a video refresh in 'delay' ms */
    static void schedule_refresh(VideoState *is, int delay) {
      SDL_AddTimer(delay, sdl_refresh_timer_cb, is);
    }

SDL_AddTimer() 是一个SDL函数，它能够在指定的毫秒数之后回调我们指定的函数。可以使用这个函数去控制视频更新，每次调用这个函数的时候，它会设置将要触发某一事件的定时器，触发事件后main()函数将调用从图像队列中拉取一帧，并显示的函数。

首先我们看看怎么发送事件:

    static Uint32 sdl_refresh_timer_cb(Uint32 interval, void *opaque) {
      SDL_Event event;
      event.type = FF_REFRESH_EVENT;
      event.user.data1 = opaque;
      SDL_PushEvent(&event);
      return 0; /* 0 means stop timer */
    }

这是事件触发。FF_REFRESH_EVENT 事件是用 SDL_USEREVENT + 1 定义的。有一点要注意的是，当我们 return 0 的时候，SDL会停掉定时器，那么就不会在处理callback

现在我们发出一个 FF_REFRESH_EVENT 事件，我们需要处理它:

    for(;;) {
    
      SDL_WaitEvent(&event);
      switch(event.type) {
      /* ... */
      case FF_REFRESH_EVENT:
        video_refresh_timer(event.user.data1);
        break;

它调用我们写的 video_refresh_timer 函数，video_refresh_timer 会从图像队列中拉取相应的数据:

    void video_refresh_timer(void *userdata) {
    
      VideoState *is = (VideoState *)userdata;
      VideoPicture *vp;
      
      if(is->video_st) {
        if(is->pictq_size == 0) {
          schedule_refresh(is, 1);
        } else {
          vp = &is->pictq[is->pictq_rindex];
          /* Timing code goes here */
    
          schedule_refresh(is, 80);
          
          /* show the picture! */
          video_display(is);
          
          /* update queue for next picture! */
          if(++is->pictq_rindex == VIDEO_PICTURE_QUEUE_SIZE) {
	    is->pictq_rindex = 0;
          }
          SDL_LockMutex(is->pictq_mutex);
          is->pictq_size--;
          SDL_CondSignal(is->pictq_cond);
          SDL_UnlockMutex(is->pictq_mutex);
        }
      } else {
        schedule_refresh(is, 100);
      }
    }

目前为止，这是一个非常简单的函数：当我们要做什么的时候就从队列中拉取数据，当下一帧视频可以显示的时候设置我们的定时器，然后调用 video_display 函数显示视频，然后累加队列计数器，并减小队列的大小。你可能注意到这个函数里我们并没有对 vp 做任何事情，稍后我们会进行用到它，当我们开始做音视频同步的时候，我们将会使用它来获取时间轴数据。现在我们看“timing code here”这个位置，这个部分，我们要计算多久显示下一帧视频，并把这个时间告诉 schedule_refresh() 函数。现在我们先写死这个值是80，针对每一个视频我们可以去一点点尝试确认这个值，但这是非常愚蠢的，之后我们会讲解如何自动得到这个值。

基本完工了，我们就剩下最后一件事情：显示视频。现在我们来看 video_display 函数:

    void video_display(VideoState *is) {
    
      SDL_Rect rect;
      VideoPicture *vp;
      float aspect_ratio;
      int w, h, x, y;
      int i;
    
      vp = &is->pictq[is->pictq_rindex];
      if(vp->bmp) {
        if(is->video_st->codec->sample_aspect_ratio.num == 0) {
          aspect_ratio = 0;
        } else {
          aspect_ratio = av_q2d(is->video_st->codec->sample_aspect_ratio) *
	    is->video_st->codec->width / is->video_st->codec->height;
        }
        if(aspect_ratio <= 0.0) {
          aspect_ratio = (float)is->video_st->codec->width /
	    (float)is->video_st->codec->height;
        }
        h = screen->h;
        w = ((int)rint(h * aspect_ratio)) & -3;
        if(w > screen->w) {
          w = screen->w;
          h = ((int)rint(w / aspect_ratio)) & -3;
        }
        x = (screen->w - w) / 2;
        y = (screen->h - h) / 2;
        
        rect.x = x;
        rect.y = y;
        rect.w = w;
        rect.h = h;
        SDL_LockMutex(screen_mutex);
        SDL_DisplayYUVOverlay(vp->bmp, &rect);
        SDL_UnlockMutex(screen_mutex);
      }
    }

因为我们的屏幕可以是任意大小的，所以我们需要动态计算出我们想要多大的视频矩形大小。所以，首先我们需要计算出视频的高宽比（width/height）。一些编解码器会有奇数采样的宽高比(odd sample aspect ratio)，这只是一个简单的单像素或者采样的宽高比。因为在codec context中用像素来测量高度和宽度的值，实际上宽高比就等于采样的宽高比的倍数。还有一些 codec 的宽高比是0，这表示每一个像素都是1*1。然后我们按比例在我们的屏幕中缩放。把视频放在中心，调用SDL_DisplayYUVOverlay()，确保屏幕互斥的访问它。
好了，现在我们还要用VideoStruct重写音频编码，但基本没有变化，可以看看那些简单的示例代码。现在我们要做最后一件事，修改我们的回调函数，去调用FFmpeg内部的回调（“退出”的回调函数）:

    VideoState *global_video_state;
    
    int decode_interrupt_cb(void) {
      return (global_video_state && global_video_state->quit);
    }

在main函数中设置 global_video_state to the big struct in main()。

好了，现在我们来编译吧:

    gcc -o tutorial04 tutorial04.c -lavutil -lavformat -lavcodec -lswscale -lz -lm `sdl-config --cflags --libs`

享受音视频不同步的视频吧，^0^! 下一章我们将做一个可以正常工作的播放器!
