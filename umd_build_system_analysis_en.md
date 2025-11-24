# UMD Build System Function Call Analysis

## Build System Architecture

### 1. Top-Level Structure

```
umd/Makefile (top-level)
    ├─> compiler target
    │   ├─> core/src/compiler/Makefile
    │   └─> apps/compiler/Makefile
    │
    └─> runtime target
        ├─> core/src/runtime/Makefile
        └─> apps/runtime/Makefile
```

### 2. Makefile Include Relationships

Each component's Makefile follows the same include pattern:

```
Component Makefile (e.g., apps/compiler/Makefile)
    │
    ├─> include $(ROOT)/make/macros.mk
    │       Defines: GET_LOCAL_DIR, MKDIR, TOBUILDDIR
    │
    ├─> include rules.mk (component-specific rules)
    │       Defines: MODULE_SRCS, MODULE_CC, MODULE_INCLUDES, etc.
    │       │
    │       └─> include $(ROOT)/make/module.mk
    │               │
    │               └─> include $(ROOT)/make/compile.mk
    │
    └─> Define final link target (LIB or TEST_BIN)
```

### 3. Detailed Call Flow

#### Step 1: macros.mk
```makefile
# Define macro functions
GET_LOCAL_DIR    # Get directory of current Makefile
MKDIR            # Create target directory
TOBUILDDIR       # Add BUILDDIR prefix
```

#### Step 2: rules.mk (component-specific)
```makefile
# Set module variables
LOCAL_DIR := $(GET_LOCAL_DIR)
MODULE_CC := gcc or $(TOOLCHAIN_PREFIX)gcc
MODULE_CPP := g++ or $(TOOLCHAIN_PREFIX)g++
MODULE_LD := ld or $(TOOLCHAIN_PREFIX)ld
MODULE_SRCS := [source file list]
MODULE_INCLUDES := [include paths]
MODULE_CPPFLAGS += [C++ compile flags]
MODULE_CFLAGS += [C compile flags]

# Include module.mk
include $(ROOT)/make/module.mk
```

#### Step 3: module.mk
```makefile
# Save module source directory
MODULE_SRCDIR := $(MODULE)

# Convert compile flags to definitions
MODULE_DEFINES += MODULE_COMPILEFLAGS="..."
MODULE_DEFINES += MODULE_CFLAGS="..."
MODULE_DEFINES += MODULE_CPPFLAGS="..."
MODULE_DEFINES += MODULE_LDFLAGS="..."
MODULE_DEFINES += MODULE_OPTFLAGS="..."
MODULE_DEFINES += MODULE_INCLUDES="..."

# Include compilation rules
include $(ROOT)/make/compile.mk

# Create module object file
MODULE_OBJECT := $(call TOBUILDDIR,$(MODULE_SRCDIR).mod.o)
$(MODULE_OBJECT): $(MODULE_OBJS) $(MODULE_EXTRA_OBJS)
    $(MODULE_LD) -r $^ -o $@

# Clear variables
MODULE := 
MODULE_SRCDIR := 
...
```

#### Step 4: compile.mk
```makefile
# Separate C and C++ source files
MODULE_CSRCS := $(filter %.c,$(MODULE_SRCS))
MODULE_CPPSRCS := $(filter %.cpp,$(MODULE_SRCS))

# Generate object file list
MODULE_COBJS := $(call TOBUILDDIR,$(patsubst %.c,%.o,$(MODULE_CSRCS)))
MODULE_CPPOBJS := $(call TOBUILDDIR,$(patsubst %.cpp,%.o,$(MODULE_CPPSRCS)))
MODULE_OBJS := $(MODULE_COBJS) $(MODULE_CPPOBJS)

# Set compilation variables for each object
$(MODULE_OBJS): MODULE_CC:=$(MODULE_CC)
$(MODULE_OBJS): MODULE_OPTFLAGS:=$(MODULE_OPTFLAGS)
$(MODULE_OBJS): MODULE_COMPILEFLAGS:=$(MODULE_COMPILEFLAGS)
$(MODULE_OBJS): MODULE_CFLAGS:=$(MODULE_CFLAGS)
$(MODULE_OBJS): MODULE_CPPFLAGS:=$(MODULE_CPPFLAGS)

# C file compilation rule
$(MODULE_COBJS): $(BUILDDIR)/%.o: %.c
    $(MODULE_CC) $(MODULE_OPTFLAGS) $(MODULE_COMPILEFLAGS) $(MODULE_CFLAGS) \
                 $(MODULE_INCLUDES) $(INCLUDES) -c $< -o $@

# C++ file compilation rule
$(MODULE_CPPOBJS): $(BUILDDIR)/%.o: %.cpp
    $(MODULE_CC) $(MODULE_OPTFLAGS) $(MODULE_COMPILEFLAGS) $(MODULE_CPPFLAGS) \
                 $(INCLUDES) $(MODULE_INCLUDES) -c $< -o $@

# Clear variables
MODULE_CSRCS := 
MODULE_CPPSRCS := 
MODULE_COBJS := 
MODULE_CPPOBJS := 
```

#### Step 5: Component Makefile (final linking)
```makefile
# For libraries (core/src/*/Makefile):
$(LIB): $(ALLMODULE_OBJS)
    g++ -shared $(ALLMODULE_OBJS) -o $@

# For applications (apps/*/Makefile):
$(TEST_BIN): $(ALLMODULE_OBJS) $(SHARED_LIBS)
    g++ $(ALLMODULE_OBJS) -L... -l... -o $@
```

### 4. Variable Flow Diagram

```
Variables defined in rules.mk:
    MODULE_SRCS
    MODULE_CC / MODULE_CPP / MODULE_LD
    MODULE_INCLUDES
    MODULE_CPPFLAGS / MODULE_CFLAGS
    MODULE_COMPILEFLAGS
    MODULE_OPTFLAGS
        ↓
    Passed to module.mk
        ↓
    module.mk converts them to MODULE_DEFINES
    and includes compile.mk
        ↓
    compile.mk uses these variables to:
        - Separate C/C++ source files
        - Generate object file paths
        - Set compile parameters for each object
        - Define compilation rules
        ↓
    compile.mk returns MODULE_OBJS
        ↓
    module.mk links MODULE_OBJS into MODULE_OBJECT
    and sets ALLMODULE_OBJS
        ↓
    Component Makefile uses ALLMODULE_OBJS
    to link final library or executable
```

### 5. Four Build Targets

#### A. libnvdla_compiler.so (compiler library)
```
core/src/compiler/Makefile
    ├─> rules.mk defines source files and compile options
    ├─> compile.mk compiles .c and .cpp to .o
    ├─> module.mk collects all .o files
    └─> g++ -shared links to libnvdla_compiler.so
```

#### B. nvdla_compiler (compiler application)
```
apps/compiler/Makefile
    ├─> rules.mk defines source files and compile options
    ├─> depends on libnvdla_compiler.so
    ├─> compile.mk compiles application source files
    ├─> module.mk collects object files
    └─> g++ links application and library
```

#### C. libnvdla_runtime.so (runtime library)
```
core/src/runtime/Makefile
    ├─> rules.mk defines source files and compile options
    ├─> compile.mk compiles .c and .cpp to .o
    ├─> module.mk collects all .o files
    └─> $(TOOLCHAIN_PREFIX)g++ -shared links to libnvdla_runtime.so
```

#### D. nvdla_runtime (runtime application)
```
apps/runtime/Makefile
    ├─> rules.mk defines source files and compile options
    ├─> depends on libnvdla_runtime.so
    ├─> compile.mk compiles application source files
    ├─> module.mk collects object files
    └─> $(TOOLCHAIN_PREFIX)g++ links application and library
```

### 6. Key Design Patterns

1. **Templated Build**: All components use the same build flow template
2. **Variable Isolation**: module.mk clears all MODULE_* variables after processing
3. **Recursive Inclusion**: Chain of rules.mk → module.mk → compile.mk
4. **Macro Functions**: macros.mk provides reusable helper functions
5. **Separate Compilation**: C and C++ files use different compilation rules

### 7. Dependencies

```
Build Order:
1. core/src/compiler (libnvdla_compiler.so)
2. apps/compiler (nvdla_compiler) - depends on step 1
3. core/src/runtime (libnvdla_runtime.so)
4. apps/runtime (nvdla_runtime) - depends on step 3

Parallelization Potential:
- Steps 1 and 3 can run in parallel
- But top-level Makefile executes serially
```

### 8. External Dependencies

- **protobuf-2.6**: Compiler requires protobuf library
- **libjpeg**: Runtime requires jpeg library
- **TOOLCHAIN_PREFIX**: Runtime components use cross-compilation toolchain
- **TOP environment variable**: Must be set before building

## Summary

The UMD build system uses a highly modular design, achieving code reuse through multiple layers of Makefile inclusion. The core flow is:

1. Each component's rules.mk defines source files and compile parameters
2. compile.mk compiles source files into object files
3. module.mk combines object files into a module object
4. Component Makefile finally links to generate library or executable

This design allows each component to reuse the same build logic, but it also introduces some inconsistency issues as noted in the previous error report.
