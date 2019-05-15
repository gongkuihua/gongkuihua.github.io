---
layout: post
title: "WKWebView的应用"
subtitle: 'WKWebView的应用详细使用使用方法,包括自定义UA,动态注入js,以及闪退问题"'
author: "gongkuihua"
header-style: text
tags:
- WKWebView
- UA
- js
---


# WKWebView的应用
在ios12.2后UIWebView系统不在给予支持了,要求更新wkwebview,12.2后![image.png](https://upload-images.jianshu.io/upload_images/3889208-71a230210797271a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
而且在12.0后一些网页视频不能播放了,找了很久原因也没有找到,最终的解决方法就是把UIwebView换成WkwebView解决视频播放问题,今天就将自己在集成的时候遇到的问题和迁移方法进行提供,供大家参考.
对于系统在11.0到11.3之间,网页打开地图闪退,在viewDidLoad添加以下代码解决
	
	//解决在11.0到11.3之间的系统打开地图闪退问题
	if (iOS_SYSTEM >= 11.0 && iOS_SYSTEM <= 11.3) {
	        setenv("JSC_useJIT", "false", 0);
	    }
# 1.初始化wkweb
### 初始化

	#import <WebKit/WebKit.h>
	WKWebView *cq_newWkWeb (void) {
	    
	    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
	    configuration.allowsAirPlayForMediaPlayback = YES;//是否运行airplay自动播放媒体功能
	    configuration.allowsInlineMediaPlayback = YES;// 是使用h5的视频播放器在线播放, 还是使用原生播放器全屏播放
	    configuration.selectionGranularity = WKSelectionGranularityCharacter;//解决WKWebView的内存泄露
	    configuration.dataDetectorTypes = UIDataDetectorTypePhoneNumber;
	    configuration.suppressesIncrementalRendering = YES;
	    configuration.processPool = [[WKProcessPool alloc] init];
	    
	    NSString *version= [UIDevice currentDevice].systemVersion;
	    if(version.doubleValue >= 10.0) {
	        // 该属性只支持10+系统
	        configuration.dataDetectorTypes = WKDataDetectorTypePhoneNumber;
	        configuration.mediaTypesRequiringUserActionForPlayback = WKAudiovisualMediaTypeNone;//允许网页自动播放视频和自动播放音频
	    }else if (version.doubleValue <= 10.0 && version.doubleValue >=9.0){
	        configuration.requiresUserActionForMediaPlayback = YES;//允许网页自动播放视频和自动播放音频
	    }
	    
	    // 创建设置对象
	    WKPreferences *preference = [[WKPreferences alloc]init];
	    //最小字体大小 当将javaScriptEnabled属性设置为NO时，可以看到明显的效果
	    preference.minimumFontSize = 0;
	    //设置是否支持javaScript 默认是支持的
	    preference.javaScriptEnabled = YES;
	    // 在iOS上默认为NO，表示是否允许不经过用户交互由javaScript自动打开窗口
	    preference.javaScriptCanOpenWindowsAutomatically = YES;
	    configuration.preferences = preference;
	    
	    //设置是否允许画中画技术 在特定设备上有效
	    configuration.allowsPictureInPictureMediaPlayback = YES;
	
	    
	    WKUserContentController *userContentController = [[WKUserContentController alloc] init];
	    configuration.userContentController = userContentController;
	    
	    WKWebView *web = [[WKWebView alloc] initWithFrame:CGRectZero configuration:configuration];
	    web.userInteractionEnabled = YES;
	    web.multipleTouchEnabled = true;
	    web.backgroundColor = [UIColor whiteColor];
	    web.opaque = NO;
	//    web.scalesPageToFit = true;
	    
	    return web;
	}
	
	- (WKWebView *)baseWebView {
	    if (!_baseWebView) {
	        _baseWebView = cq_newWkWeb();
	        _baseWebView.UIDelegate = self;
	        _baseWebView.navigationDelegate = self;
	        [_baseWebView addObserver:self forKeyPath:@"title" options:NSKeyValueObservingOptionNew context:nil];
	        [_baseWebView addObserver:self forKeyPath:@"estimatedProgress" options:NSKeyValueObservingOptionNew context:nil];
	        [self.view addSubview:_baseWebView];
	    }
	    return _baseWebView;
	}
### 协议遵守
	WKNavigationDelegate, WKUIDelegate,
	
	#pragma mark - WKNavigationDelegate
	/* 页面开始加载 */
	- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation{
	}
	/* 开始返回内容 */
	- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation{
	     
	}
	/* 页面加载完成 */
	- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation{
	     
	}
	/* 页面加载失败 */
	- (void)webView:(WKWebView *)webView didFailProvisionalNavigation:(WKNavigation *)navigation{
	     
	}
	/* 在发送请求之前，决定是否跳转 */
	- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler{
	    //允许跳转
	    decisionHandler(WKNavigationActionPolicyAllow);
	    //不允许跳转
	    //decisionHandler(WKNavigationActionPolicyCancel);
	}
	/* 在收到响应后，决定是否跳转 */
	- (void)webView:(WKWebView *)webView decidePolicyForNavigationResponse:(WKNavigationResponse *)navigationResponse decisionHandler:(void (^)(WKNavigationResponsePolicy))decisionHandler{
	     
	    NSLog(@"%@",navigationResponse.response.URL.absoluteString);
	    //允许跳转
	    decisionHandler(WKNavigationResponsePolicyAllow);
	    //不允许跳转
	    //decisionHandler(WKNavigationResponsePolicyCancel);
	}
# 2.设置监听
### 监听标题
	[_baseWebView addObserver:self forKeyPath:@"title" options:NSKeyValueObservingOptionNew context:nil];
### 监听进度条
	[_baseWebView addObserver:self forKeyPath:@"estimatedProgress" options:NSKeyValueObservingOptionNew context:nil];
	
	监听处理
	#pragma mark kov
	- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
	    if([keyPath isEqualToString:@"title"]) {
	        NSString *n_new = [change dicStringForKey:@"new"];
	        [self setJs_Title:n_new];
	    } else if ([keyPath isEqualToString:@"estimatedProgress"]) {
	        self.webProgress.progress = self.baseWebView.estimatedProgress;
	        if (self.webProgress.progress == 1) {
	            __weak typeof (self)weakSelf = self;
	            [UIView animateWithDuration:0.25f delay:0.1f options:UIViewAnimationOptionCurveEaseOut animations:^{
	                weakSelf.webProgress.transform = CGAffineTransformMakeScale(1.0f, 1.2f);
	            } completion:^(BOOL finished) {
	                weakSelf.webProgress.hidden = YES;
	            }];
	        }
	    }
	}
# 3.wkweb调用原生弹框
	- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler {
	    UIAlertAction *alertActionCancel = [UIAlertAction actionWithTitle:@"Cancel" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
	        // 返回用户选择的信息
	        completionHandler();
	    }];
	    UIAlertAction *alertActionOK = [UIAlertAction actionWithTitle:@"OK" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
	        completionHandler();
	    }];
	    // alert弹出框
	    
	    UIAlertController *alterVC =[UIAlertController alertControllerWithTitle:message message:nil preferredStyle:UIAlertControllerStyleAlert];
	    [alterVC addAction:alertActionCancel];
	    [alterVC addAction:alertActionOK];
	    [self presentViewController:alterVC animated:YES completion:nil];
	}
# 4.js注入
### 对于交互的js注入
	  //注册所有js
	    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"AppJsObj" ofType:@"js"];
	    NSString *js = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
	    
	    WKUserScript *jsFile = [[WKUserScript alloc] initWithSource:js injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:true];
	    [self.configuration.userContentController addUserScript:jsFile];
### 对于需要改变值的js,例如登陆的js需要在登陆后改变js的值的注入方法
	 //注册用户信息js方法
	    NSString *jsFunc = [NSString stringWithFormat:@"var New_AppJsObj = {function() {var userinfo;}, getUserInfo: function(){return userinfo},setUserInfo: function(user){userinfo = user},getNetworkType: function(){return '%@'}}",@([CQNetworkManager status])];
	    WKUserScript *jsFileUserInfo = [[WKUserScript alloc] initWithSource:jsFunc injectionTime:WKUserScriptInjectionTimeAtDocumentStart forMainFrameOnly:true];
	    [self.configuration.userContentController addUserScript:jsFileUserInfo];
	    
	   更新注入
	    NSString *par = [[CQUser getUserInfo] getSessionToken];
    NSString *js = [NSString stringWithFormat:@"New_AppJsObj.setUserInfo('%@')",par];
    [self evaluateJavaScript:js completionHandler:^(id _Nullable data, NSError * _Nullable error) {
        if (!error) {
            NSLog(@"更新用户信息成功");
        }
    }];
    [self evaluateJavaScript:@"New_AppJsObj.getUserInfo()" completionHandler:^(id _Nullable data, NSError * _Nullable error) {
        NSLog(@"用户信息：。。%@",data);
    }];
    
    
    使用下面这个代理进行js拦截处理
    
	- (void)webView:(WKWebView *)webView decidePolicyForNavigationAction:(WKNavigationAction *)navigationAction decisionHandler:(void (^)(WKNavigationActionPolicy))decisionHandler {
	    NSString *absoluteString = navigationAction.request.URL.absoluteString;
	    NSLog(@"%@",absoluteString);
	    
	     NSString *hostString = navigationAction.request.URL.host;
	    if ([hostString containsString:@"itunes.apple.com"]) {//跳转appstore
	        if ([[UIApplication sharedApplication] canOpenURL:[NSURL URLWithString:absoluteString]]) {
	            [[UIApplication sharedApplication] openURL:[NSURL URLWithString:absoluteString]];
	             decisionHandler(WKNavigationActionPolicyCancel);
	            return;
	        }
	    }
	    NSString *scheme = navigationAction.request.URL.scheme;
	    if ([scheme isEqualToString:@"tel"]) {
	        NSString *absoluteString = navigationAction.request.URL.absoluteString;
	        NSString *telNo = [absoluteString componentsSeparatedByString:@":"].lastObject;
	        if (!CQ_stringIsEmpty(telNo)) {
	            [Utils callPhone:telNo];
	            decisionHandler(WKNavigationActionPolicyCancel);
	            return;
	        }
	    }
	    
	    if (![webView decidePolicyForNavigationAction:navigationAction decisionHandler:decisionHandler]) {
	         decisionHandler(WKNavigationActionPolicyCancel);
	        return ;
	    }
	    if (![self decidePolicyForNavigationAction:navigationAction decisionHandler:decisionHandler]) {
	        decisionHandler(WKNavigationActionPolicyCancel);
	        return ;
	    }
	    
	    //如果是跳转一个新页面
	    if (navigationAction.targetFrame == nil) {
	        [webView loadRequest:navigationAction.request];
	        decisionHandler(WKNavigationActionPolicyCancel);
	        return;
	    }
	     decisionHandler(WKNavigationActionPolicyAllow);
	}
# 5自定义userAgent
### 首先在AppDelegate文件中获取系统UA并进行记录
	// 获取 userAgent
	    self.wkWeb = [[WKWebView alloc] init];
	    __weak typeof(self) weakSelf = self;
	    [self.wkWeb evaluateJavaScript:@"navigator.userAgent" completionHandler:^(id _Nullable data, NSError * _Nullable error) {
	        NSUserDefaults *userD = [NSUserDefaults standardUserDefaults];
	        NSString *dataString = String_nil(data);
	        [userD registerDefaults:@{kUserAgentKey : dataString}];
	        [userD synchronize];
	        weakSelf.wkWeb = nil;
	    }];
### 重写UA,并将原来的拼在后面进行更新
	/**
	 重置UA 
	 */
	+ (NSString *)cq_resetUA {
	    
	    NSString *oldAgent = String_nil([kCQUserDefaults objectForKey:kUserAgentKey]);
	    if ([oldAgent containsString:prefixString]) {
	        // 截取之前的UA
	        oldAgent = [oldAgent componentsSeparatedByString:prefixString].firstObject;
	    }
	    
	    CQUser *u = [CQUser getUserInfo];
	    //loginType（0：未登录  1：第三方登录  2：手机号登录）
	    NSString *loginType = @"0";
	    if (u.isLogged) {
	        loginType = CQ_stringIsEmpty(u.phone) ? @"1" : @"2";
	    }
	    NSString *newAgent = [NSString stringWithFormat:@"%@(%@;iOS;%@;%@;%@;%@)",prefixString,[Utils getAppBuild],kCQAPPID,u.sessionId,u.token,loginType];
	    newAgent = [oldAgent stringByAppendingString:newAgent];
	    [[NSUserDefaults standardUserDefaults] cq_registerUserAgent:newAgent];
	    return newAgent;
	}
	
### 在需要更新的地方调用进行UA更新
	self.baseweb.customUserAgent = [WKWebView cq_resetUA];
	
在这里整个WkWebView就算迁移或者集成完毕了,希望大家bug少一点,工资多一点,写的不好,请大家指正
