---
title: opengl入门(一)画一个三角形
tags: opengl
categories: opengl
date: 2020-08-19 09:04:57
img:
---


> opengl的hello world项目画一个三角形，写起来才知道opengl的细节有那么多。

### 步骤

在Android中开发opengl，可以通过GLSurfaceView

```kotlin
mGlSurface.setRenderer(MatrixRenderer(this))
```

这个Renderer是个接口类，主要在它的回调方法中做opengl元件的初始化和矩阵变换操作

```java
public interface Renderer {
    void onSurfaceCreated(GL10 gl, EGLConfig config);

    void onSurfaceChanged(GL10 gl, int width, int height);

    void onDrawFrame(GL10 gl);
}
```

opengl是一个状态机，通过编写glsl和进行矩阵变换，可以做到一些简单图形的显示以及旋转和位移。着色器有两种顶点着色器(vertex)和片元着色器(fragment)，顶点着色器用于定义绘制图案的顶点坐标，在opengl中有点、线、三角形三种图元，片元着色器用于绘制点与点之间的片段。

#### 定义顶点坐标

```kotlin
private val vertexPosition = floatArrayOf(
    0f, 0.5f,
    0.5f, 0f,
    -0.5f, 0f
)

// 一个float占4个字节
const val BYTES_PER_FLOAT = 4

// 放入buffer中
private val buffer = ByteBuffer
    //TODO allocate 与 allocateDirect的区别
    .allocateDirect(vertexPosition.size * BYTES_PER_FLOAT)
    .order(nativeOrder())
    .asFloatBuffer()
    .apply {
        put(vertexPosition)
    }
```

#### 定义顶点着色器/片元着色器

```glsl
// 顶点着色器
attribute vec4 u_Position;

void main()
{
    // 空间中点的位置
    gl_Position = u_Position;
    gl_PointSize = 10.0;
}
```

```glsl
// 片元着色器
precision mediump float;
uniform vec4 u_Color;

void main()
{
    // 片元的颜色
    gl_FragColor = u_Color;
}
```

#### 创建着色器链接/编译代码片段

```kotlin
private fun compileShader(type: Int, shaderCode: String): Int {
    // 按类型创造着色器
    val shaderId = glCreateShader(type)
    if (shaderId == 0) {
        d(TAG, "could not create new shader")
        return 0
    }
    // 上传着色器代码到着色器对象中
    glShaderSource(shaderId, shaderCode)
    // 编译着色器
    glCompileShader(shaderId)
    // 编译状态
    val status = IntArray(1)
    glGetShaderiv(shaderId, GL_COMPILE_STATUS, status, 0)
    // 打印着色器日志
    d(TAG, "result of compiling source:\n$shaderCode\n:${glGetShaderInfoLog(shaderId)}")
    if (status[0] == 0) {
        glDeleteShader(shaderId)
        d(TAG, "compilation of shader failed")
        return 0
    }
    return shaderId
}
```

#### 创建渲染程序链接着色器

这一步在onSurfaceCreate回调中实现

```kotlin
override fun onSurfaceCreated(gl: GL10?, config: EGLConfig?) {
    glClearColor(0f, 0f, 0f, 0f)
    val vertexShaderId =
        compileVertexShader(readShaderFromSource(context, R.raw.matrix_vertex_shader))
    val fragmentShaderId =
        compileFragmentShader(readShaderFromSource(context, R.raw.matrix_fragment_shader))

    // 链接着色器
    val programId = linkProgram(vertexShaderId, fragmentShaderId)

    // 验证程序id的有效性
    validateProgram(programId)
    glUseProgram(programId)
    // 获取着色器中变量在内存中的地址
    uPositionLocation = glGetAttribLocation(programId, uPosition)
    uColorLocation = glGetUniformLocation(programId, uColor)

    // buffer在上一步的赋值中position已经发生了改变，重置它，使顶点着色器从第一个顶点开始读起
    buffer.position(0)

    glVertexAttribPointer(
        uPositionLocation,
        POSITION_COMPONENT_COUNT,
        GL_FLOAT,
        false,
        0,
        buffer
    )
    glEnableVertexAttribArray(uPositionLocation)
}
```

#### 绘制三角形/着色

```kotlin
override fun onDrawFrame(gl: GL10?) {
    glClear(GL_COLOR_BUFFER_BIT)

    // 修改三角形的颜色
    glUniform4f(uColorLocation, 1f, 0f, 0f, 1f)
    glDrawArrays(GL_TRIANGLES, 0, 3)
}
```

### 结语

经过上述步骤后，一个简易的三角形就被绘制出来了。

![ ](https://dreamweaver.img.we1code.cn/triangle.png)
