# UMD 编译系统函数调用关系分析

## 构建系统架构

### 1. 顶层结构

```
umd/Makefile (顶层)
    ├─> compiler 目标
    │   ├─> core/src/compiler/Makefile
    │   └─> apps/compiler/Makefile
    │
    └─> runtime 目标
        ├─> core/src/runtime/Makefile
        └─> apps/runtime/Makefile
```

### 2. Makefile 包含关系

每个组件的 Makefile 遵循相同的包含模式：

```
组件 Makefile (例如: apps/compiler/Makefile)
    │
    ├─> include $(ROOT)/make/macros.mk
    │       定义: GET_LOCAL_DIR, MKDIR, TOBUILDDIR
    │
    ├─> include rules.mk (组件特定规则)
    │       定义: MODULE_SRCS, MODULE_CC, MODULE_INCLUDES, 等
    │       │
    │       └─> include $(ROOT)/make/module.mk
    │               │
    │               └─> include $(ROOT)/make/compile.mk
    │
    └─> 定义最终链接目标 (LIB 或 TEST_BIN)
```

### 3. 详细调用流程

#### 步骤 1: macros.mk
```makefile
# 定义宏函数
GET_LOCAL_DIR    # 获取当前 Makefile 所在目录
MKDIR            # 创建目标目录
TOBUILDDIR       # 添加 BUILDDIR 前缀
```

#### 步骤 2: rules.mk (组件特定)
```makefile
# 设置模块变量
LOCAL_DIR := $(GET_LOCAL_DIR)
MODULE_CC := gcc 或 $(TOOLCHAIN_PREFIX)gcc
MODULE_CPP := g++ 或 $(TOOLCHAIN_PREFIX)g++
MODULE_LD := ld 或 $(TOOLCHAIN_PREFIX)ld
MODULE_SRCS := [源文件列表]
MODULE_INCLUDES := [包含路径]
MODULE_CPPFLAGS += [C++编译标志]
MODULE_CFLAGS += [C编译标志]

# 包含 module.mk
include $(ROOT)/make/module.mk
```

#### 步骤 3: module.mk
```makefile
# 保存模块源目录
MODULE_SRCDIR := $(MODULE)

# 将编译标志转换为定义
MODULE_DEFINES += MODULE_COMPILEFLAGS="..."
MODULE_DEFINES += MODULE_CFLAGS="..."
MODULE_DEFINES += MODULE_CPPFLAGS="..."
MODULE_DEFINES += MODULE_LDFLAGS="..."
MODULE_DEFINES += MODULE_OPTFLAGS="..."
MODULE_DEFINES += MODULE_INCLUDES="..."

# 包含编译规则
include $(ROOT)/make/compile.mk

# 创建模块对象文件
MODULE_OBJECT := $(call TOBUILDDIR,$(MODULE_SRCDIR).mod.o)
$(MODULE_OBJECT): $(MODULE_OBJS) $(MODULE_EXTRA_OBJS)
    $(MODULE_LD) -r $^ -o $@

# 清理变量
MODULE := 
MODULE_SRCDIR := 
...
```

#### 步骤 4: compile.mk
```makefile
# 分离 C 和 C++ 源文件
MODULE_CSRCS := $(filter %.c,$(MODULE_SRCS))
MODULE_CPPSRCS := $(filter %.cpp,$(MODULE_SRCS))

# 生成对象文件列表
MODULE_COBJS := $(call TOBUILDDIR,$(patsubst %.c,%.o,$(MODULE_CSRCS)))
MODULE_CPPOBJS := $(call TOBUILDDIR,$(patsubst %.cpp,%.o,$(MODULE_CPPSRCS)))
MODULE_OBJS := $(MODULE_COBJS) $(MODULE_CPPOBJS)

# 设置每个对象的编译变量
$(MODULE_OBJS): MODULE_CC:=$(MODULE_CC)
$(MODULE_OBJS): MODULE_OPTFLAGS:=$(MODULE_OPTFLAGS)
$(MODULE_OBJS): MODULE_COMPILEFLAGS:=$(MODULE_COMPILEFLAGS)
$(MODULE_OBJS): MODULE_CFLAGS:=$(MODULE_CFLAGS)
$(MODULE_OBJS): MODULE_CPPFLAGS:=$(MODULE_CPPFLAGS)

# C 文件编译规则
$(MODULE_COBJS): $(BUILDDIR)/%.o: %.c
    $(MODULE_CC) $(MODULE_OPTFLAGS) $(MODULE_COMPILEFLAGS) $(MODULE_CFLAGS) \
                 $(MODULE_INCLUDES) $(INCLUDES) -c $< -o $@

# C++ 文件编译规则
$(MODULE_CPPOBJS): $(BUILDDIR)/%.o: %.cpp
    $(MODULE_CC) $(MODULE_OPTFLAGS) $(MODULE_COMPILEFLAGS) $(MODULE_CPPFLAGS) \
                 $(INCLUDES) $(MODULE_INCLUDES) -c $< -o $@

# 清理变量
MODULE_CSRCS := 
MODULE_CPPSRCS := 
MODULE_COBJS := 
MODULE_CPPOBJS := 
```

#### 步骤 5: 组件 Makefile (最终链接)
```makefile
# 对于库 (core/src/*/Makefile):
$(LIB): $(ALLMODULE_OBJS)
    g++ -shared $(ALLMODULE_OBJS) -o $@

# 对于应用程序 (apps/*/Makefile):
$(TEST_BIN): $(ALLMODULE_OBJS) $(SHARED_LIBS)
    g++ $(ALLMODULE_OBJS) -L... -l... -o $@
```

### 4. 变量流向图

```
rules.mk 中定义的变量:
    MODULE_SRCS
    MODULE_CC / MODULE_CPP / MODULE_LD
    MODULE_INCLUDES
    MODULE_CPPFLAGS / MODULE_CFLAGS
    MODULE_COMPILEFLAGS
    MODULE_OPTFLAGS
        ↓
    传递给 module.mk
        ↓
    module.mk 将其转换为 MODULE_DEFINES
    并包含 compile.mk
        ↓
    compile.mk 使用这些变量:
        - 分离 C/C++ 源文件
        - 生成对象文件路径
        - 为每个对象设置编译参数
        - 定义编译规则
        ↓
    compile.mk 返回 MODULE_OBJS
        ↓
    module.mk 将 MODULE_OBJS 链接为 MODULE_OBJECT
    并设置 ALLMODULE_OBJS
        ↓
    组件 Makefile 使用 ALLMODULE_OBJS
    链接最终的库或可执行文件
```

### 5. 四个构建目标

#### A. libnvdla_compiler.so (编译器库)
```
core/src/compiler/Makefile
    ├─> rules.mk 定义源文件和编译选项
    ├─> compile.mk 编译 .c 和 .cpp 为 .o
    ├─> module.mk 收集所有 .o 文件
    └─> g++ -shared 链接为 libnvdla_compiler.so
```

#### B. nvdla_compiler (编译器应用)
```
apps/compiler/Makefile
    ├─> rules.mk 定义源文件和编译选项
    ├─> 依赖 libnvdla_compiler.so
    ├─> compile.mk 编译应用源文件
    ├─> module.mk 收集对象文件
    └─> g++ 链接应用程序和库
```

#### C. libnvdla_runtime.so (运行时库)
```
core/src/runtime/Makefile
    ├─> rules.mk 定义源文件和编译选项
    ├─> compile.mk 编译 .c 和 .cpp 为 .o
    ├─> module.mk 收集所有 .o 文件
    └─> $(TOOLCHAIN_PREFIX)g++ -shared 链接为 libnvdla_runtime.so
```

#### D. nvdla_runtime (运行时应用)
```
apps/runtime/Makefile
    ├─> rules.mk 定义源文件和编译选项
    ├─> 依赖 libnvdla_runtime.so
    ├─> compile.mk 编译应用源文件
    ├─> module.mk 收集对象文件
    └─> $(TOOLCHAIN_PREFIX)g++ 链接应用程序和库
```

### 6. 关键设计模式

1. **模板化构建**: 所有组件使用相同的构建流程模板
2. **变量隔离**: module.mk 在处理完后清空所有 MODULE_* 变量
3. **递归包含**: rules.mk → module.mk → compile.mk 的链式包含
4. **宏函数**: macros.mk 提供可重用的辅助函数
5. **分离编译**: C 和 C++ 文件使用不同的编译规则

### 7. 依赖关系

```
编译顺序:
1. core/src/compiler (libnvdla_compiler.so)
2. apps/compiler (nvdla_compiler) - 依赖步骤1
3. core/src/runtime (libnvdla_runtime.so)
4. apps/runtime (nvdla_runtime) - 依赖步骤3

并行可能性:
- 步骤1和步骤3可以并行
- 但顶层 Makefile 是串行执行的
```

### 8. 外部依赖

- **protobuf-2.6**: 编译器需要 protobuf 库
- **libjpeg**: 运行时需要 jpeg 库
- **TOOLCHAIN_PREFIX**: 运行时组件使用交叉编译工具链
- **TOP 环境变量**: 必须在构建前设置

## 总结

UMD 编译系统采用了高度模块化的设计，通过多层 Makefile 包含实现代码重用。核心流程是：

1. 各组件的 rules.mk 定义源文件和编译参数
2. compile.mk 将源文件编译为对象文件
3. module.mk 将对象文件组合为模块对象
4. 组件 Makefile 最终链接生成库或可执行文件

这种设计允许每个组件复用相同的构建逻辑，但也引入了一些前述报告中指出的不一致性问题。
