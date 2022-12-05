# CMSampleBuffer 总结



### Overview

CMSampleBuffer 是 Core Foundation 的对象，系统将 Audio 和 Video 数据封装成 CMSampleBuffer， 在音视频相关的场景中，被处理和传递。一个 CMSampleBuffer 的实例包含的数据可能是压缩的（视频 h264, 音频 aac ??）数据，或者非压缩的（视频 yuv， 音频 pcm）数据。

CMSampleBuffer 会关联一些 attachments， 共 2 类型的 attachments：

* buffer-level： buffer 自身关联的 attachments
* sample-level： buffer 内部每个 sample 关联的 attachments

> Buffer-level attachments provide information about the buffer as a whole, such as playback speed and actions to perform upon consuming the buffer. You can read and write buffer-level attachments using the APIs described in [CMAttachment](https://developer.apple.com/documentation/coremedia/cmattachment?language=objc) and the keys listed under [Sample Attachment Keys](https://developer.apple.com/documentation/coremedia/cmsamplebuffer/sample\_attachment\_keys?language=objc).

> Each individual sample in a buffer may provide attachments that include information such as timestamps and video frame dependencies. You read and write sample-level attachments using the [CMSampleBufferGetSampleAttachmentsArray](https://developer.apple.com/documentation/coremedia/1489189-cmsamplebuffergetsampleattachmen?language=objc) function.

### Video SampleBuffer

#### Uncompressed Video SampleBuffer

![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16702426830571670242682221.png)

通过 [AVCaptureSession](https://developer.apple.com/documentation/avfoundation/avcapturesession?language=objc) 采集摄像头得到保存 Video 数据的 CMSampleBuffer。

```swift
func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
    let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer)
    let pts = CMSampleBufferGetPresentationTimeStamp(sampleBuffer)
    //...
}
```

通过 [ReplayKit](https://developer.apple.com/documentation/replaykit?language=objc) 采集屏幕信息得到保存 Video 数据的 CMSampleBuffer。

```swift
import ReplayKit

class SampleHandler: RPBroadcastSampleHandler {
    override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer, with sampleBufferType: RPSampleBufferType) {
        switch sampleBufferType {
        case RPSampleBufferType.video:
            onReceiveVideo(sampleBuffer: sampleBuffer)
        default:
            break
        }
    }
    func onReceiveVideo(sampleBuffer: CMSampleBuffer) {
        var rotation : Int = 0
        if let orientationAttachment = CMGetAttachment(sampleBuffer, key: RPVideoSampleOrientationKey as CFString, attachmentModeOut: nil) as? NSNumber {
            if let orientation = CGImagePropertyOrientation(rawValue: orientationAttachment.uint32Value) {
                switch orientation {
                case .up,    .upMirrored:    rotation = 0
                case .down,  .downMirrored:  rotation = 180
                case .left,  .leftMirrored:  rotation = 90
                case .right, .rightMirrored: rotation = 270
                default:   break
                }
            }
        }
        
        guard let pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer) else {
            return
        }
        
        let timestamp = CMSampleBufferGetPresentationTimeStamp(sampleBuffer).seconds * 1000
        if rotation == 90 || rotation == 270 {
            var newPixelBuffer: CVPixelBuffer?
            let error = CVPixelBufferCreate(kCFAllocatorDefault,
                                            CVPixelBufferGetHeight(pixelBuffer),
                                            CVPixelBufferGetWidth(pixelBuffer),
                                            CVPixelBufferGetPixelFormatType(pixelBuffer),
                                            nil,
                                            &newPixelBuffer)
            guard error == kCVReturnSuccess else {
                return
            }
            
            let ciImage = CIImage(cvPixelBuffer: pixelBuffer).oriented(rotation == 270 ? .left : .right)
            let context = CIContext(options: nil)
            context.render(ciImage, to: newPixelBuffer!)
            
            guard let newPixelBuffer = newPixelBuffer else {
                return
            }
            //handle newPixelBuffer...
            print(newPixelBuffer)
        }
        //handle pixelBuffer...
        print(pixelBuffer)
    }
}

```

可以看到 ReplayKit 从获取的 SampleBuffer 的 Attachments 中读取了 buffer 视频方向的信息。

```swift
if let orientationAttachment = CMGetAttachment(sampleBuffer, key: RPVideoSampleOrientationKey as CFString, attachmentModeOut: nil) as? NSNumber {
    if let orientation = CGImagePropertyOrientation(rawValue: orientationAttachment.uint32Value) {
        switch orientation {
        case .up,    .upMirrored:    rotation = 0
        case .down,  .downMirrored:  rotation = 180
        case .left,  .leftMirrored:  rotation = 90
        case .right, .rightMirrored: rotation = 270
        default:   break
        }
    }
}
```

#### Compressed Video SampleBuffer

![](https://fastly.jsdelivr.net/gh/yxibng/filebed@main/img/images/blog/16702426320591670242631892.png)

使用 [Video Toolbox](https://developer.apple.com/documentation/videotoolbox?language=objc) 可以 yuv, rgb 视频帧编码成 h264， h265，编码后的视频帧 SampleBuffer 的形式输出。

//TODO: parse SampleBuffer to get raw h264 stream

### Audio SampleBuffer

#### Uncompressed Audio SampleBuffer

通过 [AVCaptureSession](https://developer.apple.com/documentation/avfoundation/avcapturesession?language=objc) 采集麦克风数据得到保存 Audio 数据的 CMSampleBuffer。

通过 [ReplayKit](https://developer.apple.com/documentation/replaykit?language=objc) 采集 App 声音或者麦克风声音得到保存 Audio 数据的 CMSampleBuffer。

下面的代码，从 ReplayKit 中获取 CMSampleBuffer， 对音频数据处理后得到了单声道的pcm。

```objectivec
const int bufferSamples = 16000;
size_t dataPointerSize = bufferSamples;

- (void)receiveAudioSampleBuffer:(CMSampleBufferRef)sampleBuffer {
    
    OSStatus err = noErr;
    CMBlockBufferRef audioBuffer = CMSampleBufferGetDataBuffer(sampleBuffer);
    if (!audioBuffer) {
        return;
    }
    
    size_t totalBytes;
    char *samples;
    err = CMBlockBufferGetDataPointer(audioBuffer, 0, NULL, &totalBytes, &samples);
    if (!totalBytes || err != noErr) {
        return;
    }
    
    CMAudioFormatDescriptionRef format = CMSampleBufferGetFormatDescription(sampleBuffer);
    const AudioStreamBasicDescription *description = CMAudioFormatDescriptionGetStreamBasicDescription(format);
    
    //多少个帧（双声道包含两个采样，一个左声道一个右声道）
    size_t totalFrames = totalBytes / description->mBytesPerFrame;
    //多少个采样（双声道 = totalFrames *2）
    size_t totalSamples = totalBytes / (description->mBitsPerChannel / 8);
    UInt32 channels = description->mChannelsPerFrame;
    
    memset(dataPointer, 0, sizeof(int16_t) * bufferSamples);
    err = CMBlockBufferCopyDataBytes(audioBuffer,
                                     0,
                                     totalBytes,
                                     dataPointer);
    
    CMTime pts = CMSampleBufferGetPresentationTimeStamp(sampleBuffer);
    float timestamp = CMTimeGetSeconds(pts) * 1000;
    
    BOOL isFloat = description->mFormatFlags & kAudioFormatFlagIsFloat;
    BOOL isBigEndian = description->mFormatFlags & kAudioFormatFlagIsBigEndian;
    BOOL isInterleaved = !(description->mFormatFlags & kAudioFormatFlagIsNonInterleaved);
    
    // big endian to little endian
    size_t bytesPerSample = description->mBitsPerChannel / 8;
    if (isBigEndian) {
        for (int i = 0; i < totalSamples; i++) {
            uint8_t* p = (uint8_t*)dataPointer + i * bytesPerSample;
            for (int j = 0; j < bytesPerSample / 2; j++) {
                uint8_t tmp;
                tmp = p[j];
                p[j] = p[bytesPerSample - j -1];
                p[bytesPerSample -j -1] = tmp;
            }
        }
    }
    
    // float to int
    if (isFloat) {
        float* floatData = (float*)dataPointer;
        int16_t* intData = (int16_t*)dataPointer;
        for (int i = 0; i < totalSamples; i++) {
            float tmp = floatData[i] * 32767;
            intData[i] = (tmp >= 32767) ?  32767 : tmp;
            intData[i] = (tmp < -32767) ? -32767 : tmp;
        }
        totalBytes = totalSamples * sizeof(int16_t);
    }
    
    //分离出单声道
    if (channels > 1) {
        if (isInterleaved) {
            int bytesPerFrame = (*description).mBytesPerFrame;
            for (int i = 0; i < totalFrames; i++) {
                memmove(dataPointer + i, (uint8_t *)dataPointer + i * bytesPerFrame, sizeof(int16_t));
            }
        }
    }
    
    //目前只是用了一个声道的数据
    size_t srcLength = totalBytes / channels;
    uint8_t *srcData = (uint8_t *)dataPointer;
    //TODO: save or send to server
}
```
