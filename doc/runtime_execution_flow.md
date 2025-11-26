# NVDLA Runtime 程序执行流程分析

## 概述

本文档详细分析了 `umd/apps/runtime/` 目录下的运行时程序执行流程，从 `main` 函数的 `run()` 调用开始，直到程序结束，涵盖所有关键函数调用及其功能说明。

---

## 1. 程序入口 - main() 函数

**文件位置**: `umd/apps/runtime/main.cpp:94-251`

### 1.1 功能概述
`main()` 函数是程序的入口点，负责：
- 解析命令行参数
- 根据运行模式选择测试模式或服务器模式
- 返回执行结果

### 1.2 命令行参数解析

支持以下参数：
| 参数 | 功能 | 示例 |
|------|------|------|
| `-h` | 显示帮助信息 | `./nvdla_runtime -h` |
| `-s` | 启动服务器模式 | `./nvdla_runtime -s` |
| `-i` | 设置输入路径 | `./nvdla_runtime -i /path/to/input` |
| `--image` | 指定输入图像文件 | `./nvdla_runtime --image test.jpg` |
| `--loadable` | 指定 loadable 文件 | `./nvdla_runtime --loadable model.nvdla` |
| `--normalize` | 归一化值 | `./nvdla_runtime --normalize 255` |
| `--mean` | 均值（逗号分隔） | `./nvdla_runtime --mean 104,117,123` |
| `--rawdump` | 输出原始数据 | `./nvdla_runtime --rawdump` |

### 1.3 执行流程分支

```
main()
├─> 解析命令行参数
├─> 检查必需参数 (--loadable 或 -s)
│
├─> 如果是服务器模式 (-s)
│   └─> launchServer(&testAppArgs)
│       └─> runServer()
│
└─> 否则（测试模式）
    └─> launchTest(&testAppArgs)
        ├─> testSetup()
        └─> run()
```

---

## 2. 测试启动 - launchTest() 函数

**文件位置**: `umd/apps/runtime/main.cpp:79-91`

### 2.1 功能概述
初始化测试环境并启动测试运行。

### 2.2 函数签名
```cpp
static NvDlaError launchTest(const TestAppArgs* appArgs)
```

### 2.3 执行步骤

1. **初始化 TestInfo 结构体**
   - 设置 `dlaServerRunning = false`

2. **调用 testSetup()**
   - 验证输入路径和图像文件是否存在

3. **调用 run()**
   - 执行实际的推理测试

### 2.4 错误处理
使用 `PROPAGATE_ERROR_FAIL` 宏传播错误，任何步骤失败都会跳转到 `fail` 标签返回错误码。

---

## 3. 测试设置 - testSetup() 函数

**文件位置**: `umd/apps/runtime/main.cpp:41-65`

### 3.1 功能概述
验证输入路径和图像文件的有效性。

### 3.2 函数签名
```cpp
static NvDlaError testSetup(const TestAppArgs* appArgs, TestInfo* i)
```

### 3.3 执行步骤

1. **检查输入名称**
   - 如果 `appArgs->inputName` 不为空，继续验证

2. **验证输入路径**
   ```cpp
   e = NvDlaStat(appArgs->inputPath.c_str(), &stat);
   ```
   - 使用 `NvDlaStat` 检查路径是否存在

3. **验证图像文件**
   ```cpp
   imagePath = appArgs->inputName;
   e = NvDlaStat(imagePath.c_str(), &stat);
   ```
   - 检查图像文件是否存在

---

## 4. 核心执行 - run() 函数 ⭐

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:429-470`

### 4.1 功能概述
这是运行时测试的核心函数，负责：
- 创建运行时实例
- 加载 loadable 文件
- 初始化模拟器
- 执行推理测试
- 清理资源

### 4.2 函数签名
```cpp
NvDlaError run(const TestAppArgs* appArgs, TestInfo* i)
```

### 4.3 详细执行流程

```
run()
│
├─> 1. 创建运行时实例
│   i->runtime = nvdla::createRuntime()
│
├─> 2. 读取 Loadable 文件（非服务器模式）
│   readLoadable(appArgs, i)
│
├─> 3. 加载 Loadable 到运行时
│   loadLoadable(appArgs, i)
│
├─> 4. 初始化模拟器
│   i->runtime->initEMU()
│
├─> 5. 运行测试
│   runTest(appArgs, i)
│
└─> 清理阶段 (fail 标签)
    ├─> 停止模拟器: i->runtime->stopEMU()
    ├─> 卸载 Loadable: unloadLoadable()
    ├─> 释放数据缓冲区: delete[] i->pData
    └─> 销毁运行时: nvdla::destroyRuntime()
```

### 4.4 关键代码解析

```cpp
NvDlaError run(const TestAppArgs* appArgs, TestInfo* i)
{
    NvDlaError e = NvDlaSuccess;

    // 步骤1: 创建运行时上下文
    NvDlaDebugPrintf("creating new runtime context...\n");
    i->runtime = nvdla::createRuntime();
    if (i->runtime == NULL)
        ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "createRuntime() failed");

    // 步骤2: 读取 Loadable 文件
    if (!i->dlaServerRunning)
        PROPAGATE_ERROR_FAIL(readLoadable(appArgs, i));

    // 步骤3: 加载 Loadable
    PROPAGATE_ERROR_FAIL(loadLoadable(appArgs, i));

    // 步骤4: 初始化模拟器
    if (!i->runtime->initEMU())
        ORIGINATE_ERROR(NvDlaError_DeviceNotFound, "runtime->initEMU() failed");

    // 步骤5: 运行测试
    PROPAGATE_ERROR_FAIL(runTest(appArgs, i));

fail:
    // 清理工作...
}
```

---

## 5. 读取 Loadable 文件 - readLoadable() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:285-342`

### 5.1 功能概述
从文件系统读取 loadable 文件到内存。

### 5.2 函数签名
```cpp
static NvDlaError readLoadable(const TestAppArgs* appArgs, TestInfo* i)
```

### 5.3 执行步骤

1. **验证 loadable 名称**
   ```cpp
   if (appArgs->loadableName == "")
       ORIGINATE_ERROR_FAIL(NvDlaError_NotInitialized, "No loadable found to load");
   ```

2. **打开文件**
   ```cpp
   rc = NvDlaFopen(loadableName.c_str(), NVDLA_OPEN_READ, &file);
   ```

3. **获取文件信息**
   ```cpp
   rc = NvDlaFstat(file, &finfo);
   file_size = NvDlaStatGetSize(&finfo);
   ```

4. **分配缓冲区并读取**
   ```cpp
   buf = new NvU8[file_size];
   NvDlaFseek(file, 0, NvDlaSeek_Set);
   rc = NvDlaFread(file, buf, file_size, &actually_read);
   ```

5. **关闭文件并保存数据指针**
   ```cpp
   NvDlaFclose(file);
   i->pData = buf;
   ```

---

## 6. 加载 Loadable - loadLoadable() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:344-357`

### 6.1 功能概述
将内存中的 loadable 数据加载到运行时引擎。

### 6.2 函数签名
```cpp
NvDlaError loadLoadable(const TestAppArgs* appArgs, TestInfo* i)
```

### 6.3 执行步骤

1. **获取运行时实例**
   ```cpp
   nvdla::IRuntime* runtime = i->runtime;
   ```

2. **调用运行时 load 方法**
   ```cpp
   if (!runtime->load(i->pData, 0))
       ORIGINATE_ERROR_FAIL(NvDlaError_BadParameter, "runtime->load failed");
   ```

---

## 7. 运行测试 - runTest() 函数 ⭐

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:377-427`

### 7.1 功能概述
执行实际的神经网络推理测试，包括：
- 设置输入/输出缓冲区
- 提交推理任务
- 收集输出结果
- 保存输出文件

### 7.2 函数签名
```cpp
NvDlaError runTest(const TestAppArgs* appArgs, TestInfo* i)
```

### 7.3 详细执行流程

```
runTest()
│
├─> 1. 创建输入/输出图像对象
│   i->inputImage = new NvDlaImage()
│   i->outputImage = new NvDlaImage()
│
├─> 2. 设置输入缓冲区
│   setupInputBuffer(appArgs, i, &pInputBuffer)
│
├─> 3. 设置输出缓冲区
│   setupOutputBuffer(appArgs, i, &pOutputBuffer)
│
├─> 4. 记录开始时间
│   clock_gettime(CLOCK_MONOTONIC, &before)
│
├─> 5. 提交推理任务
│   runtime->submit()
│
├─> 6. 记录结束时间并输出耗时
│   clock_gettime(CLOCK_MONOTONIC, &after)
│   NvDlaDebugPrintf("execution time = %f s\n", ...)
│
├─> 7. 将输出缓冲区转换为图像
│   DlaBuffer2DIMG(&pOutputBuffer, i->outputImage)
│
├─> 8. 保存输出文件
│   DIMG2DIMGFile(i->outputImage, OUTPUT_DIMG, ...)
│
└─> 清理阶段 (fail 标签)
    ├─> cleanupOutputBuffer()
    ├─> delete i->outputImage
    ├─> cleanupInputBuffer()
    └─> delete i->inputImage
```

### 7.4 关键代码解析

```cpp
NvDlaError runTest(const TestAppArgs* appArgs, TestInfo* i)
{
    NvDlaError e = NvDlaSuccess;
    void* pInputBuffer = NULL;
    void* pOutputBuffer = NULL;
    struct timespec before, after;

    nvdla::IRuntime* runtime = i->runtime;

    // 创建图像对象
    i->inputImage = new NvDlaImage();
    i->outputImage = new NvDlaImage();

    // 设置输入缓冲区
    PROPAGATE_ERROR_FAIL(setupInputBuffer(appArgs, i, &pInputBuffer));

    // 设置输出缓冲区
    PROPAGATE_ERROR_FAIL(setupOutputBuffer(appArgs, i, &pOutputBuffer));

    // 提交推理任务
    NvDlaDebugPrintf("submitting tasks...\n");
    clock_gettime(CLOCK_MONOTONIC, &before);
    if (!runtime->submit())
        ORIGINATE_ERROR(NvDlaError_BadParameter, "runtime->submit() failed");
    clock_gettime(CLOCK_MONOTONIC, &after);

    // 输出执行时间
    NvDlaDebugPrintf("execution time = %f s\n", get_elapsed_time(&before,&after));

    // 转换输出数据
    PROPAGATE_ERROR_FAIL(DlaBuffer2DIMG(&pOutputBuffer, i->outputImage));

    // 保存输出文件
    PROPAGATE_ERROR_FAIL(DIMG2DIMGFile(i->outputImage, OUTPUT_DIMG, true, appArgs->rawOutputDump));

fail:
    // 清理资源...
}
```

---

## 8. 设置输入缓冲区 - setupInputBuffer() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:136-170`

### 8.1 功能概述
为神经网络输入创建并配置内存缓冲区。

### 8.2 函数签名
```cpp
NvDlaError setupInputBuffer(
    const TestAppArgs* appArgs,
    TestInfo* i,
    void** pInputBuffer
)
```

### 8.3 执行步骤

1. **获取输入张量数量**
   ```cpp
   runtime->getNumInputTensors(&numInputTensors);
   ```

2. **获取输入张量描述符**
   ```cpp
   runtime->getInputTensorDesc(0, &tDesc);
   ```

3. **分配系统内存**
   ```cpp
   runtime->allocateSystemMemory(&hMem, tDesc.bufferSize, pInputBuffer);
   i->inputHandle = (NvU8 *)hMem;
   ```

4. **复制图像数据到输入张量**
   ```cpp
   copyImageToInputTensor(appArgs, i, pInputBuffer, &tDesc);
   ```

5. **绑定输入张量**
   ```cpp
   runtime->bindInputTensor(0, hMem);
   ```

---

## 9. 复制图像到输入张量 - copyImageToInputTensor() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:69-116`

### 9.1 功能概述
读取输入图像文件并转换为网络所需的张量格式。

### 9.2 函数签名
```cpp
static NvDlaError copyImageToInputTensor(
    const TestAppArgs* appArgs,
    TestInfo* i,
    void** pImgBuffer,
    nvdla::IRuntime::NvDlaTensor *tensorDesc
)
```

### 9.3 执行步骤

1. **创建临时图像对象**
   ```cpp
   NvDlaImage* R8Image = new NvDlaImage();
   ```

2. **确定图像类型**
   ```cpp
   TestImageTypes imageType = getImageType(imgPath);
   ```

3. **根据类型解析图像**
   ```cpp
   switch(imageType) {
       case IMAGE_TYPE_PGM:
           PGM2DIMG(imgPath, R8Image, tensorDesc);
           break;
       case IMAGE_TYPE_JPG:
           JPEG2DIMG(imgPath, R8Image, tensorDesc);
           break;
   }
   ```

4. **创建图像副本并转换格式**
   ```cpp
   createImageCopy(appArgs, R8Image, tensorDesc, tensorImage);
   ```

5. **转换为 DLA 缓冲区格式**
   ```cpp
   DIMG2DlaBuffer(tensorImage, pImgBuffer);
   ```

---

## 10. 设置输出缓冲区 - setupOutputBuffer() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:208-248`

### 10.1 功能概述
为神经网络输出创建并配置内存缓冲区。

### 10.2 函数签名
```cpp
NvDlaError setupOutputBuffer(
    const TestAppArgs* appArgs,
    TestInfo* i,
    void** pOutputBuffer
)
```

### 10.3 执行步骤

1. **获取输出张量数量**
   ```cpp
   runtime->getNumOutputTensors(&numOutputTensors);
   ```

2. **获取输出张量描述符**
   ```cpp
   runtime->getOutputTensorDesc(0, &tDesc);
   ```

3. **分配系统内存**
   ```cpp
   runtime->allocateSystemMemory(&hMem, tDesc.bufferSize, pOutputBuffer);
   i->outputHandle = (NvU8 *)hMem;
   ```

4. **准备输出张量**
   ```cpp
   prepareOutputTensor(&tDesc, pOutputImage, pOutputBuffer, appArgs);
   ```

5. **绑定输出张量**
   ```cpp
   runtime->bindOutputTensor(0, hMem);
   ```

---

## 11. 图像处理辅助函数

### 11.1 getImageType() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:53-67`

**功能**: 根据文件扩展名判断图像类型

```cpp
static TestImageTypes getImageType(std::string imageFileName)
{
    std::string ext = imageFileName.substr(imageFileName.find_last_of(".") + 1);
    if (ext == "pgm") return IMAGE_TYPE_PGM;
    else if (ext == "jpg") return IMAGE_TYPE_JPG;
    return IMAGE_TYPE_UNKNOWN;
}
```

### 11.2 DIMG2DlaBuffer() 函数

**文件位置**: `umd/apps/runtime/TestUtils.cpp:41-49`

**功能**: 将 NvDlaImage 数据复制到 DLA 缓冲区

```cpp
NvDlaError DIMG2DlaBuffer(const NvDlaImage* image, void** pBuffer)
{
    memcpy(*pBuffer, image->m_pData, image->m_meta.size);
    return NvDlaSuccess;
}
```

### 11.3 DlaBuffer2DIMG() 函数

**文件位置**: `umd/apps/runtime/TestUtils.cpp:51-59`

**功能**: 将 DLA 缓冲区数据复制到 NvDlaImage

```cpp
NvDlaError DlaBuffer2DIMG(void** pBuffer, NvDlaImage* image)
{
    memcpy(image->m_pData, *pBuffer, image->m_meta.size);
    return NvDlaSuccess;
}
```

### 11.4 createImageCopy() 函数

**文件位置**: `umd/apps/runtime/TestUtils.cpp:146-273`

**功能**: 创建符合张量描述符要求的图像副本，执行格式转换和归一化

主要步骤：
1. 设置输出图像的维度信息
2. 根据像素格式设置表面格式
3. 分配输出缓冲区
4. 遍历每个像素进行数据转换
5. 对于 FP16 格式：应用归一化 `(value - mean) / normalize_value`
6. 对于 INT8 格式：压缩值范围 `value * 127.0 / 255.0`

---

## 12. 清理函数

### 12.1 cleanupInputBuffer() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:172-206`

**功能**: 清理输入缓冲区资源

```cpp
static void cleanupInputBuffer(const TestAppArgs *appArgs, TestInfo *i)
{
    // 释放输入图像数据
    if (i->inputImage != NULL && i->inputImage->m_pData != NULL) {
        NvDlaFree(i->inputImage->m_pData);
        i->inputImage->m_pData = NULL;
    }
    
    // 释放运行时分配的内存
    runtime->freeSystemMemory(i->inputHandle, tDesc.bufferSize);
}
```

### 12.2 cleanupOutputBuffer() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:250-283`

**功能**: 清理输出缓冲区资源

```cpp
static void cleanupOutputBuffer(const TestAppArgs *appArgs, TestInfo *i)
{
    // 仅在非服务器模式下释放输出图像数据
    if (!i->dlaServerRunning && i->outputImage != NULL && 
        i->outputImage->m_pData != NULL) {
        NvDlaFree(i->outputImage->m_pData);
    }
    
    // 释放运行时分配的内存
    runtime->freeSystemMemory(i->outputHandle, tDesc.bufferSize);
}
```

### 12.3 unloadLoadable() 函数

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:359-368`

**功能**: 卸载已加载的 loadable

```cpp
void unloadLoadable(const TestAppArgs* appArgs, TestInfo *i)
{
    if (i->runtime != NULL) {
        i->runtime->unload();
    }
}
```

---

## 13. 服务器模式 - runServer() 函数

**文件位置**: `umd/apps/runtime/Server.cpp:657-686`

### 13.1 功能概述
启动 TCP 服务器，监听客户端连接并处理推理请求。

### 13.2 函数签名
```cpp
NvDlaError runServer(const TestAppArgs* appArgs, TestInfo *testInfo)
```

### 13.3 执行流程

```
runServer()
│
├─> 1. 验证端口号 (> 1024)
│
├─> 2. 启动服务器
│   startServer(appArgs, testInfo)
│
└─> 3. 主循环
    while (dlaServerRunning)
    │
    ├─> 等待客户端连接
    │   accept(dlaServerSock, ...)
    │
    └─> 处理客户端请求
        mainEventLoop(appArgs, testInfo)
```

### 13.4 支持的命令

| 命令 | 处理函数 | 功能 |
|------|----------|------|
| `GET_WELCOME` | `processWelcomeMessage()` | 发送欢迎消息 |
| `QUERY_FLATBUF` | `processQueryCachedFlatbuf()` | 查询是否有缓存的 flatbuf |
| `RUN_FLATBUF` | `processRunFlatbuf()` | 使用缓存的 flatbuf 运行测试 |
| `READ_FLATBUF` | `processReadFlatbuf()` | 读取 flatbuf 数据 |
| `RUN_IMAGE_*` | `processRunImage()` | 运行图像推理 |
| `GET_NUMOUTPUTS` | `processGetNumOutputs()` | 获取输出数量 |
| `GET_OUTPUT` | `processGetOutput()` | 获取输出数据 |
| `SHUTDOWN` | `processShutDownServer()` | 关闭服务器 |

---

## 14. 数据结构说明

### 14.1 TestAppArgs 结构体

**文件位置**: `umd/apps/runtime/RuntimeTest.h:42-61`

```cpp
struct TestAppArgs
{
    std::string inputPath;      // 输入文件路径
    std::string inputName;      // 输入图像名称
    std::string loadableName;   // loadable 文件名
    NvS32 serverPort;           // 服务器端口 (默认 6666)
    NvU8 normalize_value;       // 归一化值 (默认 1)
    float mean[4];              // 均值数组 (默认 {0, 0, 0, 0})
    bool rawOutputDump;         // 是否输出原始数据
};
```

### 14.2 TestInfo 结构体

**文件位置**: `umd/apps/runtime/RuntimeTest.h:63-92`

```cpp
struct TestInfo
{
    nvdla::IRuntime* runtime;   // 运行时实例指针
    std::string inputLoadablePath;
    NvU8 *inputHandle;          // 输入缓冲区句柄
    NvU8 *outputHandle;         // 输出缓冲区句柄
    NvU8 *pData;                // loadable 数据指针
    bool dlaServerRunning;      // 服务器运行状态
    NvS32 dlaRemoteSock;        // 远程 socket
    NvS32 dlaServerSock;        // 服务器 socket
    NvU32 numInputs;            // 输入数量
    NvU32 numOutputs;           // 输出数量
    NvDlaImage* inputImage;     // 输入图像
    NvDlaImage* outputImage;    // 输出图像
};
```

### 14.3 NvDlaImage 类

**文件位置**: `umd/apps/runtime/DlaImage.h:38-131`

```cpp
class NvDlaImage
{
public:
    // 像素格式枚举（支持 48 种格式）
    typedef enum _PixelFormat { ... } PixelFormat;
    
    // 元数据
    struct Metadata {
        PixelFormat surfaceFormat;
        NvU32 width, height, channel;
        NvU32 lineStride, surfaceStride, size;
    } m_meta;
    
    void* m_pData;  // 图像数据指针
    
    // 成员函数
    NvS8 getBpe() const;          // 获取每元素字节数
    NvS32 getAddrOffset(...);     // 计算地址偏移
    NvDlaError serialize(...);     // 序列化
    NvDlaError deserialize(...);   // 反序列化
};
```

---

## 15. 完整调用序列图

```
main()
│
├─> 参数解析
│
└─> launchTest()
    │
    ├─> testSetup()
    │   └─> NvDlaStat() [验证文件存在]
    │
    └─> run()
        │
        ├─> nvdla::createRuntime()
        │
        ├─> readLoadable()
        │   ├─> NvDlaFopen()
        │   ├─> NvDlaFstat()
        │   ├─> NvDlaFread()
        │   └─> NvDlaFclose()
        │
        ├─> loadLoadable()
        │   └─> runtime->load()
        │
        ├─> runtime->initEMU()
        │
        ├─> runTest()
        │   │
        │   ├─> setupInputBuffer()
        │   │   ├─> runtime->getNumInputTensors()
        │   │   ├─> runtime->getInputTensorDesc()
        │   │   ├─> runtime->allocateSystemMemory()
        │   │   ├─> copyImageToInputTensor()
        │   │   │   ├─> getImageType()
        │   │   │   ├─> PGM2DIMG() 或 JPEG2DIMG()
        │   │   │   ├─> createImageCopy()
        │   │   │   └─> DIMG2DlaBuffer()
        │   │   └─> runtime->bindInputTensor()
        │   │
        │   ├─> setupOutputBuffer()
        │   │   ├─> runtime->getNumOutputTensors()
        │   │   ├─> runtime->getOutputTensorDesc()
        │   │   ├─> runtime->allocateSystemMemory()
        │   │   ├─> prepareOutputTensor()
        │   │   │   ├─> Tensor2DIMG()
        │   │   │   └─> DIMG2DlaBuffer()
        │   │   └─> runtime->bindOutputTensor()
        │   │
        │   ├─> runtime->submit()  [执行推理]
        │   │
        │   ├─> DlaBuffer2DIMG()
        │   │
        │   ├─> DIMG2DIMGFile()  [保存输出]
        │   │
        │   ├─> cleanupOutputBuffer()
        │   │   └─> runtime->freeSystemMemory()
        │   │
        │   └─> cleanupInputBuffer()
        │       └─> runtime->freeSystemMemory()
        │
        ├─> runtime->stopEMU()
        │
        ├─> unloadLoadable()
        │   └─> runtime->unload()
        │
        └─> nvdla::destroyRuntime()
```

---

## 16. 错误处理机制

### 16.1 错误宏定义

**文件位置**: 引用自 `ErrorMacros.h`

| 宏 | 功能 |
|----|------|
| `ORIGINATE_ERROR` | 生成新错误并返回 |
| `ORIGINATE_ERROR_FAIL` | 生成错误并跳转到 fail 标签 |
| `PROPAGATE_ERROR` | 传播已存在的错误 |
| `PROPAGATE_ERROR_FAIL` | 传播错误并跳转到 fail 标签 |
| `REPORT_ERROR` | 报告错误但继续执行 |

### 16.2 错误码类型

```cpp
typedef enum {
    NvDlaSuccess = 0,
    NvDlaError_BadParameter,
    NvDlaError_NotInitialized,
    NvDlaError_InsufficientMemory,
    NvDlaError_DeviceNotFound,
    NvDlaError_FileOperationFailed,
    NvDlaError_NotSupported,
    NvDlaError_TestApplicationFailed,
    // ...
} NvDlaError;
```

---

## 17. 性能测量

**文件位置**: `umd/apps/runtime/RuntimeTest.cpp:370-375`

```cpp
double get_elapsed_time(struct timespec *before, struct timespec *after)
{
    double deltat_s  = (after->tv_sec - before->tv_sec) * 1000000;
    double deltat_ns = (after->tv_nsec - before->tv_nsec) / 1000;
    return deltat_s + deltat_ns;  // 返回微秒
}
```

在 `runTest()` 中使用：
```cpp
clock_gettime(CLOCK_MONOTONIC, &before);
runtime->submit();
clock_gettime(CLOCK_MONOTONIC, &after);
NvDlaDebugPrintf("execution time = %f s\n", get_elapsed_time(&before,&after));
```

---

## 18. 总结

NVDLA Runtime 程序的执行流程可以概括为以下关键阶段：

1. **初始化阶段**
   - 解析命令行参数
   - 验证输入文件
   - 创建运行时实例

2. **加载阶段**
   - 读取 loadable 文件到内存
   - 加载到运行时引擎
   - 初始化模拟器

3. **执行阶段**
   - 配置输入/输出缓冲区
   - 读取并预处理输入图像
   - 提交推理任务
   - 收集输出结果

4. **清理阶段**
   - 释放缓冲区内存
   - 停止模拟器
   - 卸载 loadable
   - 销毁运行时实例

程序设计采用了模块化架构，将图像处理、张量操作、运行时管理等功能分离到不同的文件和函数中，便于维护和扩展。
