:mod:`pwnlib.dynelf` --- 使用leak解析远程函数
===============================================================

这个模块主要用于解析ELF文件中加载的以及动态链接的符号。只要给一个可以泄漏任
何地址内容的函数，我们就可以解析加载的库中的任何地址。

例子

.. code-block:: python

   # Assume a process or remote connection
   p = process('./pwnme')

   # Declare a function that takes a single address, and
   # leaks at least one byte at that address.
   def leak(address):
       data = p.read(address, 4)
       log.debug("%#x => %s" % (address, (data or '').encode('hex')))
       return data

   # For the sake of this example, let's say that we
   # have any of these pointers.  One is a pointer into
   # the target binary, the other two are pointers into libc
   main   = 0xfeedf4ce
   libc   = 0xdeadb000
   system = 0xdeadbeef

   # With our leaker, and a pointer into our target binary,
   # we can resolve the address of anything.
   #
   # We do not actually need to have a copy of the target
   # binary for this to work.
   d = DynELF(leak, main)
   assert d.lookup(None,     'libc') == libc
   assert d.lookup('system', 'libc') == system

   # However, if we *do* have a copy of the target binary,
   # we can speed up some of the steps.
   d = DynELF(leak, main, elf=ELF('./pwnme'))
   assert d.lookup(None,     'libc') == libc
   assert d.lookup('system', 'libc') == system

   # Alternately, we can resolve symbols inside another library,
   # given a pointer into it.
   d = DynELF(leak, libc + 0x1234)
   assert d.lookup('system')      == system


DynELF

``class pwnlib.dynelf.DynELF(leak,pointer,elf=None)``

Dynelf 知道如何借助 ``pwnlib.memleak.MemLeak``. 通过 infoleak 或者 memleak 漏洞解
析远程程序中的符号地址。

应用细节

    解析函数：

        ELF文件（例如 ``libc.so`` ）一般会导出自己声明的符号，以便于其他库使用。这样，就会存在一系列的表格，它们会给出导出符号的名字，地址，以及这些导出符号的 ``hash`` 值。通过将hash函数应用在期望符号上(例如 ``printf`` )，我们就可以在hash表中找到对应的位置。其相应的索引提供了到string name table( `strtab <https://refspecs.linuxbase.org/elf/gabi4+/ch4.strtab.html>`_ )和symbol address( `symtab <https://refspecs.linuxbase.org/elf/gabi4+/ch4.symtab.html>`_ ) 的寻找的方式。

        假设我们有 ``libc.so`` 的基地址，解析 ``printf`` 的地址的方式就是去定位 ``symtab``, ``strtab``, ``hashtabe``。字符串 ``"printf"`` 根据 hash 表的风格( `YSV <https://refspecs.linuxbase.org/elf/gabi4+/ch5.dynamic.html#hash>`_ 或者 `GNU <https://blogs.oracle.com/ali/entry/gnu_hash_elf_sections>`_ )被hash，因此我们可以遍历hashtable来找到一个匹配的入口。我们可以通过检查 strtab 来验证一个匹配，然后从 symtab 中获取其在 ``libc.so`` 中的偏移。

    解析库地址：

        如果我们有一个指向动态连接可执行文件的指针，我们可以得到一个内部名叫 `link map <https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/link.h;h=eaca8028e45a859ac280301a6e955a14eed1b887;hb=HEAD#l84>`_ 的指针结构。这是一个链表，其中包含了每一个被加载的库，以及它的完整路径和基地址。

          - 我们可以有两种方法来寻找指向 ``link map`` 的指针。两者都参考了 `DYNAMIC <http://www.sco.com/developers/gabi/latest/ch5.dynamic.html#dynamic_section>`_ 数组中的入口。在非RELRO二进制文件中，指针被放在二进制文件的 `.got.plt <https://refspecs.linuxbase.org/LSB_3.1.1/LSB-Core-generic/LSB-Core-generic/specialsections.html>`_ 处。这通过在二进制文件里寻找 `DT_PLTGOT <http://refspecs.linuxfoundation.org/ELF/zSeries/lzsabi0_zSeries/x2251.html>`_ 来标记。
          - 在所有的二进制文件里，指针可以在 `DT_DEBUG <https://reverseengineering.stackexchange.com/questions/6525/elf-link-map-when-linked-as-relro>`_ 区域中找到。这个区域在stripped 的二进制文件里仍然存在。

        为了充分的灵活性，这两种机制都被充分地使用了。

   :func:`__init__(self, leak, pointer=None, elf=None)`

   这是DynELF的一个实例化方法，通过它，如果我们给了 :class:`pwnlib.memleak.MemLeak` leaker,
   以及一个binary文件中的指针，我们可以生成一个可以解析运行库中符号的实例对象。
   
   参数：
       - **leak** (MemLeak) - 泄漏内存的 pwnlib.memleak.MemLeak 的实例。
       - **pointer** (int)  - 已经被load的ELF文件中的一个指针。
       - **elf** (str,ELF)  - ELF文件对应的路径，或者说一个已经加载的 :class:`pwnlib.elf.ELF` 。

   :func:`bases()`

   解析所有已经被加载的二进制文件的基地址。

   返回一个将库路径映射到它的基地址的字典。

   :func:`static find_base(leak,ptr)`

   给定一个 :class:`pwnlib.memleak.Memleak` 对象，以及一个其二进制文件内部的binary，找到它的基地址。

   :func:`heap()`

   通过 __curbrk 来查找当前堆的基地址，这个符号在链接器中已经被导出了，指向当前的brk。

   :func:`lookup(symb=None,lib=None)-->int`

   在lib中找到对应符号的地址。参数如下

      - **symb** (str) - 要查找的名字。
      - **lib** (str)  - 用来匹配库文件名的子串。缺省情况下会搜索当前的库，如果设置为 ``libc`` , ``libc.so``就会被查找。

   返回对应符号的地址，或者None。

   :func:`stack()`  

   通过 __envrion （这个符号是由libc导出的，指向环境变量部分）来寻找一个指向栈上的变量，

   :func:`dynamic`

   返回指向 ``.DYNAMIC`` 的指针。

   :func:`elfclass(self)` 

   返回对应的二进制文件的类型。

   :func:`libc(self)`

   得到远程libc.so的build版本，下载对应的文件，并且使用正确的基地址加载 ``ELF`` 对象。

   返回一个ELF对象，或者什么也不返回。

   :func:`link_map(self)`

   返回指向运行时的link_map对象。

``pwnlib.dynelf.gnu_hash(str)-->int``

为字符串生成GNU格式的hash值。

``pwnlib.dynelf.sysv_hash(str)-->int``

为字符串生成SYSV格式的hash值。
