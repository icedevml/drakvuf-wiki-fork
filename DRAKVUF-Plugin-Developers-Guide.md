DRAKVUF Plugin Developers Guide
===============================

Since DRAKVUF 0.2 new functionality can be added in form of independent plugins. 
This guide aims to help developing new plugins.

File structure
--------------

Plugins should be placed in the `src/plugins` directory. It is highly recommended to choose a unique letters-only name for your plugins and use it consistently. In this guide our imaginary plugin will be called `myplugin` and will be referenced with the lower- or upper-case version of this name depending on the context. 

You should create at least the following files for your plugin:
  * `src/plugins/myplugin/myplugin.c` - Main plugin functionality
  * `src/plugins/myplugin/myplugin.h` - Function headers

Some plugins contain a separate `private.h` header too that contains definitions for structures internally used by the plugin. You can further add more source files and private headers as you see fit during the development of your plugin.

Plugin interfaces
-----------------

Each plugin must adhere to the plugin interface defined by DRAKVUF in order to be properly loaded. You should implement the following functions that will serve as the main interaction points between your plugin and DRAKVUF:
  * `int plugin_myplugin_init(drakvuf_t drakvuf, const char *rekall_profile)` - Initialization: Collect information about where and how traps should be placed.
  * `int plugin_myplugin_start(drakvuf_t drakvuf)` - Startup: Traps are placed to the target VM based on the data collected during initialization.
  * `int plugin_myplugin_close(drakvuf_t drakvuf)` - Cleanup: Internal data structures and allocated memory should be deleted/freed here.

These functions should return 0 on error and 1 on success. Place the declaration of these functions to `myplugin.h`

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
sources += myplugin/myplugin.c
endif
```

before the `sources += plugins.c plugins.h` line (indicated by comments),

### src/plugins/plugins.h

Add `PLUGIN_MYPLUGIN` to the `drakvuf_plugin` enumeration

### src/plugins/private.h

Include your plugins header file:

```c
#include "myplugin/myplugin.h"
```

And extend the `plugins` array like this:

```c
static plugin_t plugins[] = { 
    // ...
    [PLUGIN_MYPLUGIN] = { .init = plugin_myplugin_init,
                          .start = plugin_myplugin_start,
                          .close = plugin_myplugin_close },
}
```