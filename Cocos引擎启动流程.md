##Android端启动流程
#####1.Android 应用启动之后 在Cocos2dxActivity 的onCreate()生命周期中 加载jni,c++ onLoad方法中启动appDelegate。当Android的VM(Virtual Machine)执行到C组件(即*so档)里的System.loadLibrary()函数时，首先会去执行C组件里的JNI_OnLoad()函数。   

```   
cocos2d/cocos/platform/android/javaactivity-android.cpp
	 JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved)
	{
	    JniHelper::setJavaVM(vm);   
	    cocos_android_app_init(JniHelper::getEnv());    
	    return JNI_VERSION_1_4;    
	}
```
	
```   
 	app/jni/hellocpp/main.cpp

   	namespace {
		std::unique_ptr<AppDelegate> appDelegate;
	}

	void cocos_android_app_init(JNIEnv* env) {
   		 LOGD("cocos_android_app_init");
    	appDelegate.reset(new AppDelegate());
	}
```   
       
#####2.在init方法中初始化Cocos2dxGLSurfaceView，用于绘制游戏内容。设置Cocos2dxRenderer,Cocos2dxRenderer继承自 GLSurfaceView.Renderer，在onSurfaceCreated 方法中会调用nativeInit,jni调用c++中的方法，设置OpenGLView  
```
cocos2d/cocos/platform/android/javaactivity-android.cpp    

	JNIEXPORT void Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit(JNIEnv*  env, jobject thiz, jint w, jint h)
	{
   		auto director = cocos2d::Director::getInstance();
    	auto glview = director->getOpenGLView();
		if (!glview)
		{
		    glview = cocos2d::GLViewImpl::create("Android app");
		    glview->setFrameSize(w, h);
		    director->setOpenGLView(glview);

		    cocos2d::Application::getInstance()->run();
		 }
		 else
		 {
		    cocos2d::GL::invalidateStateCache();
		    cocos2d::GLProgramCache::getInstance()->reloadDefaultGLPrograms();
		    cocos2d::DrawPrimitives::init();
		    cocos2d::VolatileTextureMgr::reloadAllTextures();

		    cocos2d::EventCustom recreatedEvent(EVENT_RENDERER_RECREATED);
		    director->getEventDispatcher()->dispatchEvent(&recreatedEvent);
		    director->setGLDefaultValues();
		 }
		 cocos2d::network::_preloadJavaDownloaderClass();
	}      

``` 
#####3.Cocos2dxRenderer继承自 GLSurfaceView.Renderer，会循坏调用onDrawFrame方法，在onDrawFrame方法中调用Native方法nativeRender();c++中具体实现
```
cocos2d/cocos/platform/android/jni/Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeRender.cpp    

	JNIEXPORT void JNICALL Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeRender(JNIEnv* env) {
        cocos2d::Director::getInstance()->mainLoop();
    }
```      
#####4.通过以上步骤即实现Android应用启动后启动cocos引擎，循环调用cocos的mainLoop方法，处理游戏逻辑，绘制界面    
       
       
 ---------     
 
      
##IOS端启动流程       

#####1.ios程序启动入口 main.m中启动AppController这个类
```
	int main(int argc, char *argv[]) {
    
    	NSAutoreleasePool * pool = [[NSAutoreleasePool alloc] init];
    	int retVal = UIApplicationMain(argc, argv, nil, @"AppController");
    	[pool release];
    	return retVal;
	}
```     

#####2.AppController.mm 中执行application的run方法
```
	static AppDelegate s_sharedApplication;

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
		    cocos2d::Application *app = cocos2d::Application::getInstance();
   			 app->initGLContextAttrs();

   			 *
   			 *
   			 *
   			  // IMPORTANT: Setting the GLView should be done after creating the RootViewController
    		cocos2d::GLView *glview = cocos2d::GLViewImpl::createWithEAGLView(eaglView);
    		cocos2d::Director::getInstance()->setOpenGLView(glview);

    		app->run();
	}
```
#####3.CCApplication-ios.mm的run方法具体实现
```
	int Application::run()
	{
    	if (applicationDidFinishLaunching())
    	{
        	[[CCDirectorCaller sharedDirectorCaller] startMainLoop];
    	}
    	return 0;
	}
```

#####4.CCDirectorCaller-ios.mm 的方法startMainLoop具体实现
```    
	-(void) startMainLoop
	{
        // Director::setAnimationInterval() is called, we should invalidate it first
    	[self stopMainLoop];
    
    	displayLink = [NSClassFromString(@"CADisplayLink") displayLinkWithTarget:self selector:@selector(doCaller:)];
    	[displayLink setFrameInterval: self.interval];
    	[displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSDefaultRunLoopMode];
	}
```      
CADisplayLink是一个能让我们以和屏幕刷新率相同的频率将内容画到屏幕上的定时器。  doCaller是定时器的回调函数

```     
	-(void) doCaller: (id) sender
	{
    	if (isAppActive)
    	{
        	cocos2d::Director* director = cocos2d::Director::getInstance();
        	[EAGLContext setCurrentContext: [(CCEAGLView*)director->getOpenGLView()->getEAGLView() context]];
        		director->mainLoop();
    	}
	}
``` 

#####5.可以看到最终执行到的还是 cocos2d::Director 的mainLoop方法。即每帧都会调用mainLoop方法，循环调用。
---

##mainLoop方法分析
`/cocos2d/cocos/base/CCDirector.cpp`     

```
	void DisplayLinkDirector::mainLoop()
	{
	    if (_purgeDirectorInNextLoop)//是否清除导演退出,director->end()将_purgeDirectorInNextLoop置为true,下一帧mainLoop方法时销毁director
	    {
	        _purgeDirectorInNextLoop = false;
	        purgeDirector();//清除缓存退出游戏
	    }
	    else if (_restartDirectorInNextLoop)//是否重新启动director, director->restar() 将_restartDirectorInNextLoop置为true,下一帧mainLoop方法时调用restartDirector()方法重启director
	    {
	        _restartDirectorInNextLoop = false;
	        restartDirector();
	    }
	    else if (! _invalid)//是否无效场景，初始值为false
	    {
	        drawScene(); //图形渲染以及消息事件处理
	        // release the objects
	        PoolManager::getInstance()->getCurrentPool()->clear();//渲染一帧后，进行对当前内存池进行清理，每一帧都会创建一个内存池
	    }
	}
```    

#####1.判断是否清除导演退出。调用director->end() 方法会将_purgeDirectorInNextLoop置为true,在下一帧mainLoop方法时执行purgeDirector()方法，清除缓存，清除director退出游戏
#####2.判断是否重新启动director。调用director->restar() 方法会将_restartDirectorInNextLoop值置为true,在下一帧mainLoop方法执行restartDirector()，重启director
#####3.判断当前场景是否是无效场景， _invalid初始值为false,执行drawScene()方法，渲染图形以及处理消息事件

### drawScene分析   
```
	// Draw the Scene
	void Director::drawScene()
	{
	    // calculate "global" dt
	    calculateDeltaTime();//计算与上一帧的间隔,保存在全局变量_deltaTime中
	    
	    if (_openGLView)
	    {
	        _openGLView->pollEvents();
	    }

	    //tick before glClear: issue #533
	    if (! _paused)
	    {
	        _eventDispatcher->dispatchEvent(_eventBeforeUpdate);
	        _scheduler->update(_deltaTime);
	        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
	    }

	    _renderer->clear();
	    experimental::FrameBuffer::clearAllFBOs();
	    /* to avoid flickr, nextScene MUST be here: after tick and before draw.
	     * FIXME: Which bug is this one. It seems that it can't be reproduced with v0.9
	     */
	    if (_nextScene)
	    {
	        setNextScene();
	    }

	    pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
	    
	    if (_runningScene)
	    {
	#if (CC_USE_PHYSICS || (CC_USE_3D_PHYSICS && CC_ENABLE_BULLET_INTEGRATION) || CC_USE_NAVMESH)
	        _runningScene->stepPhysicsAndNavigation(_deltaTime);
	#endif
	        //clear draw stats
	        _renderer->clearDrawStats();
	        
	        //render the scene
	        _openGLView->renderScene(_runningScene, _renderer);
	        
	        _eventDispatcher->dispatchEvent(_eventAfterVisit);
	    }

	    // draw the notifications node
	    if (_notificationNode)
	    {
	        _notificationNode->visit(_renderer, Mat4::IDENTITY, 0);
	    }

	    if (_displayStats)
	    {
	        showStats();
	    }
	    _renderer->render();

	    _eventDispatcher->dispatchEvent(_eventAfterDraw);

	    popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

	    _totalFrames++;

	    // swap buffers
	    if (_openGLView)
	    {
	        _openGLView->swapBuffers();
	    }

	    if (_displayStats)
	    {
	        calculateMPF();
	    }
	}
```     
#####1.calculateDeltaTime();计算时间差，两帧之间的时间差，保存在全局变量_deltaTime中   
#####2.定时任务调用，执行update(_deltaTime)方法.    
Scheduler用于处理更新函数调用，场景中经常使用的schedulerUpdate()，然后实现一个update(float dt)方法，实际上就是将调用请求注册到了Scheduler中，之后再每一帧的mainLoop方法进行统一调用update()函数。    

#####3.事件处理分发，将事件分发出去   
  
```   
	_eventDispatcher->dispatchEvent(_eventBeforeUpdate);//马上要调用更新了
	...    
    _eventDispatcher->dispatchEvent(_eventAfterUpdate);//更新调用完毕   
```    
#####4.设置下一个场景    
判断下一个场景_nextScene是否不为空，如果不为空则调用setNextScene()方法，主要是把当前场景\_runningScene释放掉，\_runningScene生命周期进入onExit()，然后把 _nextScene设置为当前运行场景\_runningScene，将\_nextScene置空。当前运行场景生命周期进入onEnter().   

```   
	void Director::setNextScene()
	{
	    bool runningIsTransition = dynamic_cast<TransitionScene*>(_runningScene) != nullptr;
	    bool newIsTransition = dynamic_cast<TransitionScene*>(_nextScene) != nullptr;
	
	    // If it is not a transition, call onExit/cleanup
	     if (! newIsTransition)
	     {
	         if (_runningScene)//当前运行场景不为空
	         {
	             _runningScene->onExitTransitionDidStart();
	             _runningScene->onExit();//生命周期进入onExit();
	         }
	 
	         // issue #709. the root node (scene) should receive the cleanup message too
	         // otherwise it might be leaked.
	         if (_sendCleanupToScene && _runningScene)
	         {
	             _runningScene->cleanup();
	         }
	     }
	
	    if (_runningScene)
	    {
	        _runningScene->release();//释放当前场景
	    }
	    _runningScene = _nextScene;//将_nextScene置为当前场景
	    _nextScene->retain();
	    _nextScene = nullptr;
	
	    if ((! runningIsTransition) && _runningScene)
	    {
	        _runningScene->onEnter();//生命周期进入onEnter();
	        _runningScene->onEnterTransitionDidFinish();
	    }
	}
```   
#####5.调用当前场景的渲染方法
  



