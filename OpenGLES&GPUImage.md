## OpenGLES入门

### OpenGL 渲染管线Pipeline
是指三维渲染的过程中显卡执行的、从几何体到最终渲染图像的、数据传输处理计算的过程。

1. 顶点着色器:渲染管线的第一个部分是顶点着色器（vertex shader），它把一个单独的顶点作为输入。顶点着色器主要的目的是把3D坐标转为另一种3D坐标（投影坐标），同时顶点着色器允许我们对顶点属性进行一些基本处理。
2. 图元组装:输出会传递给几何着色器（geometry shader）。几何着色器把基本图形(点，线，三角形)形成的一系列顶点的集合作为输入，它可以通过产生新顶点构造出新的（或是其他的）基本图形来生成其他形状。
3. 光栅化（rasterization）：这里它会把基本图形映射为屏幕上相应的像素，生成供片段着色器（fragment shader）使用的fragment（OpenGL中的一个fragment是OpenGL渲染一个独立像素所需的所有数据。）。在片段着色器运行之前，会执行裁切（clipping）。裁切会丢弃超出你的视图以外的那些像素，来提升执行效率。
4. 片段着色器：片段着色器的主要目的是计算一个像素的最终颜色，这也是OpenGL高级效果产生的地方。通常，片段着色器包含用来计算像素最终颜色的3D场景的一些数据（比如光照、阴影、光的颜色等等）。
5. 在所有相应颜色值确定以后，最终它会传到另一个阶段，我们叫做alpha测试和混合（blending）阶段。这个阶段检测像素的相应的深度（和stencil）值，使用这些来检查这个像素是否在另一个物体的前面或后面，如此做到相应取舍。这个阶段也会查看alpha值（alpha值是一个物体的透明度值）和物体之间的混合（blend）。所以即使在像素着色器中计算出来了一个像素所输出的颜色，最后的像素颜色在渲染多个三角形的时候也可能完全不同。


### OpenGL标准化设备坐标
范围[-1,1],(0, 0)是在屏幕的正中间。任何落在范围外的坐标都会被丢弃/裁剪，不会显示在你的屏幕上。你的标准化设备坐标接着会变换为屏幕空间坐标(Screen-space Coordinates)，这是使用你通过glViewport函数提供的数据，进行视口变换(Viewport Transform)完成的。


### 着色器
着色器(Shader)是运行在GPU上的小程序。这些小程序为图形渲染管线的某个特定部分而运行。从基本意义上来说，着色器只是一种把输入转化为输出的程序。着色器也是一种非常独立的程序，因为它们之间不能相互通信；它们之间唯一的沟通只有通过输入和输出。着色器程序的使用：

```
        //1.从文件路径中获取顶点着色器和片段着色器
        NSString* vertexContent = [NSString stringWithContentsOfFile:vertexPath encoding:NSUTF8StringEncoding error:nil];
        NSString* fragmentContent = [NSString stringWithContentsOfFile:fragmentPath encoding:NSUTF8StringEncoding error:nil];
        
        const GLchar* vertextSource = [vertexContent UTF8String];
        const GLchar* fragmentSource = [fragmentContent UTF8String];
        
        //2.编译和链接着色器程序
        GLuint vertextShader , fragmentShader;
        GLchar message[512];
        GLint sucess;
        
        //顶点着色器
        vertextShader = glCreateShader(GL_VERTEX_SHADER);
        glShaderSource(vertextShader, 1, &vertextSource, NULL);
        glCompileShader(vertextShader);
        glGetShaderiv(vertextShader, GL_COMPILE_STATUS, &sucess);
        if (!sucess) {
            glGetShaderInfoLog(vertextShader, 512, NULL, message);
            NSLog(@"error : vertext shader compile failed : %@",[NSString stringWithUTF8String:message]);
            
            return nil;
            
        }
        
        
        //片段着色器
        fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
        glShaderSource(fragmentShader, 1, &fragmentSource, NULL);
        glCompileShader(fragmentShader);
        glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &sucess);
        if (!sucess) {
            glGetShaderInfoLog(fragmentShader, 512, NULL, message);
            NSLog(@"error : fragment shader compile failed : %@",[NSString stringWithUTF8String:message]);
            
            return nil;
            
        }
        
        //3.创建着色器程序
        self.program = glCreateProgram();
        //连接着色器
        glAttachShader(self.program, vertextShader);
        glAttachShader(self.program, fragmentShader);
        glLinkProgram(self.program);
        glGetProgramiv(self.program, GL_LINK_STATUS, &sucess);
        if (!sucess) {
            glGetProgramInfoLog(self.program, 512, NULL, message);
            NSLog(@"error : program link failed : %@",[NSString stringWithUTF8String:message]);
            
            return nil;
        }
        
        //4.删除着色器
        glDeleteShader(vertextShader);
        glDeleteShader(fragmentShader);
        
        //5.使用着色器程序
		glUseProgram(self.program);

```

### 顶点缓冲对象VBO
定义顶点数据以后，我们会把它作为输入发送给图形渲染管线的第一个处理阶段：顶点着色器。它会在GPU上创建内存用于储存我们的顶点数据，还要配置OpenGL如何解释这些内存，并且指定其如何发送给显卡。顶点着色器接着会处理我们在内存中指定数量的顶点。

我们通过顶点缓冲对象(Vertex Buffer Objects, VBO)管理这个内存，它会在GPU内存(通常被称为显存)中储存大量顶点。使用这些缓冲对象的好处是我们可以一次性的发送一大批数据到显卡上，而不是每个顶点发送一次。从CPU把数据发送到显卡相对较慢，所以只要可能我们都要尝试尽量一次性发送尽可能多的数据。当数据发送至显卡的内存中后，顶点着色器几乎能立即访问顶点，这是个非常快的过程。

```
   //顶点位置的数组
	GLfloat vertices[] = {
        0.5f, 0.5f, 0.0f,   // 右上角
        0.5f, -0.5f, 0.0f,  // 右下角
        -0.5f, -0.5f, 0.0f, // 左下角
        -0.5f, 0.5f, 0.0f   // 左上角
    };
	
    //顶点缓冲对象VBO
	GLint vertexPosition = glGetAttribLocation(self.shader.program, "inputVertexPosition");
    GLuint VBO;
    glGenBuffers(1, &VBO);//生成VBO标识符
    glBindBuffer(GL_ARRAY_BUFFER, VBO);//设置缓冲对象的保存方式，这里表示放在数组当中
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);//将VBO从CPU复制到GPU
    //告诉OpenGL如何解析顶点属性
    glVertexAttribPointer(vertexPosition,
                          3,//指定顶点属性的大小
                          GL_FLOAT,//顶点的数据类型
                          GL_FALSE,//是否标准化，若GL_TRUE所有数据会被映射到-1到1之间
                          3 * sizeof(GLfloat),//步长，表示连续顶点属性之间的间隔。
                          (GLvoid *)0);//它表示位置数据在缓冲中起始位置的偏移量(Offset)。由于位置数据在数组的开头，所以这里是0。
    glEnableVertexAttribArray(vertexPosition);    
```

### 顶点数组对象 VAO
顶点数组对象(Vertex Array Object, VAO)可以像顶点缓冲对象那样被绑定，任何随后的顶点属性调用都会储存在这个VAO中。这样的好处就是，当配置顶点属性指针时，你只需要将那些调用执行一次，之后再绘制物体的时候只需要绑定相应的VAO就行了。这使在不同顶点数据和属性配置之间切换变得非常简单，只需要绑定不同的VAO就行了，刚刚设置的所有状态都将存储在VAO中。

```
    //VAO
    GLUint VAO;
    glGenVertexArraysOES(1, &VAO);
    glBindVertexArrayOES(VAO);
    
    ...
    
    //解绑VAO
    glBindVertexArrayOES(0);
```

### 索引缓冲对象 EBO
对于重叠的顶点只储存不同的顶点，并设定绘制这些顶点的顺序。，之后只要指定绘制的顺序就行了。

```
	//顶点顺序的数组
    GLuint indices [] = {
        0,1,3,
        3,1,2
    };
    
    //EBO
	GLuint EBO;
    glGenBuffers(1, &EBO);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
    
    //绘制
    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
        
```

### OpenGL中缓冲对象使用的一般套路

1. 生成，请求OpenGL为GPU控制的缓存生成一个独一无二的标识符：glGenBuffers（）
2. 绑定，告诉OpenGL接下来的运算使用一个缓存:glBindBuffer()
3. 缓存数据,让OpenGL为当前绑定的缓存分配并初始化足够的连续内存，通常是从CPU复制到GPU:glBufferData()
4. 设置指针，告诉OpenGL在缓存中的数据的类型和所需要访问的数据的内存偏移量: glVertexAttribPointer()
5. 启用，告诉OpenGL在接下来的渲染中是否使用缓存中的数据：glEnableVertexAttribArray
6. 绘图，告诉OpenGL使用当前绑定并启用的缓存中的数据渲染：glDrawArrays()或glDrawElements()
7. 删除，告诉OpenGL释放资源：glDeleteBuffers()


### 帧缓冲 FBO

帧缓冲是由像素组成的二维数组，每一个存储单元对应屏幕上的一个像素，整个帧缓冲对应一帧图像即当前屏幕画面。帧缓冲通常包括：颜色缓冲，深度缓冲，模板缓冲和累积缓冲。可以同时存在很多的帧缓冲，并且可以通过OpenGLES让GPU把渲染结果存储到任意数量的帧缓冲中。但是屏幕显示限速要受到保存在*前帧缓冲*的特定帧缓冲中的像素颜色元素的控制。程序不会直接把渲染结果保存到前帧缓冲，因为这样会让用户看到还没渲染完成的图像，而是保存在包括*后帧缓冲*在内的其他缓冲中，当后帧缓存包含一个完成的图像时，前帧缓冲会和后帧缓冲瞬间切换。

在OPenGL中生成并绑定帧缓冲的方式如下：

```
	glGenFramebuffers(1, &_defaultFrameBuffer);
	glBindFramebuffer(GL_FRAMEBUFFER, _defaultFrameBuffer);
        
	//颜色缓冲区
	glGenRenderbuffers(1, &_colorRenderBuffer);
	glBindRenderbuffer(GL_RENDERBUFFER, _colorRenderBuffer);
        
	//将_colorRenderBuffer装配到GL_COLOR_ATTACHMENT0 这个装配点上
	glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_RENDERBUFFER, _colorRenderBuffer);
        
```

### 纹理
2D纹理图像,纹理坐标在x和y轴上，范围为0到1之间
纹理坐标起始于(0, 0)，也就是纹理图片的左下角，终始于(1, 1)，即纹理图片的右上角。


```
	NSData* imageData = ....;
	 GLUint texture;
    glEnable(GL_TEXTURE_2D);
    
    glGenTextures(1, & texture);
    glBindTexture(GL_TEXTURE_2D, texture);
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR );
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR );
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri( GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, (GLsizei)width,(GLsizei)height, 0, GL_RGBA, GL_UNSIGNED_BYTE, imageData.bytes);
    
```

### GLSL
着色器语言，跟C语言语法很类似。

### GLSL常用数据类型
- float :浮点型
- vecn : 包含n个float分量的默认向量。一个向量的分量可以通过vec.x这种方式获取，这里x是指这个向量的第一个分量。你可以分别使用.x、.y、.z和.w来获取它们的第1、2、3、4个分量。GLSL也允许你对颜色使用rgba，或是对纹理坐标使用stpq访问相同的分量。
- matn:矩阵
- sampler2D : 纹理采样器，设置为uniform类型，从application中传入纹理标识符。

### GLSL的三种变量类型

- attribute变量：attribute变量是只能在vertex shader中使用的变量。一般用于Application向GLSL传递信息。

- varying变量:varying变量是vertex和fragment shader之间做数据传递用的。一般vertex shader修改varying变量的值，然后fragment shader使用该varying变量的值。因此varying变量在vertex和fragment shader二者之间的声明必须是一致的。application不能使用此变量。

- Uniform变量: Uniform是一种从CPU中的应用向GPU中的着色器发送数据的方式，它是全局的。我们用glGetUniformLocation查询uniform ourColor的位置值.我们可以通过glUniform4f函数设置uniform值。注意，查询uniform地址不要求你之前使用过着色器程序，但是更新一个uniform之前你必须先使用程序（调用glUseProgram)，因为它是在当前激活的着色器程序中设置uniform的。


### GLSL的数据精度

- lowp 低
- mediump 中
- highp 高
- 一般来说，顶点位置为highp、纹理坐标为mediump、颜色使用lowp。

### GLSL常用的内置函数
- gl_Position,顶点着色器中使用，表示顶点位置
- gl_FragColor，片段着色器中使用，表示该像素点的颜色
- texture2D，获取2d纹理中某点的颜色，第一个参数传纹理采样器，第二个参数传纹理坐标
- mix，纹理混合。

### 使用GLSL的一些tips

- GPU程序无法保存静态变量，每一帧程序中的变量都会重新计算。
- 都是浮点计算，应尽量减少比较操作
- GLSL中的类型无法自动转换。


一个使用GLSL绘制移动的棋盘格例子:

```
//顶点着色器

//输入
attribute vec3 inputPosition;//从application中传入的顶点位置信息
attribute vec2 textureCoord;//从application中传入的纹理坐标信息
uniform highp float globalTime;//从application中传入的时间信息
uniform highp float columnCount;//从application中传入的棋盘列数

//输出到片段着色器
varying lowp vec4 passPosition; 
varying highp vec2 passCoord;
varying highp float passTime;
varying highp float passCount;


void main() {
    gl_Position = vec4(inputPosition,1.0);
    passPosition = vec4(inputPosition,1.0);
    passCoord = textureCoord;
    passTime = globalTime;
    passCount = columnCount;
}


//片段着色器
varying lowp vec4 passPosition;
varying highp vec2 passCoord;
varying highp float passTime;
varying highp float passCount;


void main() {
    
    highp float x = passCoord.x;
    highp float y = passCoord.y;
    highp float gap = (1.0/passCount);
    
    if (mod(ceil(((x + passTime)/ gap)),2.0) == 0.0) {
        if (mod(ceil((y / gap)),2.0) == 0.0) {
            gl_FragColor = vec4(0.0,0.0,0.0,1.0);
        } else {
            gl_FragColor = vec4(1.0,1.0,1.0,1.0);
        }
    } else {
        if (mod(ceil((y / gap)),2.0) == 0.0) {
            gl_FragColor = vec4(1.0,1.0,1.0,1.0);
        } else {
            gl_FragColor = vec4(0.0,0.0,0.0,1.0);
        }
    }
}

```

### OpenGL坐标系统
- 局部空间：你的模型的所有顶点都是在局部空间中,它们相对于你的物体来说都是局部的。
- 世界空间：是指顶点相对于世界的坐标。
- 观察空间：观察空间是将世界空间坐标转化为用户视野前方的坐标而产生的结果。因此观察空间就是从摄像机的视角所观察到的空间。而这通常是由一系列的位移和旋转的组合来完成，平移/旋转场景从而使得特定的对象被变换到摄像机的前方。
- 裁剪空间：在一个顶点着色器运行的最后，OpenGL期望所有的坐标都能落在一个特定的范围内，且任何在这个范围之外的点都应该被裁剪掉(Clipped)。被裁剪掉的坐标就会被忽略，所以剩下的坐标就将变为屏幕上可见的片段。这也就是裁剪空间(Clip Space)名字的由来。


## 在iOS中使用OpenGLES

### GLKit概述
- 在 iOS中使用EAGL提供的EAGLContext类 来实现和提供一个呈现环境，用来保持OpenGLES使用到的硬件状态。  EAGL是一个Objective-C API，提供使OpenGL ES与Core Animation和UIKit集成的接口。
- GLKView 和GLKViewController类提供一个标准的OpenGLES视图和相关联的呈现循环。GLKView可以作为OpenGLES内容的呈现目标，GLKViewController提供内容呈现的控制和动画。视图管理和维护一个framebuffer，应用只需在framebuffer进行绘画即可。
- GLKTextureLoader 为应用提供从iOS支持的各种图像格式的源自动加载纹理图像到OpenGLES 图像环境的方式，并能够进行适当的转换，并支持同步和异步加载方式。
- 数学运算库，提供向量、矩阵、四元数的实现和矩阵堆栈操作等OpenGL ES 1.1功能。
- Effect效果类提供标准的公共着色效果的实现。能够配置效果和相关的顶点数据，然后创建和加载适当的着色器。GLKit 包括三个可配置着色效果类：GLKBaseEffect实现OpenGL ES 1.1规范中的关键的灯光和材料模式, GLKSkyboxEffect提供一个skybox效果的实现, GLKReflectionMapEffect 在GLKBaseEffect基础上包括反射映射支持。

### GLKit中的变换操作
#### 基本变换
- 平移 : GLKMatrix4MakeTranslation(float tx, float ty, float tz)
- 旋转 : GLKMatrix4MakeRotation(float radians, float x, float y, float z)
- 缩放 : GLKMatrix4MakeScale(float sx, float sy, float sz)
- 透视 : GLKMatrix4MakeFrustum(float left, float right, float bottom, float top, float nearZ, float farZ)

#### projecttionMatrix 和 modelviewMatrix

GLKitBaseEffect的tranform属性是一个GLKEffectPropertyTransform类型实例并为支持常见的操作保存了三个不同的矩阵。其中的两个矩阵projectionMatrix和modelviewMatrix分别定义了一个用于整个场景的坐标系和一个用于控制对象显示位置的坐标系。GLKBaseEffect会级联modelviewMatrix和projectionMatrix来产生一个modelViewProjectionMatrix矩阵，这个矩阵会把对象顶点完全地变换到OpenGLES默认坐标系中。


#### 看向一个特定的3D位置
- GLKMatrix4MakeLookAt()：前三个参数是观察者眼睛的位置，接下来的3个参数指定观察者正在看下的位置。


### 一些优化方法
- 剔除看不见的细节。
- 简化渲染内容。
- 尽量减少缓冲复制，无论是从CPU到GPU（glBufferData()）还是从GPU到CPU（glReadPixel()）。
- 使用VAO，减少状态变化。
- 使用Instruments分析。


## GPUImage源码分析

### GPUImage的渲染管线
首先，GPUImage中有这么几个概念：

- output输出源，表示这个对象可以作为管线中的输出者，都是GPUImageOutput或它的子类。
- input输入源，表示这个对象可以作为管线中的输入者，都遵循GPUImageInput协议。
- filter滤镜，GPUImageFilter或它的子类，即是GPUImageOutput的子类又遵循GPUImageInput协议，表示这个对象既能接受输入又可以输出。一般接收一个输入后进行相应的处理（比如颜色处理、混合纹理、添加滤镜等）然后输出给渲染管线中的下一个输入源进行后续处理。

一个完整的渲染处理流程是这样的 output + （n * filter） + input ,n >= 0。

下面按照GPUImage源码结构中的公共类、纯输出源、滤镜、纯输入源的顺序依次分析。

--- 

### 公共类
#### GLProgram
封装了OpenGLES的着色器程序。把顶点着色器和片段着色器的GLSL源码传入，然后生成着色器程序编译、链接、使用。

#### GPUImageContext
一个单例，用来管理EAGLContext、着色器程序、GCD队列等。

- contextQueue:一个串行队列，通过dispatch_queue_set_specific设置了标识符。GPUImage中的输出都通过*runSynchronouslyOnVideoProcessingQueue*方法放在这个串行队列中执行。
- shaderProgramCache：缓存了着色器程序，一个字典，以[NSString stringWithFormat:@"V: %@ - F: %@", vertexShaderString, fragmentShaderString]为key，GLProgram为Value。
- shaderProgramUsageHistory ：一个数组，着色器程序使用历史记录。在使用一个着色器程序后，就会添加到这个数组中*（这段代码被注释了，不知道为啥，所以其实没什么用）*。
- framebufferCache：一个GPUImageFramebufferCache对象，用来缓存GPUImageFrameBuffer对象。

#### GPUImageFrameBuffer

封装了OpenGLES中的帧缓冲对象。

- generateFramebuffer :通过runSynchronouslyOnVideoProcessingQueue方法在GPUImage的全局串行队列中执行OpenGL的相关方法生成帧缓冲和纹理。
- GPUTextureOptions：对应OpenGL中纹理的相关属性，用来设置设置环绕方式、多级渐远纹理和纹理过滤。
- lock和unlock ： GPUImageFrameBuffer自己维护了一个引用计数，当执行lock方法时，这个GPUImageFrameBuffer的引用计数加1.执行unlock方法时，这个GPUImageFrameBuffer的引用计数减1，当引用计数为0时，会执行GPUImageContext的*returnFramebufferToCache*方法把这个GPUImageFrameBuffer对象放入到GPUImageContext 的 framebufferCache中缓存起来。

#### framebufferCache
---
### 滤镜
---
### 纯输入源
