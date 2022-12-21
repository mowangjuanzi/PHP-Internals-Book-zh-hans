.. highlight:: bash

.. _building_php:

编译 PHP
============

本章解释了如何以适合扩展开发和核心修改的方式编译 PHP。 我们只涵盖了类 UNIX 系统。如果想要在 Windows 中编译 PHP，应该查看 PHP WIKI
中的 `step-by-step build instructions`__ [#]_。

本章还概述了 PHP 构建系统的工作原理及其使用的工具，但对它的详细描述超出了本书的范围。

.. __: https://wiki.php.net/internals/windows/stepbystepbuild_sdk_2

.. [#] 免责声明：对尝试在 Windows 上编译 PHP 造成的任何不良健康影响，我们概不负责。

为什么不使用包？
---------------------

如果你当前正在使用 PHP，可能会使用 ``sudo apt-get install php`` 之类的命令通过包管理器安装
PHP。在解释实际的编译之前，应该首先了解为什么需要自己编译，而不能只使用预编译的。原因有很多：

首先，预编译包只会提供生成的二进制文件，但会缺少编译扩展必需的其它内容，比如：头文件。这可以通过安装开发包（通常称为
``php-dev``）解决。为了便于使用 valgrind 或 gdb 进行调试，可以安装调试符号，这通常另外一个由称为 ``php-dbg``
的包提供这些符号。

但即使安装了头文件和调试符号，你仍然会使用 PHP 的发行版本。这意味着 PHP 将以高优化级别编译，这会使调试变得困难。
另外发行版本也不启用断言，也不生成有关内存泄露的警告。还有，预编译包也不启用线程安全，这可能有助于确保扩展在线程安装配置中构建。

还有一个问题是几乎所有的发行版都对 PHP 应用了额外的补丁。在某些情况下，这些补丁仅包含于配置相关的微小更改，但也有一些发行版使用了 Suhosin
等具有高度浸入的补丁。众所周知，其中一些补丁与低级扩展（如 opcache）有不兼容的内容。

PHP 只会为 `php.net`_ 上提供的软件提供支持，不为分发修改后的版本提供支持。如果想要报告
bug、提交补丁、或者使用我们的帮助渠道编写扩展，应该始终使用官方 PHP 版本。当我们在本书中谈论“PHP”时，始终说的是官方支持的版本。

.. _`php.net`: http://www.php.net

获取源代码
-------------------------

在编译 PHP 之前，首先需要获取源代码。有两种方式：从 `PHP 下载页面`_ 下载归档文件或从 `Github`_ 克隆 git 存储库。

两者的构建过程略有不同：git 存储库不捆绑 ``configure`` 脚本，所以需要使用 ``buildconf`` 脚本生成，该脚本利用了 autoconf。另外，git
存储库不包含预生成的 lexer 和 parser，还需要安装 re2c 和 bison。

建议从 git 检出源代码，因为这将会提供一个简单的方法来保持更新并尝试使用不同版本的代码。如果你香味 PHP 提交补丁或者拉取请求，也需要 git 检出。

在终端中克隆存储库，运行下列命令::

    ~> git clone https://github.com/php/php-src.git
    ~> cd php-src
    # 默认是 master 分支，这是当前
    # 开发版本。可以检出到稳定分支：
    ~/php-src> git checkout PHP-8.1

如果对检出有问题，请查看 PHP wiki 上的 `Git 问答`_。如果想对 PHP 本身做贡献，Git FAQ 还解释了如何设置 git。另外还包含为多个
PHP 版本设置多个工作目录的说明。如果需要针对多个 PHP 版本和配置测试扩展或者更改，这非常有用。

进行下一步之前，应该使用包管理器安装一些基本的编译依赖项（默认已经安装了三个）：

* ``gcc`` 和 ``g++`` 或者一些编译器工具链。
* ``libc-dev`` 提供 C 标准库，包括头文件。
* ``make`` 这是 PHP 使用的编译管理工具。
* ``autoconf`` 用于生成 ``configure`` 脚本。

  * 2.59 或更高（用于 PHP 7.0-7.1）
  * 2.64 或更高（用于 PHP 7.2）
  * 2.68 或更高（用于 PHP 7.3 及其更高）
* ``libtool``，帮助管理共享库。
* ``bison`` 用于生成 PHP 解析器。

  * 2.4 或更高（用于 7.0-7.3）
  * 3.0 或更高（用于 PHP 7.4 及其更高）
* ``re2c``，用于生成 PHP 词法分析器。

  * PHP <= 7.3 时可选
  * 0.13.4 或更高（用于 PHP 7.4 及其更高）

在 Debian/Ubuntu 上，可以使用下列命令安装所有::

    ~/php-src> sudo apt-get install build-essential autoconf libtool bison re2c pkg-config

根据 ``./configure`` 阶段启用的扩展，PHP 将需要很多额外的库。安装时，将检查是否有 ``-dev`` 或 ``-devel``
结尾的软件包版本，然后安装它们。没有 ``dev`` 的包通常不包含必要的头文件。例如，默认的 PHP 编译将需要 libxml 和
libsqlite3，可以通过 ``libxml2-dev`` 和 ``libsqlite3-dev`` 包安装。

.. _PHP 下载页面: http://www.php.net/downloads.php
.. _git.php.net: http://git.php.net
.. _Github: http://www.github.com/php/php-src
.. _Git 问答: https://wiki.php.net/vcs/gitfaq

编译概述
--------------

在仔细研究每个编译步骤的作用之前，这里时需要为“默认”PHP 编译执行的命令::

    ~/php-src> ./buildconf     # 只有从 git 编译时才需要
    ~/php-src> ./configure
    ~/php-src> make -jN

为了快速编译，替换 ``N`` 为有效的 CPU 内核数（可以运行 ``nproc`` 来确定）。

默认 PHP 将为 CLI 和 CGI SAPI 编译二进制文件，分别位于 ``sapi/cli/php`` 和
``sapi/cgi/php-cgi``。要检查是否一切顺利，请尝试运行 ``sapi/cli/php -v``。

此外，可以运行 ``sudo make install`` 将 PHP 安装到 ``/usr/local``。可以在配置阶段指定 ``--prefix`` 来更改目标目录::

    ~/php-src> ./configure --prefix=$HOME/myphp
    ~/php-src> make -jN
    ~/php-src> make install

这里的 ``$HOME/myphp`` 时在 ``make install`` 步骤中使用的安装位置。注意，安装 PHP
不是必需的，但想要在扩展开发之外使用 PHP 编译则可能会很方便。

现在睁大双眼看每个编译步骤吧！

``./buildconf`` 脚本
--------------------------

如果从 git 存储库编译，第一件事就是运行 ``./buildconf`` 脚本。该脚本只是调用 ``build/build.mk`` makefile，又调用 ``build/build2.mk``。

这些 makefile 的主要工作是运行 ``autoconf`` 生成 ``./configure`` 脚本和 ``autoheader`` 生成 ``main/php_config.h.in``
模板。后面的文件会被配置生成最终的配置头文件 ``main/php_config.h``。

Both utilities produce their results from the ``configure.ac`` file (which specifies most of the PHP build process),
the ``build/php.m4`` file (which specifies a large number of PHP-specific M4 macros) and the ``config.m4`` files of
individual extensions and SAPIs (as well as a bunch of other `m4 files <http://www.gnu.org/software/m4/m4.html>`_).

好消息是编写扩展甚至进行核心修改都不需要跟编译系统进行太多交互。稍后需要编写小的 ``config.m4`` 文件，但这些文件通常只使用
``build/php.m4`` 提供的两到三个高级宏。因此，不会再这里进一步详细介绍。

``./buildconf`` 脚本只有两个选项：``--debug`` 将在调用 autoconf 和 autoheader 时禁用警告抑制。除非在编译系统上工作，否则不会对这个选项感兴趣。

第二个选项是 ``--force``，允许在发行包中运行 ``./buildconf``（例如，下载了打包的源代码并想生成新的
``./configure``）并另外清除配置缓存 ``config.cache`` 和 ``autom4te.cache/``。

如果使用 ``git pull`` （或其它命令）更新 git 存储库在 ``make`` 阶段出现奇怪的错误，这通常意味着编译配置中的某些内容发生了变化，需要重新运行
``./buildconf``。

``./configure`` 脚本
--------------------------

一旦生成了 ``./configure`` ，就可以使用它来自定义 PHP 编译。可以使用 ``--help`` 列出所有支持的选项::

    ~/php-src> ./configure --help | less

帮助的第一部分将列出各种通用选项，所有基于 autoconf 的配置脚本都支持这些选项。其中一个是已经提到的 ``--prefix=DIR``，用于更改
``make install`` 的安装目录。另外一个有用的选项是 ``-C``，会将各种测试的结果缓存到 ``config.cache`` 文件中，并加快后续的
``./configure`` 调用。只有当存在可用的编译并且想要在不同配置间快速切换时，此选项才有意义。

除了通用的 autoconf 选项之外，还有很多特定于 PHP 的选项。例如，使用 ``--enable-NAME`` 和 ``--disable-NAME`` 开关选择编译哪些扩展和
SAPI。如果扩展或 SAPI 具有外部依赖项，则需要使用 ``--with-NAME`` and ``--without-NAME`` 代替。

如果 ``NAME`` 需要的库不在默认位置（例如自己编译的），一些扩展允许使用 ``--with-NAME=DIR`` 指定它的位置。但是，由于 PHP 7.4
大多数扩展改用 ``pkg-config``，在这种情况下将目录传递给 ``--with`` 没有作用。这时候，需要将库添加到 ``PKG_CONFIG_PATH``::

    export PKG_CONFIG_PATH=/path/to/library/lib/pkgconfig:$PKG_CONFIG_PATH

PHP 默认将编译 CLI 和 CGI SAPI，以及一些扩展。可以使用 ``-m`` 选项找出 PHP 二进制文件包含哪些扩展。对于默认编译的 PHP 7.0，结果将如下所示：

.. code-block:: none

    ~/php-src> sapi/cli/php -m
    [PHP Modules]
    Core
    ctype
    date
    dom
    fileinfo
    filter
    hash
    iconv
    json
    libxml
    pcre
    PDO
    pdo_sqlite
    Phar
    posix
    Reflection
    session
    SimpleXML
    SPL
    sqlite3
    standard
    tokenizer
    xml
    xmlreader
    xmlwriter

如果想现在停止编译 CGI SAPI 以及 *tokenizer* 和 *sqlite3* 扩展，而是启用 *opcache* 和 *gmp*，相应的配置命令是::

    ~/php-src> ./configure --disable-cgi --disable-tokenizer --without-sqlite3 \
                           --enable-opcache --with-gmp

默认大部分扩展将被静态编译，即它们将成为生成二进制文件的一部分。默认仅共享 opcache
扩展，即将在 ``modules/`` 目录中生成 ``opcache.so`` 共享对象。可以通过编写 ``--enable-NAME=shared`` 或 ``--with-NAME=shared``
将其它扩展编译为共享对象（并非所有扩展都支持）。下一节会讨论如何使用共享扩展。

要了解需要使用哪种开关以及默认情况下是否启用扩展，请查看 ``./configure --help``。如果开关是 ``--enable-NAME`` 或 ``--with-NAME``
意味着默认不编译扩展，需要手动启用。``--disable-NAME`` 或 ``--without-NAME`` 另一方面表示扩展默认编译，但可以手动禁用。

一些扩展始终编译且不能禁用。使用 ``--disable-all`` 选项创建仅包含最少量扩展的编译::

    ~/php-src> ./configure --disable-all && make -jN
    ~/php-src> sapi/cli/php -m
    [PHP Modules]
    Core
    date
    hash
    json
    pcre
    Reflection
    SPL
    standard

``--disable-all`` 选项对想要快速编译且不需要太多功能时非常有用（例如，实现语言更改）。对于尽可能下的编译，可以额外指定 ``--disable-cgi``
开关，只生成 CLI 二进制文件。

There are three more switches, which you should usually specify when developing extensions or working on PHP:

``--enable-debug`` enables debug mode, which has multiple effects: Compilation will run with ``-g`` to generate debug
symbols and additionally use the lowest optimization level ``-O0``. This will make PHP a lot slower, but make debugging
with tools like ``gdb`` more predictable. Furthermore debug mode defines the ``ZEND_DEBUG`` macro, which will enable
the use of assertions and enable various debugging helpers in the engine. Among other things memory leaks, as well as
incorrect use of some data structures, will be reported. It is possible to enable debug assertions without disabling
optimizations by using ``--enable-debug-assertions`` instead.

``--enable-zts`` (or ``--enable-maintainer-zts`` before PHP 8.0) enables thread-safety. This switch will define the
``ZTS`` macro, which in turn will enable the whole TSRM (thread-safe resource manager) machinery used by PHP. Since
PHP 7 having this switch continuously enabled is much less important than on previous versions. It is primarily
important to make sure you included all the necessary boilerplate code. If you need more information about thread
safety and global memory management in PHP, you should read :doc:`the globals management chapter <../extensions_design/globals_management>`

``--enable-werror`` (since PHP 7.4) enables the ``-Werror`` compiler flag, which will promote compiler warnings to
errors. Enabling this flag ensures that the PHP build remains warning free. However, generated warnings depend on the
used compiler, version and optimization options, so some compilers may not be usable with option.

On the other hand you should not use the ``--enable-debug`` option if you want to perform performance benchmarks for
your code. ``--enable-zts`` can also negatively impact runtime performance.

Note that ``--enable-debug`` and ``--enable-zts`` change the ABI of the PHP binary, e.g. by adding additional arguments
to functions. As such, shared extensions compiled in debug mode will not be compatible with a PHP binary built in
release mode. Similarly a thread-safe extension (ZTS) is not compatible with a non-thread-safe PHP build (NTS).

Due to the ABI incompatibility ``make install`` (and PECL install) will put shared extensions in different directories
depending on these options:

* ``$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO`` for release builds without ZTS
* ``$PREFIX/lib/php/extensions/debug-non-zts-API_NO`` for debug builds without ZTS
* ``$PREFIX/lib/php/extensions/no-debug-zts-API_NO`` for release builds with ZTS
* ``$PREFIX/lib/php/extensions/debug-zts-API_NO`` for debug builds with ZTS

The ``API_NO`` placeholder above refers to the ``ZEND_MODULE_API_NO`` and is just a date like ``20100525``, which is
used for internal API versioning.

For most purposes the configuration switches described above should be sufficient, but of course ``./configure``
provides many more options, which you'll find described in the help.

Apart from passing options to configure, you can also specify a number of environment variables. Some of the more
important ones are documented at the end of the configure help output (``./configure --help | tail -25``).

For example you can use ``CC`` to use a different compiler and ``CFLAGS`` to change the used compilation flags::

    ~/php-src> ./configure --disable-all CC=clang CFLAGS="-O3 -march=native"

In this configuration the build will make use of clang (instead of gcc) and use a very high optimization level
(``-O3 -march=native``).

An option that is particularly useful for development is ``-fsanitize``, which allows you to detect memory corruption
and undefined behavior at runtime::

    CFLAGS="-fsanitize=address -fsanitize=undefined"

These options only work reliably since PHP 7.4 and will significantly slow down the generated PHP binary.

``make`` 和 ``make install``
-----------------------------

After everything is configured, you can use ``make`` to perform the actual compilation::

    ~/php-src> make -jN    # where N is the number of cores

The main result of this operation will be PHP binaries for the enabled SAPIs (by default ``sapi/cli/php`` and
``sapi/cgi/php-cgi``), as well as shared extensions in the ``modules/`` directory.

Now you can run ``make install`` to install PHP into ``/usr/local`` (default) or whatever directory you specified using
the ``--prefix`` configure switch.

``make install`` will do little more than copy a number of files to the new location. If you specified ``--with-pear``
during configuration, it will also download and install PEAR. Here is the resulting tree of a default PHP build:

.. code-block:: none

    > tree -L 3 -F ~/myphp

    /home/myuser/myphp
    |-- bin
    |   |-- pear*
    |   |-- peardev*
    |   |-- pecl*
    |   |-- phar -> /home/myuser/myphp/bin/phar.phar*
    |   |-- phar.phar*
    |   |-- php*
    |   |-- php-cgi*
    |   |-- php-config*
    |   `-- phpize*
    |-- etc
    |   `-- pear.conf
    |-- include
    |   `-- php
    |       |-- ext/
    |       |-- include/
    |       |-- main/
    |       |-- sapi/
    |       |-- TSRM/
    |       `-- Zend/
    |-- lib
    |   `-- php
    |       |-- Archive/
    |       |-- build/
    |       |-- Console/
    |       |-- data/
    |       |-- doc/
    |       |-- OS/
    |       |-- PEAR/
    |       |-- PEAR5.php
    |       |-- pearcmd.php
    |       |-- PEAR.php
    |       |-- peclcmd.php
    |       |-- Structures/
    |       |-- System.php
    |       |-- test/
    |       `-- XML/
    `-- php
        `-- man
            `-- man1/

A short overview of the directory structure:

* *bin/* contains the SAPI binaries (``php`` and ``php-cgi``), as well as the ``phpize`` and ``php-config`` scripts.
  It is also home to the various PEAR/PECL scripts.
* *etc/* contains configuration. Note that the default *php.ini* directory is **not** here.
* *include/php* contains header files, which are needed to build additional extensions or embed PHP in custom software.
* *lib/php* contains PEAR files. The *lib/php/build* directory includes files necessary for building extensions, e.g.
  the ``php.m4`` file containing PHP's M4 macros. If we had compiled any shared extensions those files would live
  in a subdirectory of *lib/php/extensions*.
* *php/man* obviously contains man pages for the ``php`` command.

As already mentioned, the default *php.ini* location is not *etc/*. You can display the location using the ``--ini``
option of the PHP binary:

.. code-block:: none

    ~/myphp/bin> ./php --ini
    Configuration File (php.ini) Path: /home/myuser/myphp/lib
    Loaded Configuration File:         (none)
    Scan for additional .ini files in: (none)
    Additional .ini files parsed:      (none)

As you can see the default *php.ini* directory is ``$PREFIX/lib`` (libdir) rather than ``$PREFIX/etc`` (sysconfdir). You
can adjust the default *php.ini* location using the ``--with-config-file-path=PATH`` configure option.

Also note that ``make install`` will not create an ini file. If you want to make use of a *php.ini* file it is your
responsibility to create one. For example you could copy the default development configuration:

.. code-block:: none

    ~/myphp/bin> cp ~/php-src/php.ini-development ~/myphp/lib/php.ini
    ~/myphp/bin> ./php --ini
    Configuration File (php.ini) Path: /home/myuser/myphp/lib
    Loaded Configuration File:         /home/myuser/myphp/lib/php.ini
    Scan for additional .ini files in: (none)
    Additional .ini files parsed:      (none)

Apart from the PHP binaries the *bin/* directory also contains two important scripts: ``phpize`` and ``php-config``.

``phpize`` is the equivalent of ``./buildconf`` for extensions. It will copy various files from *lib/php/build* and
invoke autoconf/autoheader. You will learn more about this tool in the next section.

``php-config`` provides information about the configuration of the PHP build. Try it out:

.. code-block:: none

    ~/myphp/bin> ./php-config
    Usage: ./php-config [OPTION]
    Options:
      --prefix            [/home/myuser/myphp]
      --includes          [-I/home/myuser/myphp/include/php -I/home/myuser/myphp/include/php/main -I/home/myuser/myphp/include/php/TSRM -I/home/myuser/myphp/include/php/Zend -I/home/myuser/myphp/include/php/ext -I/home/myuser/myphp/include/php/ext/date/lib]
      --ldflags           [ -L/usr/lib/i386-linux-gnu]
      --libs              [-lcrypt   -lresolv -lcrypt -lrt -lrt -lm -ldl -lnsl  -lxml2 -lxml2 -lxml2 -lcrypt -lxml2 -lxml2 -lxml2 -lcrypt ]
      --extension-dir     [/home/myuser/myphp/lib/php/extensions/debug-zts-20100525]
      --include-dir       [/home/myuser/myphp/include/php]
      --man-dir           [/home/myuser/myphp/php/man]
      --php-binary        [/home/myuser/myphp/bin/php]
      --php-sapis         [ cli cgi]
      --configure-options [--prefix=/home/myuser/myphp --enable-debug --enable-maintainer-zts]
      --version           [5.4.16-dev]
      --vernum            [50416]

The script is similar to the ``pkg-config`` script used by linux distributions. It is invoked during the extension
build process to obtain information about compiler options and paths. You can also use it to quickly get information
about your build, e.g. your configure options or the default extension directory. This information is also provided by
``./php -i`` (phpinfo), but ``php-config`` provides it in a simpler form (which can be easily used by automated tools).

运行测试套件
----------------------

If the ``make`` command finishes successfully, it will print a message encouraging you to run ``make test``:

.. code-block:: none

    Build complete.
    Don't forget to run 'make test'

``make test`` will run the PHP CLI binary against our test suite, which is located in the different *tests/* directories
of the PHP source tree. As a default build is run against more than 10000 (less for a minimal build, more if
you enable additional extensions) this can take several minutes.

The ``make test`` command internally invokes the ``run-tests.php`` file using your CLI binary. For more control, it is
recommended to invoke ``run-tests.php`` directly. For example, this will allow you to enable the parallel test runner::

    ~/php-src> sapi/cli/php run-tests.php -jN

Test parallelism is only available as of PHP 7.4. On earlier PHP versions parallelism is not available, and it is
necessary to additionally pass the ``-P`` option::

    ~/php-src> sapi/cli/php run-tests.php -P

Instead of running the whole test suite, you can also limit it to certain directories by passing them as arguments to
``run-tests.php``. E.g. to test only the Zend engine, the reflection extension and the array functions::

    ~/php-src> sapi/cli/php run-tests.php -jN Zend/ ext/reflection/ ext/standard/tests/array/

This is very useful, because it allows you to quickly run only the parts of the test suite that are relevant to your
changes. E.g. if you are doing language modifications you likely don't care about the extension tests and only want to
verify that the Zend engine is still working correctly.

You can run ``sapi/cli/php run-tests.php --help`` to display a full list of options the test runner accepts. Some
particularly useful options are:

  * ``-c php.ini`` can be used to specify a php.ini file to use.
  * ``-d foo=bar`` can be used to set ini options.
  * ``-m`` runs tests under valgrind to detect memory errors. Note that this is extremely slow.
  * ``--asan`` should be set when compiling PHP with ``-fsanitize=address``. Together these are approximately
    equivalent to running under valgrind, but with much better performance.

You don't need to explicitly use ``run-tests.php`` to pass options or limit directories. Instead you can use the
``TESTS`` variable to pass additional arguments via ``make test``. E.g. the equivalent of the previous command would
be::

    ~/php-src> make test TESTS="-jN Zend/ ext/reflection/ ext/standard/tests/array/"

We will take a more detailed look at the ``run-tests.php`` system later, in particular also talk about how to write your
own tests and how to debug test failures. :doc:`See the dedicated tests chapter <../../tests/introduction>`.

修复编译问题和 ``make clean``
----------------------------------------------

As you may know ``make`` performs an incremental build, i.e. it will not recompile all files, but only those ``.c``
files that changed since the last invocation. This is a great way to shorten build times, but it doesn't always work
well: For example, if you modify a structure in a header file, ``make`` will not automatically recompile all ``.c``
files making use of that header, thus leading to a broken build.

If you get odd errors while running ``make`` or the resulting binary is broken (e.g. if ``make test`` crashes it before
it gets to run the first test), you should try to run ``make clean``. This will delete all compiled objects, thus
forcing the next ``make`` call to perform a full build. (You can use ``ccache`` to reduce the cost of rebuilds.)

Sometimes you also need to run ``make clean`` after changing ``./configure`` options. If you only enable additional
extensions an incremental build should be safe, but changing other options may require a full rebuild.

Another source of compilation issues is the modification of ``config.m4`` files or other files that are part of the PHP
build system. If such a file is changed, it is necessary to rerun the ``./buildconf`` and ``./configure`` scripts. If
you do the modification yourself, you will likely remember to run the command, but if it happens as part of a
``git pull`` (or some other updating command) the issue might not be so obvious.

If you encounter any odd compilation problems that are not resolved by ``make clean``, chances are that running
``./buildconf`` will fix the issue. To avoid typing out the previous ``./configure`` options afterwards, you can make
use of the ``./config.nice`` script (which contains your last ``./configure`` call)::

    ~/php-src> make clean
    ~/php-src> ./buildconf --force
    ~/php-src> ./config.nice
    ~/php-src> make -jN

One last cleaning script that PHP provides is ``./vcsclean``. This will only work if you checked out the source code
from git. It effectively boils down to a call to ``git clean -X -f -d``, which will remove all untracked files and
directories that are ignored by git. You should use this with care.
