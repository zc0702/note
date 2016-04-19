目前视频网站越来越多，为了兼容各种硬件设备，视频网站大都选择使用mp4格式的视频文件。

为了在文件全部加载完成前播放视频文件，播放器通常需要视频文件的 metadata，也就是 mp4 中的 moov atom，
但并不是所有的 mp4 都把 moov atom 放到文件的前面，以保证播放器可以先加载到 metadata。

所以如果想要在互联网上播放 mp4，就需要将 mp4 进行流化处理，也就是将 mp4 的 metadata 放到文件前，这样
播放器就可以在读取完 metadata 后，就开始进行播放。

ffmpeg 提供 qt-faststart 工具，用来做这个处理。

1. 直接使用 qt-faststart
  eg. 
  > qt-faststart old.mp4 new.mp4

2. 放入 ffmpeg 的 movflags 中
  eg.
  > ffmpeg -i old.mp4 -c:a copy -c:v copy -movflags faststart new.mp4
  
了解 mp4 movie atom: [[https://www.adobe.com/devnet/video/articles/mp4_movie_atom.html]]
