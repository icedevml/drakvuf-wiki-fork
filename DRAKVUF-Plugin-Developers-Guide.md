Since DRAKVUF 0.2 new functionality can be added in form of independent plugins. 
This guide aims to help developing new plugins.

File structure
--------------

Plugins should be placed in the `src/plugins` directory. It is highly recommended to choose a unique letters-only name for your plugins and use it consistently. In this guide our imaginary plugin will be called `myplugin` and will be referenced with the lower- or upper-case version of this name depending on the context. 

You should create at least the following files for your plugin:
  * `src/plugins/myplugin/myplugin.cpp` - Main plugin functionality
  * `src/plugins/myplugin/myplugin.h` - Class definition

Some plugins contain a separate `private.h` header too that contains definitions for structures internally used by the plugin. You can further add more source files and private headers as you see fit during the development of your plugin.

Plugin interfaces
-----------------

Each plugin is its own C++ class, where the class definition is placed in `myplugin.h`. The two requirements DRAKVUF places on the the C++ class is that it needs to extend the dummy plugin class (`class myplugin: public plugin`), and define a constructor which takes three inputs: `myplugin::myplugin(drakvuf_t drakvuf, const void *config, output_format_t output)`.

Integration
-----------

In order to make your plugin compile and run along with other parts of DRAKVUF you will need to hook it into the build system of DRAKVUF. The necessary changes are described in subsections

### configure.ac

You should add the following lines to make the configuration script aware of your plugin so that its compilation can be enabled/disabled.

```
AC_ARG_ENABLE([plugin_myplugin],
  [AS_HELP_STRING([--disable-plugin-myplugin],
    [Enable the MYPLUGIN example plugin @<:@yes@:>@])],
  [plugin_myplugin="$enableval"],
  [plugin_myplugin="yes"])   
AM_CONDITIONAL([PLUGIN_MYPLUGIN], [test x$plugin_myplugin = xyes])
if test x$plugin_myplugin = xyes; then
  AC_DEFINE_UNQUOTED(ENABLE_PLUGIN_MYPLUGIN, 1, "")
fi
```

When you are testing whether your changes work, make sure you run ./autogen.sh before. When properly created, the plugin can be disabled at compile time by invoking `./configure` with the `--disable-plugin-myplugin` option.

To inform the user whether the plugin will be compiled or not, add the following at the end (in `AC_MSG_RESULT`):

```
MyPlugin:      $plugin_myplugin
```

### src/plugins/Makefile.am

To tell the build system what source-files need to be compiled in your plugin, add each as follows:

```
if PLUGIN_MYPLUGIN
sources += myplugin/myplugin.cpp
endif
```

before the `sources += plugins.cpp plugins.h` line (indicated by comments),

### src/plugins/plugins.h

Add `PLUGIN_MYPLUGIN` to the `drakvuf_plugin` enumeration

### src/plugins/plugins.cpp

Include your plugins header file:

```c
#include "myplugin/myplugin.h"
```

Add a block for your plugin in the start function's switch statement:

```c
#ifdef ENABLE_PLUGIN_MYPLUGIN
        case PLUGIN_MYPLUGIN:
            this->plugins[plugin_id] = new myplugin(this->drakvuf, config);
            break;
#endif
```

Passing state through libdrakvuf
-----------
In order to maintain the state of the plugin through libdrakvuf drakvuf_trap_t, the class' `this` pointer should be passed through the libdrakvuf trap's `data` field as such:

```c
trap->data = (void*)this;
```