.. highlight:: bash

编译 PHP 扩展
=======================

现在已经知道如何编译 PHP 了，现在将继续编译其它扩展。将讨论编译过程的工作原理以及不同的有效选项。

加载共享扩展
-------------------------

正如上一节中所说，PHP 扩展可以静态编译到 PHP 二进制文件中，也可以编译到共享对象（ ``.so``
）中。静态链接是大多数捆绑扩展的默认设置，而共享对象可以通过手动传递 ``--enable-EXTNAME=shared`` 或 ``--with-EXTNAME=shared``
到 ``./configure`` 来创建。

While static extensions will always be available, shared extensions need to be loaded using the ``extension`` or
``zend_extension`` ini options. Both options take either an absolute path to the ``.so`` file or a path relative to
the ``extension_dir`` setting.

As an example, consider a PHP build compiled using this configure line::

    ~/php-src> ./configure --prefix=$HOME/myphp \
                           --enable-debug --enable-maintainer-zts \
                           --enable-opcache --with-gmp=shared

In this case both the opcache extension and GMP extension are compiled into shared objects located in the ``modules/``
directory. You can load both either by changing the ``extension_dir`` or by passing absolute paths::

    ~/php-src> sapi/cli/php -dzend_extension=`pwd`/modules/opcache.so \
                            -dextension=`pwd`/modules/gmp.so
    # or
    ~/php-src> sapi/cli/php -dextension_dir=`pwd`/modules \
                            -dzend_extension=opcache.so -dextension=gmp.so

    # or (since PHP 7.2 the .so is optional)
    ~/php-src> sapi/cli/php -dextension_dir=`pwd`/modules \
                            -dzend_extension=opcache -dextension=gmp

During the ``make install`` step, both ``.so`` files will be moved into the extension directory of your PHP installation,
which you may find using the ``php-config --extension-dir`` command. For the above build options it will be
``/home/myuser/myphp/lib/php/extensions/no-debug-non-zts-MODULE_API``. This value will also be the default of the
``extension_dir`` ini option, so you won't have to specify it explicitly and can load the extensions directly::

    ~/myphp> bin/php -dzend_extension=opcache -dextension=gmp

This leaves us with one question: Which mechanism should you use? Shared objects allow you to have a base PHP binary and
load additional extensions through the php.ini. Distributions make use of this by providing a bare PHP package and
distributing the extensions as separate packages. On the other hand, if you are compiling your own PHP binary, you
likely don't have need for this, because you already know which extensions you need.

As a rule of thumb, you'll use static linkage for the extensions bundled by PHP itself and use shared extensions for
everything else. The reason is simply that building external extensions as shared objects is easier (or at least less
intrusive), as you will see in a moment. Another benefit is that you can update the extension without rebuilding PHP.

.. note:: If you need information about the difference between extensions and Zend extensions, you :doc:`may have a
          look at the dedicated chapter <../extensions_design/zend_extensions>`.

从 PECL 安装扩展
-------------------------------

PECL_, the *PHP Extension Community Library*, offers a large number of extensions for PHP. When extensions are removed
from the main PHP distribution, they usually continue to exist in PECL. Similarly, many extensions that are now bundled
with PHP were previously PECL extensions.

If you specified ``--with-pear`` during the configuration stage of your PHP build, ``make install`` will download
and install PECL as a part of PEAR. You will find the ``pecl`` script in the ``$PREFIX/bin`` directory. Installing
extensions is now as simple as running ``pecl install EXTNAME``, e.g.::

    ~/myphp> bin/pecl install apcu

This command will download, compile and install the APCu_ extension. The result will be a ``apcu.so`` file in your
extension directory, which can then be loaded by passing the ``extension=apcu`` ini option.

While ``pecl install`` is very handy for the end-user, it is of little interest to extension developers. In the
following, we'll describe two ways to manually build extensions: Either by importing it into the main PHP source tree
(this allows static linkage) or by doing an external build (only shared).

.. _PECL: http://pecl.php.net
.. _APCu: http://pecl.php.net/package/APCu

添加扩展到 PHP 源代码树
----------------------------------------

There is no fundamental difference between a third-party extension and an extension bundled with PHP. As such you can
build an external extension simply by copying it into the PHP source tree and then using the usual build procedure.
We'll demonstrate this using APCu as an example.

First of all, you'll have to place the source code of the extension into the ``ext/EXTNAME`` directory of your PHP
source tree. If the extension is available via git, this is as simple as cloning the repository from within ``ext/``::

    ~/php-src/ext> git clone https://github.com/krakjoe/apcu.git

Alternatively you can also download a source tarball and extract it::

    /tmp> wget http://pecl.php.net/get/apcu-4.0.2.tgz
    /tmp> tar xzf apcu-4.0.2.tgz
    /tmp> mkdir ~/php-src/ext/apcu
    /tmp> cp -r apcu-4.0.2/. ~/php-src/ext/apcu

The extension will contain a ``config.m4`` file, which specifies extension-specific build instructions for use by
autoconf. To incorporate them into the ``./configure`` script, you'll have to run ``./buildconf`` again. To ensure that
the configure file is really regenerated, it is recommended to delete it beforehand::

    ~/php-src> rm configure && ./buildconf

You can now use the ``./config.nice`` script to add APCu to your existing configuration or start over with a completely
new configure line::

    ~/php-src> ./config.nice --enable-apcu
    # or
    ~/php-src> ./configure --enable-apcu # --other-options

Finally run ``make -jN`` to perform the actual build. As we didn't use ``--enable-apcu=shared`` the extension is
statically linked into the PHP binary, i.e. no additional actions are needed to make use of it. Obviously you can also
use ``make install`` to install the resulting binaries.

使用 ``phpize`` 编译扩展
------------------------------------

It is also possible to build extensions separately from PHP by making use of the ``phpize`` script that was already
mentioned in the :ref:`building_php` section.

``phpize`` plays a similar role as the ``./buildconf`` script used for PHP builds: First it will import the PHP build
system into your extension by copying files from ``$PREFIX/lib/php/build``. Among these files are ``php.m4``
(PHP's M4 macros), ``phpize.m4`` (which will be renamed to ``configure.ac`` in your extension and contains the main
build instructions) and ``run-tests.php``.

Then ``phpize`` will invoke autoconf to generate a ``./configure`` file, which can be used to customize the extension
build. Note that it is not necessary to pass ``--enable-apcu`` to it, as this is implicitly assumed. Instead you should
use ``--with-php-config`` to specify the path to your ``php-config`` script::

    /tmp/apcu-4.0.2> ~/myphp/bin/phpize
    Configuring for:
    PHP Api Version:         20121113
    Zend Module Api No:      20121113
    Zend Extension Api No:   220121113

    /tmp/apcu-4.0.2> ./configure --with-php-config=$HOME/myphp/bin/php-config
    /tmp/apcu-4.0.2> make -jN && make install

You should always specify the ``--with-php-config`` option when building extensions (unless you have only a single,
global installation of PHP), otherwise ``./configure`` will not be able to correctly determine what PHP version and
flags to build against. Specifying the ``php-config`` script also ensures that ``make install`` will move the generated
``.so`` file (which can be found in the ``modules/`` directory) to the right extension directory.

As the ``run-tests.php`` file was also copied during the ``phpize`` stage, you can run the extension tests using
``make test`` (or an explicit call to ``run-tests.php``).

The ``make clean`` target for removing compiled objects is also available and allows you to force a full rebuild of
the extension, should the incremental build fail after a change. Additionally phpize provides a cleaning option via
``phpize --clean``. This will remove all the files imported by ``phpize``, as well as the files generated by the
``./configure`` script.

展示扩展信息
---------------------------------------

The PHP CLI binary provides several options to display information about extensions. You already know ``-m``, which will
list all loaded extensions. You can use it to verify that an extension was loaded correctly::

    ~/myphp/bin> ./php -dextension=apcu -m | grep apcu
    apcu

There are several further switches beginning with ``--r`` that expose Reflection functionality. For example you can use
``--ri`` to display the configuration of an extension::

    ~/myphp/bin> ./php -dextension=apcu --ri apcu
    apcu

    APCu Support => disabled
    Version => 4.0.2
    APCu Debugging => Disabled
    MMAP Support => Enabled
    MMAP File Mask =>
    Serialization Support => broken
    Revision => $Revision: 328290 $
    Build Date => Jan  1 2014 16:40:00

    Directive => Local Value => Master Value
    apc.enabled => On => On
    apc.shm_segments => 1 => 1
    apc.shm_size => 32M => 32M
    apc.entries_hint => 4096 => 4096
    apc.gc_ttl => 3600 => 3600
    apc.ttl => 0 => 0
    # ...

The ``--re`` switch lists all ini settings, constants, functions and classes added by an extension:

.. code-block:: none

    ~/myphp/bin> ./php -dextension=apcu --re apcu
    Extension [ <persistent> extension #27 apcu version 4.0.2 ] {
      - INI {
        Entry [ apc.enabled <SYSTEM> ]
          Current = '1'
        }
        Entry [ apc.shm_segments <SYSTEM> ]
          Current = '1'
        }
        # ...
      }

      - Constants [1] {
        Constant [ boolean APCU_APC_FULL_BC ] { 1 }
      }

      - Functions {
        Function [ <internal:apcu> function apcu_cache_info ] {

          - Parameters [2] {
            Parameter #0 [ <optional> $type ]
            Parameter #1 [ <optional> $limited ]
          }
        }
        # ...
      }
    }

The ``--re`` switch only works for normal extensions, Zend extensions use ``--rz`` instead. You can try this on
opcache::

    ~/myphp/bin> ./php -dzend_extension=opcache --rz "Zend OPcache"
    Zend Extension [ Zend OPcache 7.0.3-dev Copyright (c) 1999-2013 by Zend Technologies <http://www.zend.com/> ]

As you can see, this doesn't display any useful information. The reason is that opcache registers both a normal
extension and a Zend extension, where the former contains all ini settings, constants and functions. So in this
particular case you still need to use ``--re``. Other Zend extensions make their information available via ``--rz``
though.
