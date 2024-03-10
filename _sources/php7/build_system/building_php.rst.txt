.. highlight:: bash

.. _building_php:

编译 PHP
============

本章解释了如何以适合扩展开发和核心修改的方式编译 PHP。 我们只涵盖了类 UNIX 系统。如果想要在 Windows 中编译 PHP，应该查看 PHP WIKI
中的 `step-by-step build instructions`__ [#]_。

本章还概述了 PHP 编译系统的工作原理及其使用的工具，但对它的详细描述超出了本书的范围。

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
另外发行版本也不启用断言，也不生成有关内存泄露的警告。还有，预编译包也不启用线程安全，这可能有助于确保扩展在线程安装配置中编译。

还有一个问题是几乎所有的发行版都对 PHP 应用了额外的补丁。在某些情况下，这些补丁仅包含于配置相关的微小更改，但也有一些发行版使用了 Suhosin
等具有高度浸入的补丁。众所周知，其中一些补丁与低级扩展（如 opcache）有不兼容的内容。

PHP 只会为 `php.net`_ 上提供的软件提供支持，不为分发修改后的版本提供支持。如果想要报告
bug、提交补丁、或者使用我们的帮助渠道编写扩展，应该始终使用官方 PHP 版本。当我们在本书中谈论“PHP”时，始终说的是官方支持的版本。

.. _`php.net`: http://www.php.net

获取源代码
-------------------------

在编译 PHP 之前，首先需要获取源代码。有两种方式：从 `PHP 下载页面`_ 下载归档文件或从 `Github`_ 克隆 git 存储库。

两者编译过程略有不同：git 存储库不捆绑 ``configure`` 脚本，所以需要使用 ``buildconf`` 脚本生成，该脚本利用了 autoconf。另外，git
存储库不包含预生成的 lexer 和 parser，还需要安装 re2c 和 bison。

建议从 git 检出源代码，因为这将会提供一个简单的方法来保持更新并尝试使用不同版本的代码。如果想为 PHP 提交补丁或者拉取请求，也需要 git 检出。

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

这两个实用程序都从 ``configure.ac`` 文件（该文件指定了 PHP 大部分编译过程）、``build/php.m4`` 文件（该文件指定了大量 PHP 特定 M4
宏）以及各个扩展和 SAPI 的 ``config.m4`` 文件中（以及其他许多 `m4文件 <http://www.gnu.org/software/m4/m4.html>`_）生成结果。

好消息是编写扩展甚至进行核心修改都不需要跟编译系统进行太多交互。稍后需要编写小的 ``config.m4`` 文件，但这些文件通常只使用
``build/php.m4`` 提供的两到三个高级宏。因此，不会再这里进一步详细介绍。

``./buildconf`` 脚本只有两个选项：``--debug`` 将在调用 autoconf 和 autoheader 时禁用警告抑制。除非在编译系统上工作，否则不会对这个选项感兴趣。

第二个选项是 ``--force``，允许在发行包中运行 ``./buildconf`` （例如，下载了打包的源代码并想生成新的
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

还有另外三个开关，通常应该在开发扩展或使用 PHP 时指定它们：

``--enable-debug`` 启用调试模式，有多种效果：运行编译时将使用 ``-g`` 生成调试符号，并额外使用最低优化级别 ``-O0``。这会使 PHP
变慢很多，但可以使使用 ``gdb`` 等工具进行调试更容易预测。此外，调试模式定义了 ``ZEND_DEBUG`` 宏，它将启用断言并在引擎中启用各种调试助手。除了报告内存泄漏以外，还会报告某些没有正确使用的数据结构。通过使用
``--enable-debug-assertions`` 可以在不禁用优化的情况下启用调试断言。

``--enable-zts`` （PHP 8.0 之前为 ``--enable-maintainer-zts``）启用线程安全。这个开关将定义 ``ZTS`` 宏，它反过来将使 PHP 启用完整
TSRM（线程安全资源管理器）机制。自 PHP 7 起，持续启用此开关的重要性要比之前的版本低得多。确保包含所有必要的样板代码非常重要。如果需要有关
PHP 的线程安全和全局内存管理的更多信息，应该阅读 :doc:`全局管理章节 <../extensions_design/globals_management>`

``--enable-werror`` （自 PHP 7.4 起）启用 ``-Werror`` 编译器 flag，这会使编译器警告升级为错误。启用此 flag 可确保 PHP
编译时保持无警告。但是，生成的警告取决于使用的编译器、版本和优化选项，因此某些编译器可能无法使用此选项。

另一方面，如果要对代码执行性能测试，则不应使用 ``--enable-debug`` 选项。``--enable-zts`` 也会对运行时的性能产生负面影响。

注意，``--enable-debug`` 和 ``--enable-zts`` 会更改 PHP 二进制文件的 ABI，例如，向函数添加其他参数。因此，以调试模式编译的共享扩展将与发布模式编译的
PHP 二进制文件不兼容。同样，线程安全的扩展 (ZTS) 与编译后的非线程安全 PHP (NTS) 不兼容。

由于 ABI 不兼容，根据这些选项，``make install`` （和 PECL install）会将共享扩展放在不同的目录中:

* 不带 ZTS 的发行版本： ``$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO``
* 不带 ZTS 的调试版本： ``$PREFIX/lib/php/extensions/debug-non-zts-API_NO``
* 带 ZTS 的发行版本： ``$PREFIX/lib/php/extensions/no-debug-zts-API_NO``
* 带 ZTS 的调试版本： ``$PREFIX/lib/php/extensions/debug-zts-API_NO``

上面的 ``API_NO`` 占位符指的是 ``ZEND_MODULE_API_NO``，只是一个像 ``20100525`` 这样的日期，用于内部 API 版本控制。

对于大多数用途来说，上述配置开关应该足够了，但当然 ``./configure`` 提供了更多选项，可以在帮助中找到这些选项。

除了选项传递给 configure 之外，还可以指定许多环境变量。一些更重要的内容记录在 configure 帮助输出的末尾 (``./configure --help | tail -25``)。

例如，可以使用 ``CC`` 来使用不同的编译器，并使用 ``CFLAGS`` 来更改使用的编译flags： ::

    ~/php-src> ./configure --disable-all CC=clang CFLAGS="-O3 -march=native"

在此配置中，编译将使用 clang（而不是 gcc）并使用非常高的优化级别 (``-O3 -march=native``)。

对于开发特别有用的选项是 ``-fsanitize``，它允许在运行时检测内存损坏和未定义的行为：::

    CFLAGS="-fsanitize=address -fsanitize=undefined"

这些选项仅自 PHP 7.4 起能狗稳定运行，并且会显着降低生成 PHP 二进制文件的速度。

``make`` 和 ``make install``
-----------------------------

配置完所有后，可以使用 ``make`` 来执行实际编译：::

    ~/php-src> make -jN    #  N 是核心数

此操作的主要结果将是启用 SAPI（默认为 ``sapi/cli/php`` 和 ``sapi/cgi/php-cgi``）的 PHP 二进制文件，以及 ``modules/`` 目录中的共享扩展。

现在，可以运行 ``make install`` 将 PHP 安装到 ``/usr/local``（默认）或使用 ``--prefix`` 配置项指定的目录。

``make install`` 只会将一些文件复制到新位置。如果在配置过程中指定了 ``--with-pear``，也将会下载并安装 PEAR。以下是默认 PHP 编译的结果树：

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

目录结构的简短概述：

* *bin/* 包含 SAPI 二进制文件（ ``php`` 和 ``php-cgi``），以及 ``phpize`` 和 ``php-config`` 脚本。也是各种 PEAR/PECL 脚本的所在地。
* *etc/* 包含配置。注意默认 *php.ini* 目录 **不** 在这里。
* *include/php* 包含头文件，用于需要编译附加的扩展或者在自定义软件中内嵌 PHP。
* *lib/php* 包含 PEAR 文件。*lib/php/build* 目录包含编译扩展所需的文件，比如
  ``php.m4`` 文件包含 PHP M4 宏。如果编译了任何共享扩展，这些文件将位于 *lib/php/extensions* 的子目录中。
* *php/man* 显而易见包含 ``php`` 命令的手册页。

正如刚才所说，默认 *php.ini* 位置不是 *etc/*。可以使用 PHP 二进制文件的 ``--ini`` 选项显示位置：

.. code-block:: none

    ~/myphp/bin> ./php --ini
    Configuration File (php.ini) Path: /home/myuser/myphp/lib
    Loaded Configuration File:         (none)
    Scan for additional .ini files in: (none)
    Additional .ini files parsed:      (none)

如你所见，默认 *php.ini* 目录是 ``$PREFIX/lib`` (libdir) 而不是 ``$PREFIX/etc`` (sysconfdir)。可以使用
``--with-config-file-path=PATH`` 配置选项调整默认 *php.ini* 位置。

另需要注意， ``make install`` 不会创建 ini 文件。如果想使用 *php.ini* 文件，则需要自己创建文。例如，可以复制默认的开发配置：

.. code-block:: none

    ~/myphp/bin> cp ~/php-src/php.ini-development ~/myphp/lib/php.ini
    ~/myphp/bin> ./php --ini
    Configuration File (php.ini) Path: /home/myuser/myphp/lib
    Loaded Configuration File:         /home/myuser/myphp/lib/php.ini
    Scan for additional .ini files in: (none)
    Additional .ini files parsed:      (none)

*bin/* 目录除了 PHP 二进制文件之外，还包含两个重要的脚本: ``phpize`` 和 ``php-config``。

``phpize`` 等同于扩展的 ``./buildconf``。将从 *lib/php/build* 复制各种文件并调用
autoconf/autoheader。此工具的更多信息将在下一节了解到。

``php-config`` 提供了有关 PHP 编译配置的信息，试试看：

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

该脚本类似于 Linux 发行版使用的 ``pkg-config`` 脚本。会在扩展编译过程中调用它，以获取有关编译器选项和路径的信息。还可以使用它来快速获取有关编译的信息，例如配置选项或默认扩展目录。此信息也由
``./php -i`` (phpinfo) 提供，但 ``php-config`` 以更简单的形式提供（可以轻松用于自动化工具）。

运行测试套件
----------------------

如果 ``make`` 命令成功完成，它将打印一条消息，鼓励运行 ``make test``：

.. code-block:: none

    Build complete.
    Don't forget to run 'make test'

``make test`` 命令在内部使用 CLI 二进制文件调用 ``run-tests.php`` 文件。为了进行更多控制，建议直接调用 ``run-tests.php``。例如，这将允许启用并行测试运行程序::

    ~/php-src> sapi/cli/php run-tests.php -jN

并行性测试仅自 PHP 7.4 起可用。在早期 PHP 版本中，并行不可用，并且需要另外传递 ``-P`` 选项::

    ~/php-src> sapi/cli/php run-tests.php -P

除了运行整个测试套件之外，还可以将某些目录作为参数传递给 ``run-tests.php``。例如，仅测试 Zend 引擎、反射扩展和数组函数::

    ~/php-src> sapi/cli/php run-tests.php -jN Zend/ ext/reflection/ ext/standard/tests/array/

这非常有用，因为它允许快速仅运行与更改相关的测试套件部分。例如，如果正在进行语言修改，可能不关心扩展测试，而只想验证 Zend 引擎是否仍然正常工作。

可以运行 ``sapi/cli/php run-tests.php --help`` 以显示测试运行程序接受选项的完整列表。一些特别有用的选项是：

  * ``-c php.ini`` 可以用于指定要使用的 php.ini 文件。
  * ``-d foo=bar`` 可以用于设置 ini 选项。
  * ``-m`` 在 valgrind 下运行测试以检测内存错误。注意，这非常慢。
  * ``--asan`` 用于使用 ``-fsanitize=address`` 编译 PHP 时应设置。它们组合在一起大约相当于在 valgrind 下运行，但性能要好很多。

不需要手动使用 ``run-tests.php`` 来传递选项或限制目录。而是可以使用 ``TESTS`` 变量通过 ``make test`` 传递附加参数。例如，与上一个命令等效的是::

    ~/php-src> make test TESTS="-jN Zend/ ext/reflection/ ext/standard/tests/array/"

稍后将更详细地了解 ``run-tests.php`` 系统，特别是如何编写自己的测试以及如何调试测试失败。:doc:`请参阅专门的测试章节<../../tests/introduction>`。

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
