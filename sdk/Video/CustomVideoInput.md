# 自定义视频采集

<!-- tabs:start -->

## ** Web **

### 实现方法

balabala……    

### 示例代码

balabala……    


## ** Windows **

## ** Android **

实时音视频应用中，通常会使用默认的音视频采集模块，在部分场景中，需要自定义视频采集。
本文介绍如何实现通过外部视频源，实现自定义视频采集。

### 实现方法

Android sdk支持`yuv420p`系列的外部源以及`rgba`、`abgr`，`rgb565`等`rgba`系列数据，作为自定义视频源。    

### 开发注意事项

```
public interface UcloudRTCDataProvider {

    //0-3 表示转换类型
    //4-7 表示rgba_stride的宽度的倍数
    //8-11 表示yuv_stride宽度移位数
    //12-15 表示uv左移位数
    public static final int RGBA_TO_I420 = 0x01001040;
    public static final int ABGR_TO_I420 = 0x01001041;
    public static final int BGRA_TO_I420 = 0x01001042;
    public static final int ARGB_TO_I420 = 0x01001043;
    public static final int RGB24_TO_I420 = 0x01001034;
    public static final int RGB565_TO_I420 = 0x01001025;
    public static final int NV21 = 0x01001090;
    public static final int I420 = 0x01001099;

    /**
     * 供sdk采集外部数据进行推流的方法，由sdk 使用者来提供数据，具体使用方式请参考rgb转yuv使用说明
     * @param params  需要传入3个参数 ，参数顺序不可颠倒，参数1：类型，例如RGBA_TO_I420，参数2：宽 参数3： 高
     * @return  扩展推流的原始rgb数据或者yuv420p，NV21
     */
    ByteBuffer provideRGBData(List<Integer> params);


    /**
     * 释放申请的buffer
     */
    void releaseBuffer();
}

```
设置外部采集模式参数

```
 //设置sdk 外部扩展模式及其采集的帧率，同时sdk内部会自动调整初始码率和最小码率
 //扩展模式只支持720p的分辨率及以下，若要自定义更高分辨率，请联系Ucloud商务定制，否则sdk会抛出异常，终止运行。
 sdkEngine.setVideoProfile(UCloudRtcSdkVideoProfile.UCLOUD_RTC_SDK_VIDEO_PROFILE_EXTEND.extendParams(30,640,480));
// sdkEngine.setVideoProfile(UCloudRtcSdkVideoProfile.matchValue(mVideoProfile));
//设置捕获模式，二选一
//        //普通摄像头捕获方式，与扩展模式二选一
//        UCloudRtcSdkEnv.setCaptureMode(
//                UCloudRtcSdkCaptureMode.UCLOUD_RTC_CAPTURE_MODE_LOCAL);
        //rgb数据捕获，与普通捕获模式二选一
//        UCloudRtcSdkEnv.setCaptureMode(
//                UCloudRtcSdkCaptureMode.UCLOUD_RTC_CAPTURE_MODE_EXTEND);
```

调用范例，具体内容请参考demo源码内RoomActivity，需要根据自己的实际情况来，范例只是做个参考
```
//生产者消费者队列
 private ArrayBlockingQueue<RGBSourceData> mQueue = new ArrayBlockingQueue(2);

//生产者往队列中push数据，这里只是用两张Bitmap decode 后的结果作为数据源模拟功能，如果有直接的数据源则可以直接调用provideRGBData方法
       Runnable imgTask = new Runnable() {
            @Override
            public void run() {
                    while(startCreateImg){
                        try{
//                        synchronized (mUCloudRTCDataProvider){
//                            if(mQueue.size() != 0){
//                                mUCloudRTCDataProvider.wait();
//                            }
//                            if(mQueue.size() == 0){
                            RGBSourceData sourceData;
                            Bitmap bitmap = null;
                            int type;
                            if(mPictureFlag < 25){
                                BitmapFactory.Options options = new BitmapFactory.Options();
                                options.inPreferredConfig = Bitmap.Config.RGB_565;
                                bitmap= BitmapFactory.decodeResource(getResources(),R.mipmap.timg2,options);
                                type = UcloudRTCDataProvider.RGB565_TO_I420;
                            }
                            else{
                                BitmapFactory.Options options = new BitmapFactory.Options();
                                options.inPreferredConfig = Bitmap.Config.ARGB_8888;
                                bitmap= BitmapFactory.decodeResource(getResources(),R.mipmap.img3,options);
                                type = UcloudRTCDataProvider.RGBA_TO_I420;
                            }

                            if(++mPictureFlag >50)
                                mPictureFlag = 0;
                            if(bitmap != null){
                                sourceData = new RGBSourceData(bitmap,bitmap.getWidth(),bitmap.getHeight(),type);
                                mQueue.put(sourceData);
                                Log.d(TAG, "create bitmap: " + bitmap + "count :" + memoryCount.incrementAndGet());
                            }
//                            }
//                        }

                        }catch (Exception e){
                            e.printStackTrace();
                        }
                    }
                    //这里在回收一遍 防止队列不阻塞了在destroy以后又产生了bitmap没回收
                    while(mQueue.size() != 0 ){
                        RGBSourceData rgbSourceData = mQueue.poll();
                        if(rgbSourceData != null){
                            recycleBitmap(rgbSourceData.getSrcData());
                        }
                    }
            }
        };

      if(UCloudRtcSdkEnv.getCaptureMode() == UcloudRtcSdkCaptureMode.UCLOUD_RTC_CAPTURE_MODE_EXTEND &&
                (mRole == UCloudRtcSdkStreamRole.UCLOUD_RTC_SDK_STREAM_ROLE_BOTH ||
                        mRole == UCloudRtcSdkStreamRole.UCLOUD_RTC_SDK_STREAM_ROLE_PUB)){
            mCreateImgThread = new Thread(imgTask);
            mCreateImgThread.start();
            //把接口实现提供给sdk
            UCloudRtcSdkEngine.onRGBCaptureResult(mUCloudRTCDataProvider);
        }
            
        }

```

```
 		//sdk作为消费者消费数据
    private UcloudRTCDataProvider mUCloudRTCDataProvider = new UcloudRTCDataProvider() {
        private ByteBuffer cacheBuffer;
        private RGBSourceData rgbSourceData;

        @Override
        public ByteBuffer provideRGBData(List<Integer> params) {
            rgbSourceData = mQueue.poll();
            if(rgbSourceData == null){
//                mUCloudRTCDataProvider.notify();
                return null;
            }else{
                params.add(rgbSourceData.getType());
                params.add(rgbSourceData.getWidth());
                params.add(rgbSourceData.getHeight());
                if(cacheBuffer == null){
                    cacheBuffer = sdkEngine.getNativeOpInterface().
                            createNativeByteBuffer(4096*2160*4);
                }else{
                    cacheBuffer.clear();
                }
                cacheBuffer.limit(rgbSourceData.getWidth()*rgbSourceData.getHeight()*4);
                rgbSourceData.getSrcData().copyPixelsToBuffer(cacheBuffer);
                recycleBitmap(rgbSourceData.getSrcData());
                return cacheBuffer;
            }
        }

        public void releaseBuffer(){
            if(rgbSourceData != null && !rgbSourceData.getSrcData().isRecycled()){
                rgbSourceData.getSrcData().recycle();
            }
            if(cacheBuffer != null){
                sdkEngine.getNativeOpInterface().realeaseNativeByteBuffer(cacheBuffer);
            }
        }
    };
```

## ** iOS **

### 实现方法

balabala……    

### 示例代码

balabala……    


## ** macOS **

### 实现方法

balabala……    

### 示例代码

balabala……    



<!-- tabs:end -->

## 开发注意事项

balabala……  

