#Image I/O的基本使用

Image I/O 接口允许应用读写大多数图片文件格式。它最初是 Core Graphics 框架的一部分，现在已经成为开发者可以自由使用的独立框架。Image I/O 提供了明确的访问图片数据的方式，因为它非常高效，允许方便的访问元数据，并提供色彩管理。

**注:** OS X v10.4 / iOS4 及以后可以使用

##Image I/O的基本使用
Image I/O 框架提供了不透明的数据类型用于从`CGImageSourceRef`读取图片数据以及从`CGImageDestinationRef`写入图片数据。它支持很多种图片格式。同时它也有一下一些特征：

	1. Mac 平台上最快的图片编码和解码速度
	2. 能够递增的加载图片
	3. 高效的缓存

有一下几种方式可以创建`CGImageSourceRef`和`CGImageDestinationRef`：

	1. URLs（CFURLRef）
	2. CFDataRef 和 CFMutableDataRef
	3. CGDataConsumerRef 和 CGDataProviderRef

######在项目中引入Image I/O

```
#import <ImageIO/ImageIO.h>
```

######支持的图片类型
Image I/O 支持大多数的图片格式，比如：JPEG, JPEG2000, RAW, TIFF, BMP, PNG。但也有不支持的。
你可以通过以下两个两个方法获取到 Image I/O 所支持的图片类型（UTI）：

```
CFArrayRef mySourceTypes = CGImageSourceCopyTypeIdentifiers(); //source支持的类型
CFShow(mySourceTypes); // 输出到控制台
CFArrayRef myDestinationTypes = CGImageDestinationCopyTypeIdentifiers(); //destination支持的类型
CFShow(myDestinationTypes);
```

在`MobileCoreServices`框架的`UTType`图片常量部分中，已经对这部分类型进行了定义：

```
let kUTTypeImage: CFString  //The abstract type identifier for image data.
let kUTTypeJPEG: CFString  //The type identifier for a JPEG image.
let kUTTypeJPEG2000: CFString  //The type identifier for a JPEG-2000 image.
let kUTTypeTIFF: CFString  //The type identifier for a TIFF image.
let kUTTypePICT: CFString  //The type identifier for a Quickdraw PICT.
let kUTTypeGIF: CFString  //The type identifier for a GIF image.
let kUTTypePNG: CFString  //The type identifier for a PNG image.
let kUTTypeQuickTimeImage: CFString  //The type identifier for a QuickTime image. Corresponds to the 'qtif' OSType.
let kUTTypeAppleICNS: CFString  //The type identifier for Apple icon data.
let kUTTypeBMP: CFString  //The type identifier for a Windows bitmap.
let kUTTypeICO: CFString  //The type identifier for Windows icon data.
```
 

##创建并使用Image Source
Image Source 对数据存取任务进行了抽象并为你免去了对内存的管理。一个 image source 可以包含多张图片，缩略图，各张图片的属性，以及图片文件。如果你正在处理图片数据并且你的应用所处的系统大于等于 OS X v10.4，那么使用 image sources 来引入图片数据是一个推荐的方式。在创建`CGImageSource`对象后，你可以通过`CGImageSource`对象提供的一系列方法来获取图片、缩略图、图片属性以及其他图片信息等。

######如何创建Image Source

当你在使用 Image I/O 框架时，可能最主要的任务就是从 image source 获取一张图片。下面这个列子会展示如何通过一个路径名来创建一个`CGImageSource`，以及提取图片。在创建 image source 的时候，可以指定图片格式，当然，也可以不指定。
	在创建图片时，指定图片的索引（index）是必须的，但是提供一个属性字典用于描述是否需要创建缩略图以及是否允许缓存却是可选的。
	提供图片索引是因为某些图片文件可能包含多张图片。对于只有一张图片的文件，传`0`就可以了。如果你想知道包含了多少张图片，请调用`CGImageSourceGetCount`方法。

```
CGImageRef MyCreateCGImageFromFile (NSString* path) {
    // Get the URL for the pathname passed to the function.
    NSURL *url = [NSURL fileURLWithPath:path];
    CGImageRef        myImage = NULL;
    CGImageSourceRef  myImageSource;
    
    NSMutableDictionary *options = [NSMutableDictionary dictionary];
    [options setValue:(id)kCFBooleanTrue forKey:(id)kCGImageSourceShouldCache];
    [options setValue:(id)kCFBooleanTrue forKey:(id)kCGImageSourceShouldAllowFloat];
    // Create an image source from the URL.
    myImageSource = CGImageSourceCreateWithURL((CFURLRef)url, (__bridge CFDictionaryRef)options);
    // Make sure the image source exists before continuing
    if (myImageSource == NULL){
        fprintf(stderr, "Image source is NULL.");
        return  NULL;
    }
    // Create an image from the first item in the image source.
    myImage = CGImageSourceCreateImageAtIndex(myImageSource,
                                              0,
                                              NULL);
    
    CFRelease(myImageSource);
    // Make sure the image exists before continuing
    if (myImage == NULL){
        fprintf(stderr, "Image not created from image source.");
        return NULL;
    }
    
    return myImage;
}

```

######创建缩略图
	
有些 image source 包含了缩略图，可以直接拿来用。但是对于那些没有缩略图的， Image I/O 提供了创建它的机会。你甚至可以指定缩略图的大小以及进行`transform`转换。比如：
	
```
CGImageRef MyCreateThumbnailImageFromData (NSData * data, int imageSize) {
    
    CGImageRef myThumbnailImage = NULL;
    
    // Create an image source from NSData; no options.
    CGImageSourceRef myImageSource = CGImageSourceCreateWithData((CFDataRef)data,
                                                NULL);
    // Make sure the image source exists before continuing.
    if (myImageSource == NULL){
        fprintf(stderr, "Image source is NULL.");
        return  NULL;
    }
    
    NSMutableDictionary *options = [NSMutableDictionary dictionary];
    [options setValue:(id)kCFBooleanTrue forKey:(id)kCGImageSourceCreateThumbnailWithTransform];
    [options setValue:(id)kCFBooleanTrue forKey:(id)kCGImageSourceCreateThumbnailFromImageIfAbsent];
    [options setValue:(id)(@(imageSize)) forKey:(id)kCGImageSourceThumbnailMaxPixelSize];
    
    // Create the thumbnail image using the specified options.
    myThumbnailImage = CGImageSourceCreateThumbnailAtIndex(myImageSource, 0, (__bridge CFDictionaryRef)options);
    CFRelease(myImageSource);
    
    // Make sure the thumbnail image exists before continuing.
    if (myThumbnailImage == NULL){
        fprintf(stderr, "Thumbnail image not created from image source.");
        return NULL;
    }
    
    return myThumbnailImage;
}

```

######递增加载图片
	
如果有一张很大的图片，或者加载一张网络图片。你可能需要逐步加载它。逐步加载的过程如下：
	
		1. 创建 `CFData`对象用于累加图片数据。
		2. 通过调用`CGImageSourceCreateIncremental`方法创建一个递增的 image source。
		3. 把接收到的图片数据添加到 `CFData` 对象上。
		4. 调用`CGImageSourceUpdateData`方法，并传入`CFData`对象参数，以及一个用于指示图片是否全部加载完毕的`Bool`值。在任何情况下，传入的`CFData`对象必须是当前累加的data数据。
		5. 如何累加了足够的数据，调用`CGImageSourceCreateImageAtIndex`可以获得一张图片，记得及时释放，因为它是`CF`对象。
		6. 通过调用`CGImageSourceGetStatusAtIndex`方法，检查所有图片数据是否都已经加载完毕。如果完毕，它会返回`kCGImageStatusComplete`，如果没有，继续执行3、4步，直到加载完毕。
		7. 释放资源（image source）

######获取图片属性

```
- (void) imageProperties:(NSData*)data {
    CFDataRef df = (__bridge CFDataRef) data;
    CGImageSourceRef source = CGImageSourceCreateWithData(df, NULL);
    if (source) {
        NSDictionary* props =
        (NSDictionary*) CFBridgingRelease(CGImageSourceCopyPropertiesAtIndex(source, 0, NULL)); // 2
        
        NSString* uti = (NSString*)CGImageSourceGetType(source); //
        
        CFDictionaryRef fileProps = CGImageSourceCopyProperties(source, nil); // 11
    }else{
        NSLog(@"nothing happen.......");
    }
}

```

##使用Image Destinations
Image Destination 对数据存取任务进行了抽象并为你免去了对内存的管理。一个 `image destination ` 代表了一张或者多种图片。它可能包含了每张图片的缩略图和属性。在创建了 `CGImageDestination` 对象后，可以添加图片数据和属性。添加完后，需要调用`CGImageDestinationFinalize`方法。

######设置Image Destination属性

```
- (NSDictionary *)options {
    float compression = 1.0; // Lossless compression if available.
    int orientation = 4; // Origin is at bottom, left.
    
    NSMutableDictionary *options = [NSMutableDictionary dictionary];
    [options setValue:(id)kCFBooleanTrue forKey:(id)kCGImagePropertyHasAlpha];
    [options setValue:(id)(@(orientation)) forKey:(id)kCGImagePropertyOrientation];
    [options setValue:(id)(@(compression)) forKey:(id)kCGImageDestinationLossyCompressionQuality];
    return options;
}
```
更多属性及其设置请阅读：[CGImageProperties](https://developer.apple.com/reference/imageio/1666330-cgimageproperties) 和 [CGImageDestination](https://developer.apple.com/reference/imageio/1666386-cgimagedestination)

######把图片写入Image Destination
首先，需要调用`CGImageDestinationCreateWithURL`、`CGImageDestinationCreateWithData`或`CGImageDestinationCreateWithDataConsumer`创建一个`image destination`对象。传入图片格式参数，图片数量，以及属性字典。
其次，调用`CGImageDestinationAddImage`或`CGImageDestinationAddImageFromSource`方法添加图片。如果图片格式支持多张图片，可以继续添加。
最后，调用`CGImageDestinationFinalize`方法结束添加。结束后不能添加任何数据。

```
void MyAddImageFromData (NSMutableData * mdata, CGImageRef image, NSDictionary *options) {
    
    CGImageDestinationRef myImageDestination = CGImageDestinationCreateWithData((__bridge CFMutableDataRef)mdata, kUTTypePNG, 1, NULL);
    
    CGImageDestinationAddImage(myImageDestination, image, (__bridge CFDictionaryRef)options);
    
    CGImageDestinationFinalize(myImageDestination);
    CFRelease(myImageDestination);
}
```

######创建动态图片


```
void MyAnimatedImage(NSMutableData * mdata, CGImageRef image1, CGImageRef image2) {
    int frameCount = 4;
    
    NSMutableDictionary *fileProperties = [NSMutableDictionary dictionary];
    [fileProperties setObject:@{(id)kCGImagePropertyGIFLoopCount:@(0)} forKey:(id)kCGImagePropertyGIFDictionary];
    
    NSMutableDictionary *frameProperties = [NSMutableDictionary dictionary];
    [frameProperties setObject:@{(id)kCGImagePropertyGIFDelayTime:@(1.0/(double)frameCount)} forKey:(id)kCGImagePropertyGIFDictionary];
    
    CGImageDestinationRef destination = CGImageDestinationCreateWithData((__bridge CFMutableDataRef)mdata, kUTTypeGIF, frameCount, nil);
    
    
    CGImageDestinationSetProperties(destination, (__bridge CFDictionaryRef)fileProperties);
    
    for (int i = 0; i < frameCount; i++) {
        @autoreleasepool {
            if (i%2 == 0) {
                CGImageDestinationAddImage(destination, image1, (__bridge CFDictionaryRef)frameProperties);
            }else{
                CGImageDestinationAddImage(destination, image2, (__bridge CFDictionaryRef)frameProperties);
            }
            
        }
    }
    
    CGImageDestinationFinalize(destination);
}

```

[原文链接](https://developer.apple.com/reference/imageio)


