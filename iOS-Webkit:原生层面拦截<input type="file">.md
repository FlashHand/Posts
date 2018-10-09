#iOS-Webkit:原生层面拦截\<input type="file">

##0x00 为什么要这么做
有个朋友让我帮他把网站嵌入到APP里，那个网站的一个页面包含了一个上传文件的按钮，他们只能接受后缀为jpg的图片，但是iOS相册里png图片是常见的。
然而他们对客户端要求必须有这种限制。所以客户端需要想办法，把用户挑选出来的图片转为jpg格式，后缀改为jpg再上传。

初步考虑后大致想出了三个路子：
1. 能不能在UIImageViewCongtroller里做method-swizzling.
2. 能不能注入JS，通过JS拦截来转换图片格式。
3. 在WebKit里做method-swizzling,毕竟总得通过一个回调把片传给WebView.

方法1无法走通，因为不知道找不到可行的方法，但是可以利用UIImagePickerControllerDelegate，这个由方法3描述。
方法2也是走不通，原因就是浏览器层面不支持。

然后就是方法3了。这是一个完全可行的方法。

##0x01 寻找拦截的入口
在xcode里的文档并没有指出UIImagePickerController的delegate是谁。
这里可以从WebKit2文档里找到的，就是[WKFileUploadPanel](https://opensource.apple.com/source/WebKit2/WebKit2-7601.1.46.9/UIProcess/ios/forms/WKFileUploadPanel.mm.auto.html).
不过我用了一个自下而上的方法来寻找，swizzle了viewDidAppear方法。
```
- (void)swizzled_viewDidAppear {
    [self swizzled_viewDidAppear];
    if ([self isKindOfClass:[UIImagePickerController class]]) {
            UIImagePickerController *that = self;
            NSLog(@"%@",that.delegate);
        }
```
打印出来的便是<WKFileUploadPanel: 0x10bdc3710>,google一下即可发现WKFileUploadPanel.mm源码，后面需要用到。
##0x02 置换回调方法
根据文档，和对mediaInfo的打印结果确认，只有didFinishPickingMediaWithInfo包含了图片和后缀信息，是适用于重写图片信息的。
```
__TVOS_PROHIBITED @protocol UIImagePickerControllerDelegate<NSObject>
@optional
// The picker does not dismiss itself; the client dismisses it in these callbacks.
// The delegate will receive one or the other, but not both, depending whether the user
// confirms or cancels.
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingImage:(UIImage *)image editingInfo:(nullable NSDictionary<NSString *,id> *)editingInfo NS_DEPRECATED_IOS(2_0, 3_0);
- (void)imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info;
- (void)imagePickerControllerDidCancel:(UIImagePickerController *)picker;
```
```
//MediaInfo
[0]	(null)	@"UIImagePickerControllerMediaType" : @"public.image"
[1]	(null)	@"UIImagePickerControllerOriginalImage" : (no summary)
[2]	(null)	@"UIImagePickerControllerReferenceURL" : @"assets-library://asset/asset.JPG?id=7C993AB5-2881-4261-BAB4-BB0559E8C65C&ext=JPG"
[3]	(null)	@"UIImagePickerControllerImageURL" : @"file:///private/var/mobile/Containers/Data/Application/F024EE26-344E-4A83-AD0F-8E4BCEFA18AA/tmp/E1167ADD-779C-4496-B4E3-5AD45A0B2478.jpeg"
```
然后参照WKFileUploadPanel.mm源码进行swizzling.
先贴出swizzle后的方法实现
```
- (void)swizzled_imagePickerController:(UIImagePickerController *)picker didFinishPickingMediaWithInfo:(NSDictionary<NSString *,id> *)info{
    NSMutableDictionary *dict = [[NSMutableDictionary alloc]initWithDictionary:info];
    //UIImagePickerControllerReferenceURL设为空是关键一步。
    [dict setValue:nil forKey:@"UIImagePickerControllerReferenceURL"];
    NSURL *oriPath = [dict valueForKey:@"UIImagePickerControllerImageURL"];
    NSString *imgfolder =[NSString stringWithFormat:@"file://%@",fast_PathInDocumentDirectory(@"tmpImgs")];
    NSString *imgsPath = fast_PathInDocumentDirectory(@"tmpImgs");
    if ([oriPath.absoluteString hasSuffix:@"png"]){
        UIImage *oriImg = [dict valueForKey:@"UIImagePickerControllerOriginalImage"];
        NSData* data = UIImageJPEGRepresentation(oriImg,0);
        NSString * shortUUID = [NSUUID shortUUIDString];
        NSString *finalPath = [imgsPath stringByAppendingPathComponent:[NSString stringWithFormat:@"%@.jpg",shortUUID]];
        [data writeToFile:finalPath atomically:YES];
        UIImage *jpgImg = [UIImage imageWithData:data];
        NSURL *jpgPath = [@"file://"stringByAppendingString:finalPath].mj_url;
        [dict setObject:jpgImg forKey:@"UIImagePickerControllerOriginalImage"];
        [dict setObject:jpgPath forKey:@"UIImagePickerControllerImageURL"];
    }else if([oriPath.absoluteString hasSuffix:@"jpeg"]){
        UIImage *oriImg = [dict valueForKey:@"UIImagePickerControllerOriginalImage"];
        NSData* data = UIImageJPEGRepresentation(oriImg,0);
        NSString * shortUUID = [NSUUID shortUUIDString];
        NSString *finalPath = [imgsPath stringByAppendingPathComponent:[NSString stringWithFormat:@"%@.jpg",shortUUID]];
        [data writeToFile:finalPath atomically:YES];
        UIImage *jpgImg = [UIImage imageWithData:data];
        NSURL *jpgPath = [@"file://"stringByAppendingString:finalPath].mj_url;
        [dict setObject:jpgImg forKey:@"UIImagePickerControllerOriginalImage"];
        [dict setObject:jpgPath forKey:@"UIImagePickerControllerImageURL"];
    }
    [self swizzled_imagePickerController:picker didFinishPickingMediaWithInfo:dict];
}
```
fast_PathInDocumentDirectory的实现
```
NSString *fast_PathInDocumentDirectory(NSString *fileName)
{
    NSArray *documentDirectories =
    NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,
                                        NSUserDomainMask, YES);

    NSString *documentDirectory = [documentDirectories objectAtIndex:0];

    return [documentDirectory stringByAppendingPathComponent:fileName];
}
```
[NSUUID shortUUIDString] 是来源于[UUIDShortener](https://github.com/kishikawakatsumi/UUIDShortener)的扩展方法
###对置换方法的解释
在结合mediaInfo从源码寻找的过程中发现了突破口，实际上在发现这个方法之前我都不确定是否可以实现对图片的置换。

因为这个方法之前，我单纯尝试性地修改了mediaInfo的里相关的后缀和图片格式。但这样都无法把图片上传给webview。

所以必须再深入一点去了解源码，确认是否可行。

下面是WKFileUploadPanel中上传图片的方法：
```
- (void)_uploadItemFromMediaInfo:(NSDictionary *)info successBlock:(void (^)(_WKFileUploadItem *))successBlock failureBlock:(void (^)(void))failureBlock
{
    NSString *mediaType = [info objectForKey:UIImagePickerControllerMediaType];

    // For videos from the existing library or camera, the media URL will give us a file path.
    if (UTTypeConformsTo((CFStringRef)mediaType, kUTTypeMovie)) {
        NSURL *mediaURL = [info objectForKey:UIImagePickerControllerMediaURL];
        if (![mediaURL isFileURL]) {
            LOG_ERROR("WKFileUploadPanel: Expected media URL to be a file path, it was not");
            ASSERT_NOT_REACHED();
            failureBlock();
            return;
        }

        successBlock(adoptNS([[_WKVideoFileUploadItem alloc] initWithFileURL:mediaURL]).get());
        return;
    }

    if (!UTTypeConformsTo((CFStringRef)mediaType, kUTTypeImage)) {
        LOG_ERROR("WKFileUploadPanel: Unexpected media type. Expected image or video, got: %@", mediaType);
        ASSERT_NOT_REACHED();
        failureBlock();
        return;
    }

    UIImage *originalImage = [info objectForKey:UIImagePickerControllerOriginalImage];
    if (!originalImage) {
        LOG_ERROR("WKFileUploadPanel: Expected image data but there was none");
        ASSERT_NOT_REACHED();
        failureBlock();
        return;
    }

    // If we have an asset URL, try to upload the native image.
    NSURL *referenceURL = [info objectForKey:UIImagePickerControllerReferenceURL];
    if (referenceURL) {
        [self _uploadItemForImage:originalImage withAssetURL:referenceURL successBlock:successBlock failureBlock:failureBlock];
        return;
    }

    // Photos taken with the camera will not have an asset URL. Fall back to a JPEG representation.
    [self _uploadItemForJPEGRepresentationOfImage:originalImage successBlock:successBlock failureBlock:failureBlock];
}
```
上面这些代码，主要关注一下几个部分
1. [info objectForKey:UIImagePickerControllerMediaURL]是无需修改的
2. [info objectForKey:UIImagePickerControllerOriginalImage]不能为空，而且需要修改为转换后的图片。
3. 然后就是最后几行涉及到referenceURL的代码:
```
NSURL *referenceURL = [info objectForKey:UIImagePickerControllerReferenceURL];
if (referenceURL) {
        [self _uploadItemForImage:originalImage withAssetURL:referenceURL successBlock:successBlock failureBlock:failureBlock];
        return;
    }

    // Photos taken with the camera will not have an asset URL. Fall back to a JPEG representation.
    [self _uploadItemForJPEGRepresentationOfImage:originalImage successBlock:successBlock failureBlock:failureBlock];
```

这里有个比较巧妙的过程，按注释的意思是，相机是不会有asset URL,即referenceURL会为空，所以这里不需要传asset URL，直接传图片对象即可。

对于web页面来说，只需要我的图片就行了。

回到自己写的置换的方法中去：

[dict setValue:nil forKey:@"UIImagePickerControllerReferenceURL"];为什么是关键一步呢。

就是因为我希望WKFileUploadPanel以为从相册来的图片也是从相机来的。

最后一步执行了
```
[self swizzled_imagePickerController:picker didFinishPickingMediaWithInfo:dict];
```
这里就是调用原始的WKFileUploadPanel对UIImagePickerDelegate的实现了，被修改后的mediaInfo被传给webview，实现了置换。


