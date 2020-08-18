---
layout:     post
title:      "SDWebImage源码阅读(一)WebCache Categories"
subtitle:   ""
date:       2020-08-17
author:     "xurong"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - SDWebImage源码阅读
---



version：5.8.4

从GitHub下载源码后，打开SDWebImage.xcodeproj，可以发现，项目主要分为11个目录。



![SDWebImage](/img/post/SDWebImage.png)

在iOS开发中会经常用到SDWebImage，其中最常用的到的方法就是```- (void)sd_setImageWithURL:(nullable NSURL *)url```了。本次源码阅读，从这个函数进入源码，探索SDWebImage的设计思想。

这个方法位于 WebCache Categories 中，通过比较里面NSButton、UIButton、UIImageView的分类可以看出，他们对外的接口几乎是一致的：

例如在```UIImageView+WebCache.h```中：

```objective-c
- (void)sd_setImageWithURL:(nullable NSURL *)url NS_REFINED_FOR_SWIFT;

- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder NS_REFINED_FOR_SWIFT;

- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options NS_REFINED_FOR_SWIFT;

···
  
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
                   context:(nullable SDWebImageContext *)context
                  progress:(nullable SDImageLoaderProgressBlock)progressBlock
                 completed:(nullable SDExternalCompletionBlock)completedBlock;
```

无论你选择哪一个方法，最终都会调用本分类最后一个也是参数最多的那个方法，在这个方法中，会调用``UIView+WebCache.h``的这个方法：

```objective-c
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock;
```

也就是说，前面几个文件只是对外的高层接口以及一些关于那个类设置图片的一些处理，而真正的核心就在这个函数中。

这里用到了C++的设计思想：[委托构造函数](https://docs.microsoft.com/en-us/cpp/cpp/delegating-constructors?view=vs-2019)，这样做的好处是减少重复代码，使得代码将更易于理解和维护。

现在，我们进入``UIView+WebCache.m``这个文件，进一步阅读源码。

作者通过Runtime的关联对象，动态地给UIView添加了几个属性：

- @property (nonatomic, strong, readonly, nullable) NSURL *sd_imageURL;
- @property (nonatomic, strong, readonly, nullable) NSString *sd_latestOperationKey;
- @property (nonatomic, strong, null_resettable) NSProgress *sd_imageProgress;
- @property (nonatomic, strong, nullable) SDWebImageTransition *sd_imageTransition;
- @property (nonatomic, strong, nullable) id\<SDWebImageIndicator> sd_imageIndicator;



进一步分析核心函数，为了简短代码，把条件编译的部分去掉了，只留下了iOS的部分。

```objective-c
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock {
		//如果有context，直接调用copy方法复制一份。没有则初始化为一个字典。
		//context为NSDictionary类型，其定义为：
		//typedef NSDictionary<SDWebImageContextOption, id> SDWebImageContext;
		if (context) {
        // copy to avoid mutable object
        context = [context copy];
    } else {
        context = [NSDictionary dictionary];
    }
    NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
		//通过 SDWebImageContextSetImageOperationKey 找是否有对应的值。没有的话，就用本类的类名作为值。
    if (!validOperationKey) {
        // pass through the operation key to downstream, which can used for tracing operation or image view class
        validOperationKey = NSStringFromClass([self class]);
        SDWebImageMutableContext *mutableContext = [context mutableCopy];
        mutableContext[SDWebImageContextSetImageOperationKey] = validOperationKey;
        context = [mutableContext copy];
    }
    self.sd_latestOperationKey = validOperationKey;
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    self.sd_imageURL = url;
                
		//如果没有选项也没有标记为延迟加载占位图，那么直接加载占位图。	
    if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
        });
    }
        
		//有url的情况
    if (url) {
    	···
    } else {
      	//没有url时回到主线程并报错：图片地址为空
        dispatch_main_async_safe(^{
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:SDWebImageErrorInvalidURL userInfo:@{NSLocalizedDescriptionKey : @"Image url is nil"}];
                completedBlock(nil, nil, error, SDImageCacheTypeNone, YES, url);
            }
        });
    }
```



```if (url) {}```里面的代码比较长，这里单独拿出来分析。

```objective-c
				// reset the progress
				// 重置下载进度
        NSProgress *imageProgress = objc_getAssociatedObject(self, @selector(sd_imageProgress));
        if (imageProgress) {
            imageProgress.totalUnitCount = 0;
            imageProgress.completedUnitCount = 0;
        }
        
				//处理manager
        SDWebImageManager *manager = context[SDWebImageContextCustomManager];
        if (!manager) {
            manager = [SDWebImageManager sharedManager];
        } else {
            // remove this manager to avoid retain cycle (manger -> loader -> operation -> context -> manager)
          	// 移除这个manager避免循环引用
            SDWebImageMutableContext *mutableContext = [context mutableCopy];
            mutableContext[SDWebImageContextCustomManager] = nil;
            context = [mutableContext copy];
        }
        
				//定义combinedProgressBlock，作为operation的progress参数
        SDImageLoaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            if (imageProgress) {
                imageProgress.totalUnitCount = expectedSize;
                imageProgress.completedUnitCount = receivedSize;
            }
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };

        @weakify(self);
				//至此，我们发现，真正的下载操作是manager的loadImageWithURL函数中完成的
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            @strongify(self);
            if (!self) { return; }
            // if the progress not been updated, mark it to complete state
          	// 如果进度没有被更新，把它标记为完成状态。
            if (imageProgress && finished && !error && imageProgress.totalUnitCount == 0 && imageProgress.completedUnitCount == 0) {
                imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!self) { return; }
                if (!shouldNotSetImage) {
                    [self sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, data, error, cacheType, finished, url);
                }
            };
            
            // case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set
            // OR
            // case 1b: we got no image and the SDWebImageDelayPlaceholder is not set
          	//有两种情况，一是我们拿到了一张图片，设置了SDWebImageAvoidAutoSetImage标记；或者没拿到图片，并且也没设置SDWebImageDelayPlaceholder标记。就是不需要设置图片，直接回到主线程并调用callCompletedBlockClojure，并且return本函数。
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }
            
            dispatch_main_async_safe(^{
                [self sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:cacheType imageURL:imageURL];
                callCompletedBlockClojure();
            });
        }];
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
```



总结。SDWebImage使用了委托构造函数的思想，使得构造灵活多变，用户无需关注每一个参数，只需要按自己业务需要选择。context的设计和处理，起到了承上启下的作用。代码中多处使用了@weakify(self) @strongify(self)，dispatch_main_async_safe，可见作者对防止循环引用和在主线程刷新UI的重视。这也是我们设计和编写代码时值得学习的地方。