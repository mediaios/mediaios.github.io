---
layout: post
title: OpenGL入门
description: 讲解OpenGL在ios上的入门级开发
category: blog
tag: ios, Objective-C, OpenGL
---

## 概述

OpenGL是一个低级编程的API，它可以实现2D和3D图形编程。cocos2d，sparrow，corona，unity 这些框架，你会发现其实它们都是基于OpenGL上创建的。

本篇教程是入门级教程，主要讲述在使用OpenGL时的一些重要概念，以及对一些示例程序进行讲解。



## 重要概念

### 着色器

在OpenGL ES2.0 的世界，在场景中渲染任何一种几何图形，你都需要创建两个称之为“着色器”的小程序。着色器由一个类似C的语言编写- GLSL 。

有两种类型的着色器，它们分别是：顶点着色器和片段着色器。

* Vertex shaders(顶点着色器)：假如你在渲染一个简单的场景：一个长方形，每个角只有一个顶点。于是vertex shader 会被调用四次。它负责执行：诸如灯光、几何变换等等的计算。得出最终的顶点位置后，为下面的片段着色器提供必须的数据。
* Fragment shaders(片段着色器)：在一个简单的场景，也是刚刚说到的长方形。这个长方形所覆盖到的每一个像素，都会调用一次fragment shader。片段着色器的责任是计算灯光，以及更重要的是计算出每个像素的最终颜色。

## 示例程序

### HelloOpenGL

#### 1）创建工程并添加OpenGLView类，继承自UIView

如下是 OpenGLView.h

	#import <UIKit/UIKit.h>
	#import <OpenGLES/ES2/gl.h>
	#import <OpenGLES/ES2/glext.h>
	#import <QuartzCore/QuartzCore.h>
	
	@interface OpenGLView : UIView
	{
	    CAEAGLLayer* _eaglLayer;
	    EAGLContext* _context;
	    GLuint _colorRenderBuffer;
	}
	
	@end

#### 2）设置 layer class 为 CAEAGLLayer

	+ (Class)layerClass {
	    return [CAEAGLLayer class];
	}

想要显示OpenGL的内容，你需要把它缺省的layer设置为一个特殊的layer。（CAEAGLLayer）。这里通过直接复写layerClass的方法。

#### 3） 设置layer为不透明

	- (void)setupLayer {
	    _eaglLayer = (CAEAGLLayer*) self.layer;
	    _eaglLayer.opaque = YES;
	}

因为缺省的话，CALayer是透明的。而透明的层对性能负荷很大，特别是OpenGL的层。

#### 4）创建OpenGL context

	- (void)setupContext {   
	    EAGLRenderingAPI api = kEAGLRenderingAPIOpenGLES2;
	    _context = [[EAGLContext alloc] initWithAPI:api];
	    if (!_context) {
	        NSLog(@"Failed to initialize OpenGLES 2.0 context");
	        exit(1);
	    }
	 
	    if (![EAGLContext setCurrentContext:_context]) {
	        NSLog(@"Failed to set current OpenGL context");
	        exit(1);
	    }
	}

无论你要OpenGL帮你实现什么，总需要这个 `EAGLContext` ,`EAGLContext`管理所有通过OpenGL进行draw的信息。这个与Core Graphics context类似。

#### 5) 创建render buffer(渲染缓冲区)

	- (void)setupRenderBuffer {
	    glGenRenderbuffers(1, &_colorRenderBuffer);
	    glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderBuffer);        
	    [_context renderbufferStorage:GL_RENDERBUFFER fromDrawable:_eaglLayer];    
	}
	
Render buffer 是OpenGL的一个对象，用于存放渲染过的图像。有时候你会发现render buffer会作为一个color buffer被引用，因为本质上它就是存放用于显示的颜色。

创建render buffer的三步：

* 调用glGenRenderbuffers来创建一个新的render buffer object。这里返回一个唯一的integer来标记render buffer（这里把这个唯一值赋值到_colorRenderBuffer）。有时候你会发现这个唯一值被用来作为程序内的一个OpenGL 的名称。
* 调用glBindRenderbuffer ，告诉这个OpenGL：我在后面引用GL_RENDERBUFFER的地方，其实是想用_colorRenderBuffer。其实就是告诉OpenGL，我们定义的buffer对象是属于哪一种OpenGL对象。
* 最后，为render buffer分配空间。renderbufferStorage

#### 6）创建一个 frame buffer（帧缓冲区）

	- (void)setupFrameBuffer {    
	    GLuint framebuffer;
	    glGenFramebuffers(1, &framebuffer);
	    glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
	    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
	        GL_RENDERBUFFER, _colorRenderBuffer);
	 }
	
Frame buffer也是OpenGL的对象，它包含了前面提到的render buffer，以及其它后面会讲到的诸如：depth buffer、stencil buffer 和 accumulation buffer。

前两步创建frame buffer的动作跟创建render buffer的动作很类似。（反正也是用一个glBind什么的）

而最后一步  glFramebufferRenderbuffer 这个才有点新意。它让你把前面创建的buffer render依附在frame buffer的GL_COLOR_ATTACHMENT0位置上。

#### 7）清理屏幕

	- (void)render {
	    glClearColor(0, 104.0/255.0, 55.0/255.0, 1.0);
	    glClear(GL_COLOR_BUFFER_BIT);
	    [_context presentRenderbuffer:GL_RENDERBUFFER];
	}
	
　为了尽快在屏幕上显示一些什么，在我们和那些 vertexes、shaders打交道之前，把屏幕清理一下，显示另一个颜色吧。（RGB 0, 104, 55，绿色吧）

这里每个RGB色的范围是0~1，所以每个要除一下255.

下面解析一下每一步动作：

* 调用glClearColor ，设置一个RGB颜色和透明度，接下来会用这个颜色涂满全屏。
* 调用glClear来进行这个“填色”的动作（大概就是photoshop那个油桶嘛）。还记得前面说过有很多buffer的话，这里我们要用到GL_COLOR_BUFFER_BIT来声明要清理哪一个缓冲区。
* 调用OpenGL context的presentRenderbuffer方法，把缓冲区（render buffer和color buffer）的颜色呈现到UIView上。

#### 8）串联以上动作，修改OpenGLView.m

	// Replace initWithFrame with this
	- (id)initWithFrame:(CGRect)frame
	{
	    self = [super initWithFrame:frame];
	    if (self) {        
	        [self setupLayer];        
	        [self setupContext];                
	        [self setupRenderBuffer];        
	        [self setupFrameBuffer];                
	        [self render];        
	    }
	    return self;
	}

#### 9) 在 ViewController 中引入 OpenGLView

	#import "ViewController.h"
	#import "OpenGLView.h"
	@interface ViewController ()
	@property (nonatomic,strong) OpenGLView *glView;
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    // Do any additional setup after loading the view, typically from a nib.
	    _glView = [[OpenGLView alloc] initWithFrame:self.view.bounds];
	    [self.view addSubview:_glView];
	}
	
	- (void)didReceiveMemoryWarning {
	    [super didReceiveMemoryWarning];
	    // Dispose of any resources that can be recreated.
	}
	@end

#### 10) 运行

如果以上步骤一切顺利的话，你将会得到纯颜色（绿色）界面。

### 添加着色器

#### 1）添加顶点着色器

创建一个空文件，命名为 SimpleVertex.gls 

	attribute vec4 Position; // 1
	attribute vec4 SourceColor; // 2
	 
	varying vec4 DestinationColor; // 3
	 
	void main(void) { // 4
	    DestinationColor = SourceColor; // 5
	    gl_Position = Position; // 6
	}

1 “attribute”声明了这个shader会接受一个传入变量，这个变量名为“Position”。在后面的代码中，你会用它来传入顶点的位置数据。这个变量的类型是“vec4”,表示这是一个由4部分组成的矢量。

2 与上面同理，这里是传入顶点的颜色变量。

3 这个变量没有“attribute”的关键字。表明它是一个传出变量，它就是会传入片段着色器的参数。“varying”关键字表示，依据顶点的颜色，平滑计算出顶点之间每个像素的颜色。

4 每个shader都从main开始– 跟C一样嘛。

5 设置目标颜色 = 传入变量：SourceColor

6 gl_Position 是一个内建的传出变量。这是一个在 vertex shader中必须设置的变量。这里我们直接把gl_Position = Position; 没有做任何逻辑运算。

#### 2)添加片段着色器

	varying lowp vec4 DestinationColor; // 1
	 
	void main(void) { // 2
	    gl_FragColor = DestinationColor; // 3
	}

1 这是从vertex shader中传入的变量，这里和vertex shader定义的一致。而额外加了一个关键字：lowp。在fragment shader中，必须给出一个计算的精度。出于性能考虑，总使用最低精度是一个好习惯。这里就是设置成最低的精度。如果你需要，也可以设置成medp或者highp.

2 也是从main开始嘛

3 正如你在vertex shader中必须设置gl_Position, 在fragment shader中必须设置gl_FragColor.

#### 3）编译顶点着色器和片段着色器

目前为止，xcode仅仅会把这两个文件copy到application bundle中。我们还需要在运行时编译和运行这些shader。你可能会感到诧异。为什么要在app运行时编译代码？这样做的好处是，我们的着色器不用依赖于某种图形芯片。（这样才可以跨平台嘛）

	- (GLuint)compileShader:(NSString*)shaderName withType:(GLenum)shaderType {
	 
	    // 1
	    NSString* shaderPath = [[NSBundle mainBundle] pathForResource:shaderName 
	        ofType:@"glsl"];
	    NSError* error;
	    NSString* shaderString = [NSString stringWithContentsOfFile:shaderPath 
	        encoding:NSUTF8StringEncoding error:&error];
	    if (!shaderString) {
	        NSLog(@"Error loading shader: %@", error.localizedDescription);
	        exit(1);
	    }
	 
	    // 2
	    GLuint shaderHandle = glCreateShader(shaderType);    
	 
	    // 3
	constchar* shaderStringUTF8 = [shaderString UTF8String];    
	    int shaderStringLength = [shaderString length];
	    glShaderSource(shaderHandle, 1, &shaderStringUTF8, &shaderStringLength);
	 
	    // 4
	    glCompileShader(shaderHandle);
	 
	    // 5
	    GLint compileSuccess;
	    glGetShaderiv(shaderHandle, GL_COMPILE_STATUS, &compileSuccess);
	    if (compileSuccess == GL_FALSE) {
	        GLchar messages[256];
	        glGetShaderInfoLog(shaderHandle, sizeof(messages), 0, &messages[0]);
	        NSString *messageString = [NSString stringWithUTF8String:messages];
	        NSLog(@"%@", messageString);
	        exit(1);
	    }
	 
	    return shaderHandle;
	 
	}

解析：

1 这是一个UIKit编程的标准用法，就是在NSBundle中查找某个文件。大家应该熟悉了吧。

2 调用 glCreateShader来创建一个代表shader 的OpenGL对象。这时你必须告诉OpenGL，你想创建 fragment shader还是vertex shader。所以便有了这个参数：shaderType

3 调用glShaderSource ，让OpenGL获取到这个shader的源代码。（就是我们写的那个）这里我们还把NSString转换成C-string

4 最后，调用glCompileShader 在运行时编译shader

5 大家都是程序员，有程序的地方就会有fail。有程序员的地方必然会有debug。如果编译失败了，我们必须一些信息来找出问题原因。 glGetShaderiv 和 glGetShaderInfoLog  会把error信息输出到屏幕。（然后退出）


关联两个着色器

	- (void)compileShaders {
	 
	    // 1
	    GLuint vertexShader = [self compileShader:@"SimpleVertex" 
	        withType:GL_VERTEX_SHADER];
	    GLuint fragmentShader = [self compileShader:@"SimpleFragment" 
	        withType:GL_FRAGMENT_SHADER];
	 
	    // 2
	    GLuint programHandle = glCreateProgram();
	    glAttachShader(programHandle, vertexShader);
	    glAttachShader(programHandle, fragmentShader);
	    glLinkProgram(programHandle);
	 
	    // 3
	    GLint linkSuccess;
	    glGetProgramiv(programHandle, GL_LINK_STATUS, &linkSuccess);
	    if (linkSuccess == GL_FALSE) {
	        GLchar messages[256];
	        glGetProgramInfoLog(programHandle, sizeof(messages), 0, &messages[0]);
	        NSString *messageString = [NSString stringWithUTF8String:messages];
	        NSLog(@"%@", messageString);
	        exit(1);
	    }
	 
	    // 4
	    glUseProgram(programHandle);
	 
	    // 5
	    _positionSlot = glGetAttribLocation(programHandle, "Position");
	    _colorSlot = glGetAttribLocation(programHandle, "SourceColor");
	    glEnableVertexAttribArray(_positionSlot);
	    glEnableVertexAttribArray(_colorSlot);
	}

下面是解析：

　　1 用来调用你刚刚写的动态编译方法，分别编译了vertex shader 和 fragment shader

　　2 调用了glCreateProgram glAttachShader  glLinkProgram 连接 vertex 和 fragment shader成一个完整的program。

　　3 调用 glGetProgramiv  lglGetProgramInfoLog 来检查是否有error，并输出信息。

　　4 调用 glUseProgram  让OpenGL真正执行你的program

　　5 最后，调用 glGetAttribLocation 来获取指向 vertex shader传入变量的指针。以后就可以通过这写指针来使用了。还有调用 glEnableVertexAttribArray来启用这些数据。（因为默认是 disabled的。）
　　
最后还有两步：

　1 在 initWithFrame方法里，在调用render之前要加入这个：

	[self compileShaders];

　2 在@interface in OpenGLView.h 中添加两个变量：

	GLuint _positionSlot;
	GLuint _colorSlot;

如果你仍能正常地看到之前那个绿色的屏幕，就证明你前面写的代码都很好地工作了。

