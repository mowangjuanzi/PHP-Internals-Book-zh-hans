序言
============

本书由几位 PHP 开发者协同合作，旨在更好的记录和描述 PHP 如何内部工作。

本书有三个主要目标：

 * 记录和描述 PHP 如何内部工作。
 * 记录和描述如何使用扩展（extension）来扩展语言。
 * 记录和描述如何跟 PHP 社区互动以便开发 PHP 本身。

本书主要面向具有 C 编程经验的开发人员。但我们会尽可能的提炼和总结信息，以便不了解 C 的开发人员能够理解相关内容。

然而，我们坚信。如果你不了解 C 语言，你将无法实现高效、稳定（在任何平台都不会崩溃）、高性能和有用的东西。以下是关于 C
语言本身、生态系统、构建工具、以及操作系统 API 等一些非常不错的在线资源。

* http://www.tenouk.com/
* https://en.wikibooks.org/wiki/C_Programming
* http://c-faq.com/
* https://www.gnu.org/software/libc/
* http://www.faqs.org/docs/Linux-HOWTO/Program-Library-HOWTO.html

我们也强烈推荐你一些书。你将会学习如何高效使用 C 语言以及如何将其转换为高效的 CPU 指令，以便设计出强大、快速、可靠和安全的程序。

* C 程序设计语言（Ritchie & Kernighan）
* Advanced Topics in C Core Concepts in Data Structures
* 笨办法学 C 语言（Learn C the Hard Way）
* 软件调试的艺术（The Art of Debugging with GDB DDD and Eclipse）
* Linux/UNIX 系统编程手册（The Linux Programming Interface）
* Advanced Linux Programming
* 算法心得（Hackers Delight）
* 编程卓越之道第2卷：运用底层语言思想编写高级语言代码（Write Great Code (2 Volumes)）

.. note:: 本书是半成品，一些章节还没写完。我们不会注意特定的顺序，而是根据我们的感觉添加内容。

本书的存储库位于 GitHub_ / `中文版本 <https://github.com/mowangjuanzi/PHP-Internals-Book-zh-hans>`_。请在 `issue 追踪器`_ 上报告问题并提供反馈。

.. _GitHub: https://github.com/phpinternalsbook/PHP-Internals-Book
.. _issue 追踪器: https://github.com/phpinternalsbook/PHP-Internals-Book/issues
