####失败原因

pod setup 会将 Specs 项目的 master clone 到 ~/.cocoapods/repos 目录，但由于 Specs 项目较大，clone 过程很慢，所以容易导致 pod setup 失败。

####处理方式

手动将 Specs clone到计算机上，并把它放到相应的文件夹。

####步骤
>step 1. clone Specs项目（https://github.com/CocoaPods/Specs，注意，不能直接下载项目的 zip 包，需要 clone）
>step 2. 将 Specs 放入 ~/.cocoapods/repos 目录下
>step 3. 将 Specs 改名为 master
>step 4. 运行 pod setup
