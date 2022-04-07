<<<<<<< HEAD
********************************
Tata Letak source files Solidity
********************************

Source files (*kode sumber*) dapat berisi sejumlah arbitrer
:ref:`contract definitions<contract_structure>`, import_ directives,
:ref:`pragma directives<pragma>` and
:ref:`struct<structs>`, :ref:`enum<enums>`, :ref:`function<functions>`, :ref:`error<errors>`
and :ref:`constant variable<constants>` definitions.

.. index:: ! license, spdx

Pengidentifikasi Lisensi SPDX
=======================++++++

Kepercayaan pada smart kontrak dapat dibangun dengan lebih baik jika kode sumbernya tersedia.
Karena menyediakan kode sumber selalu menyentuh masalah hukum yang berkaitan dengan hak cipta,
compiler Solidity mendorong penggunaan `SPDX license identifiers yang dapat dibaca mesin <https://spdx.org>`_.
Setiap kode sumber harus dimulai dengan komentar yang menunjukkan lisensinya:

``// SPDX-License-Identifier: MIT``

Kompiler tidak memvalidasi bahwa lisensi adalah bagian dari
`daftar yang diizinkan oleh SPDX <https://spdx.org/licenses/>`_, tapi
itu menyertakan string yang disediakan dalam :ref:`bytecode metadata <metadata>`.

Jika Anda tidak ingin menentukan lisensi atau jika kode sumbernya
bukan open-source, harap gunakan nilai khusus ``UNLICENSED``.

Memberikan komentar ini tentu saja tidak membebaskan Anda dari kewajiban lain
yang terkait dengan perizinan seperti harus menyebutkan
header lisensi tertentu di setiap file sumber atau
pemegang hak cipta asli.

Komentar dikenali oleh kompiler di mana saja di bagian file,
tetapi disarankan untuk meletakkannya di bagian atas file.

Informasi lebih lanjut tentang cara menggunakan pengidentifikasi lisensi SPDX
dapat ditemukan di `situs web SPDX <https://spdx.org/ids-how>`_.


.. index:: ! pragma

.. _pragma:

Pragma
=======

Kata kunci ``pragma`` digunakan untuk mengaktifkan fitur atau
pemeriksaan kompiler tertentu. Arahan pragma selalu lokal ke
file sumber, jadi Anda harus menambahkan pragma ke semua file
Anda jika Anda ingin mengaktifkannya di seluruh proyek Anda.
Jika Anda :ref:`import<import>` file lain, pragma dari file
tersebut *tidak* secara otomatis diterapkan ke file yang diimpor.

.. index:: ! pragma, version

.. _version_pragma:

Versi pragma
--------------

File sumber dapat (dan harus) dianotasi dengan versi pragma untuk menolak kompilasi dengan versi
kompiler masa depan yang mungkin memperkenalkan perubahan yang tidak kompatibel.
Kami mencoba untuk menjaga ini agar tetap minimum dan memperkenalkannya sedemikian rupa sehingga
perubahan dalam semantics juga memerlukan perubahan dalam syntax, tetapi ini tidak selalu memungkinkan.
karena itu, selalu merupakan ide yang baik untuk membaca changelog setidaknya untuk rilis yang berisi
perubahan yang melanggar. Rilis ini selalu memiliki versi dalam bentuk ``0.x.0`` atau ``x.0.0``.

penggunaan versi pragma adalah sebagai berikut: ``pragma solidity ^0.5.2;``

File sumber dengan baris di atas tidak dikompilasi dengan kompiler dibawah versi 0.5.2,
dan juga tidak bekerja pada compiler mulai dari versi 0.6.0 keatas (kondisi kedua ini
ditambahkan dengan menggunakan  tanda ``^``). Karena tidak akan ada perubahan yang mengganggu
hingga versi ``0.6.0``, Anda dapat yakin bahwa kode Anda dikompilasi dengan cara yang Anda inginkan.
Versi kompiler yang tepat tidak diperbaiki, sehingga rilis perbaikan bug masih dimungkinkan.

Dimungkinkan untuk menentukan aturan yang lebih kompleks untuk versi kompiler,
dengan menggunakan syntax yang sama yang digunakan `npm <https://docs.npmjs.com/cli/v6/using-npm/semver>`_.

.. note::
  Menggunakan versi pragma *tidak* mengubah versi kompilerr.
  juga *tidak* mengaktifkan atau menonaktifkan fitur sebuah kompiler.
  Ini hanya menginstruksikan kompiler untuk memeriksa apakah versinya
  cocok dengan yang dibutuhkan oleh pragma. Jika tidak cocok, akan
  terjadi kesalahan pada kompiler.

ABI Coder Pragma
----------------

Dengan menggunakan ``pragma abicoder v1`` atau ``pragma abicoder v2`` anda dapat
memilih antara dua implementasi ABI encoder dan decoder.

ABI coder terbaru (v2) mampu untuk meng*encode* dan decode nested arrays dan structs semaunya.
Ini mungkin menghasilkan kode yang kurang optimal  dan belum menerima pengujian sebanyak encoder lama,
tetapi dianggap non-eksperimental pada Solidity 0.6.0. Anda masih harus mengaktifkannya secara eksplisit
menggunakan ``pragma abicoder v2;``. Mulai dari Solidity 0.8.0 ini akan diaktifkan secara default,
ada pilihan untuk memilih coder lama dengan menggunakan ``pragma abicoder v1;``.

Kumpulan jenis yang didukung oleh encoder baru adalah strict superset dari yang didukung oleh versi yang lama.
Kontrak yang menggunakannya dapat berinteraksi dengan kontrak yang tidak menggunakannya tanpa batasan.
Kebalikannya hanya dimungkinkan selama kontrak non-``abicoder v2`` tidak mencoba melakukan panggilan yang
memerlukan jenis dekode yang hanya didukung oleh encoder baru. Kompiler dapat mendeteksi dan akan melaporkan kesalahan.
Dengan mengaktifkan ``abicoder v2`` untuk kontrak Anda sudah cukup untuk menghilangkan kesalahan.

.. note::

  Pragma ini berlaku untuk semua kode yang ditentukan dalam file tempat kode tersebut diaktifkan,
  terlepas dari di mana kode itu akan berakhir. Ini berarti bahwa kontrak yang file sumbernya
  dipilih untuk dikompilasi dengan ABI coder v1 masih dapat berisi kode yang menggunakan encoder baru
  dengan mewarisinya dari kontrak lain. Ini diperbolehkan jika tipe baru hanya digunakan secara internal
  dan bukan dalam tanda tangan fungsi eksternal.

.. note::
  Hingga Solidity 0.7.4, dimungkinkan untuk memilih ABI coder v2
  dengan menggunakan ``pragma experimental ABIEncoderV2``, tetapi itu tidak mungkin
  untuk secara eksplisit memilih coder v1 karena itu adalah default.

.. index:: ! pragma, experimental

.. _experimental_pragma:

Experimental Pragma
-------------------

Pragma kedua adalah pragma eksperimental. Ini dapat digunakan untuk mengaktifkan fitur
kompiler atau bahasa yang belum diaktifkan secara default.
Berikut Pragma experimental yang saat ini didukung:


ABIEncoderV2
~~~~~~~~~~~~

Karena ABI coder v2 tidak dianggap eksperimental lagi,
itu dapat dipilih melalui ``pragma abicoder v2`` (silakan lihat di atas)
mulai dari Solidity 0.7.4.

.. _smt_checker:

SMTChecker
~~~~~~~~~~

Komponen ini harus diaktifkan ketika compiler Solidity dibangun dan
oleh karena itu tidak tersedia di semua binari Solidity.
:ref:`build instruction<smt_solvers_build>` menjelaskan cara mengaktifkan opsi ini.
Ini diaktifkan untuk rilis PPA Ubuntu di sebagian besar versi,
tapi tidak untuk image Docker, binari Windows atau binari built statically-built Linux.
Ini dapat diaktifkan via `smtCallback <https://github.com/ethereum/solc-js#example-usage-with-smtsolver-callback>`_
jika anda SMT solver sudah terinstal dan menjalankan solc-js via node (bukan via browser).

Jika Anda menggunakan ``pragma eksperimental SMTTChecker;``, maka Anda mendapatkan tambahan
:ref:`peringatan keamanan<formal_verification>` yang diperoleh dengan menanyakan sebuah
SMT solver.
Komponen belum mendukung semua fitur bahasa Solidity dan kemungkinan mengeluarkan banyak peringatan.
Dalam hal ini melaporkan fitur yang tidak didukung, analisisnya mungkin tidak sepenuhnya baik.

.. index:: source file, ! import, module, source unit

.. _import:

Mengimpor Source Files lain
============================

Syntax and Semantics
--------------------

Solidity mendukung pernyataan impor untuk membantu memodulasi kode Anda yang
serupa dengan yang tersedia di JavaScript (mulai dari ES6).
Namun, Solidity tidak mendukung konsep `ekspor default <https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export#Description>`_.

Di tingkat global, Anda dapat menggunakan pernyataan impor dengan form berikut:

.. code-block:: solidity

    import "filename";

Bagian ``filename`` disebut *import path*.
Pernyataan ini mengimpor semua simbol global dari "filename" (dan simbol yang diimpor di sana)
ke dalam lingkup global saat ini (berbeda dari ES6 tetapi backwards-compatible untuk Solidity).
Form tersebut tidak disarankan untuk digunakan, karena secara tak terduga mencemari namespace.
Jika Anda menambahkan item top-level baru di dalam "filename", item tersebut secara otomatis
muncul di semua file yang diimpor dari "filename". Lebih baik mengimpor simbol tertentu
secara eksplisit.

Contoh berikut membuat simbol global baru ``symbolName`` yang anggotanya
adalah semua simbol global dari ``"filename"``:

.. code-block:: solidity

    import * as symbolName from "filename";

yang menghasilkan semua simbol global yang tersedia dalam format ``symbolName.symbol``.

A variant of this syntax that is not part of ES6, but possibly useful is:

.. code-block:: solidity

  import "filename" as symbolName;

yang setara dengan ``import * sebagai nama simbol dari "filename";``.

Jika terjadi tabrakan penamaan,Anda dapat mengganti nama simbol saat mengimpor. Sebagai contoh,
kode di bawah ini membuat simbol global baru ``alias`` dan ``symbol2`` yang merujuk
``symbol1`` dan ``symbol2`` masing-masing dari dalam ``"filename"``.

.. code-block:: solidity

    import {symbol1 as alias, symbol2} from "filename";

.. index:: virtual filesystem, source unit name, import; path, filesystem path, import callback, Remix IDE

Import Path
------------

Agar dapat mendukung build yang dapat direproduksi di semua platform, kompiler Solidity
harus mengabstraksi detail sistem file tempat file sumber disimpan.
Untuk alasan ini import path tidak merujuk langsung ke file di host filesystem.
Sebaliknya kompilerr memelihara database internal (*virtual filesystem* atau *VFS* singkatnya)
di mana setiap unit sumber diberi *source unit name* unik yang merupakan pengidentifikasi buram dan tidak terstruktur.
Import path yang ditentukan dalam pernyataan import diterjemahkan ke dalam nama unit sumber dan digunakan untuk
menemukan unit sumber yang sesuai dalam database ini.

Dengan menggunakan :ref:`Standard JSON <compiler-api>` API, dimungkinkan untuk secara langsung memberikan
nama dan konten semua file sumber sebagai bagian dari input kompiler.
Dalam hal ini nama unit sumber benar-benar arbitrer.
Namun, jika Anda ingin kompilerr menemukan dan memuat kode sumber secara otomatis ke dalam VFS,
nama unit sumber Anda perlu disusun sedemikian rupa sehingga memungkinkan
:ref:`import callback <import-callback>` untuk mencari mereka.
Saat menggunakan kompiler command-line, callback impor default hanya mendukung pemuatan
kode sumber dari filesystem host, yang berarti bahwa nama unit sumber Anda harus berupa jalur.
Beberapa environment menyediakan callback khusus yang lebih fleksibel.
Misalnya `Remix IDE <https://remix.ethereum.org/>`_ menyediakan yang memungkinkan Anda
`mengimpor file dari HTTP, IPFS, dan URL Swarm atau merujuk langsung ke paket di
registri NPM <https://remix- ide.readthedocs.io/en/latest/import.html>`_.

Untuk deskripsi lengkap tentang virtual filesysytem dan logika resolusi jalur yang digunakan
oleh kompilerr, lihat :ref:`Path Resolution <path-resolution>`.

.. index:: ! comment, natspec

Komentar (Comments)
===================

Single-line comments (``//``) dan multi-line comments (``/*...*/``) dimungkinkan.

.. code-block:: solidity

    // This is a single-line comment.

    /*
    This is a
    multi-line comment.
    */

.. note::
  Sebuah single-line comment diakhiri oleh terminator baris unicode apa pun
  (LF, VF, FF, CR, NEL, LS or PS) di encoding UTF-8. Terminator masih menjadi bagian dari
  kode sumber setelah comment, jadi jika itu bukan simbol ASCII
  ( NEL, LS dan PS), itu akan menyebabkan kesalahan parser.

Selain itu, ada jenis komentar lain yang disebut komentar NatSpec,
yang dirinci dalam :ref:`style guide<style_guide_natspec>`. Mereka ditulis dengan
garis miring tiga (``///``) atau blok asterisk ganda (``/** ... */``) dan
mereka harus digunakan langsung di atas deklarasi atau pernyataan fungsi.
=======
********************************
Layout of a Solidity Source File
********************************

Source files can contain an arbitrary number of
:ref:`contract definitions<contract_structure>`, import_ ,
:ref:`pragma<pragma>` and :ref:`using for<using-for>` directives and
:ref:`struct<structs>`, :ref:`enum<enums>`, :ref:`function<functions>`, :ref:`error<errors>`
and :ref:`constant variable<constants>` definitions.

.. index:: ! license, spdx

SPDX License Identifier
=======================

Trust in smart contracts can be better established if their source code
is available. Since making source code available always touches on legal problems
with regards to copyright, the Solidity compiler encourages the use
of machine-readable `SPDX license identifiers <https://spdx.org>`_.
Every source file should start with a comment indicating its license:

``// SPDX-License-Identifier: MIT``

The compiler does not validate that the license is part of the
`list allowed by SPDX <https://spdx.org/licenses/>`_, but
it does include the supplied string in the :ref:`bytecode metadata <metadata>`.

If you do not want to specify a license or if the source code is
not open-source, please use the special value ``UNLICENSED``.
Note that ``UNLICENSED`` (no usage allowed, not present in SPDX license list)
is different from ``UNLICENSE`` (grants all rights to everyone).
Solidity follows `the npm recommendation <https://docs.npmjs.com/cli/v7/configuring-npm/package-json#license>`_.

Supplying this comment of course does not free you from other
obligations related to licensing like having to mention
a specific license header in each source file or the
original copyright holder.

The comment is recognized by the compiler anywhere in the file at the
file level, but it is recommended to put it at the top of the file.

More information about how to use SPDX license identifiers
can be found at the `SPDX website <https://spdx.org/ids-how>`_.


.. index:: ! pragma

.. _pragma:

Pragmas
=======

The ``pragma`` keyword is used to enable certain compiler features
or checks. A pragma directive is always local to a source file, so
you have to add the pragma to all your files if you want to enable it
in your whole project. If you :ref:`import<import>` another file, the pragma
from that file does *not* automatically apply to the importing file.

.. index:: ! pragma, version

.. _version_pragma:

Version Pragma
--------------

Source files can (and should) be annotated with a version pragma to reject
compilation with future compiler versions that might introduce incompatible
changes. We try to keep these to an absolute minimum and
introduce them in a way that changes in semantics also require changes
in the syntax, but this is not always possible. Because of this, it is always
a good idea to read through the changelog at least for releases that contain
breaking changes. These releases always have versions of the form
``0.x.0`` or ``x.0.0``.

The version pragma is used as follows: ``pragma solidity ^0.5.2;``

A source file with the line above does not compile with a compiler earlier than version 0.5.2,
and it also does not work on a compiler starting from version 0.6.0 (this
second condition is added by using ``^``). Because
there will be no breaking changes until version ``0.6.0``, you can
be sure that your code compiles the way you intended. The exact version of the
compiler is not fixed, so that bugfix releases are still possible.

It is possible to specify more complex rules for the compiler version,
these follow the same syntax used by `npm <https://docs.npmjs.com/cli/v6/using-npm/semver>`_.

.. note::
  Using the version pragma *does not* change the version of the compiler.
  It also *does not* enable or disable features of the compiler. It just
  instructs the compiler to check whether its version matches the one
  required by the pragma. If it does not match, the compiler issues
  an error.

ABI Coder Pragma
----------------

By using ``pragma abicoder v1`` or ``pragma abicoder v2`` you can
select between the two implementations of the ABI encoder and decoder.

The new ABI coder (v2) is able to encode and decode arbitrarily nested
arrays and structs. It might produce less optimal code and has not
received as much testing as the old encoder, but is considered
non-experimental as of Solidity 0.6.0. You still have to explicitly
activate it using ``pragma abicoder v2;``. Since it will be
activated by default starting from Solidity 0.8.0, there is the option to select
the old coder using ``pragma abicoder v1;``.

The set of types supported by the new encoder is a strict superset of
the ones supported by the old one. Contracts that use it can interact with ones
that do not without limitations. The reverse is possible only as long as the
non-``abicoder v2`` contract does not try to make calls that would require
decoding types only supported by the new encoder. The compiler can detect this
and will issue an error. Simply enabling ``abicoder v2`` for your contract is
enough to make the error go away.

.. note::
  This pragma applies to all the code defined in the file where it is activated,
  regardless of where that code ends up eventually. This means that a contract
  whose source file is selected to compile with ABI coder v1
  can still contain code that uses the new encoder
  by inheriting it from another contract. This is allowed if the new types are only
  used internally and not in external function signatures.

.. note::
  Up to Solidity 0.7.4, it was possible to select the ABI coder v2
  by using ``pragma experimental ABIEncoderV2``, but it was not possible
  to explicitly select coder v1 because it was the default.

.. index:: ! pragma, experimental

.. _experimental_pragma:

Experimental Pragma
-------------------

The second pragma is the experimental pragma. It can be used to enable
features of the compiler or language that are not yet enabled by default.
The following experimental pragmas are currently supported:


ABIEncoderV2
~~~~~~~~~~~~

Because the ABI coder v2 is not considered experimental anymore,
it can be selected via ``pragma abicoder v2`` (please see above)
since Solidity 0.7.4.

.. _smt_checker:

SMTChecker
~~~~~~~~~~

This component has to be enabled when the Solidity compiler is built
and therefore it is not available in all Solidity binaries.
The :ref:`build instructions<smt_solvers_build>` explain how to activate this option.
It is activated for the Ubuntu PPA releases in most versions,
but not for the Docker images, Windows binaries or the
statically-built Linux binaries. It can be activated for solc-js via the
`smtCallback <https://github.com/ethereum/solc-js#example-usage-with-smtsolver-callback>`_ if you have an SMT solver
installed locally and run solc-js via node (not via the browser).

If you use ``pragma experimental SMTChecker;``, then you get additional
:ref:`safety warnings<formal_verification>` which are obtained by querying an
SMT solver.
The component does not yet support all features of the Solidity language and
likely outputs many warnings. In case it reports unsupported features, the
analysis may not be fully sound.

.. index:: source file, ! import, module, source unit

.. _import:

Importing other Source Files
============================

Syntax and Semantics
--------------------

Solidity supports import statements to help modularise your code that
are similar to those available in JavaScript
(from ES6 on). However, Solidity does not support the concept of
a `default export <https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export#Description>`_.

At a global level, you can use import statements of the following form:

.. code-block:: solidity

    import "filename";

The ``filename`` part is called an *import path*.
This statement imports all global symbols from "filename" (and symbols imported there) into the
current global scope (different than in ES6 but backwards-compatible for Solidity).
This form is not recommended for use, because it unpredictably pollutes the namespace.
If you add new top-level items inside "filename", they automatically
appear in all files that import like this from "filename". It is better to import specific
symbols explicitly.

The following example creates a new global symbol ``symbolName`` whose members are all
the global symbols from ``"filename"``:

.. code-block:: solidity

    import * as symbolName from "filename";

which results in all global symbols being available in the format ``symbolName.symbol``.

A variant of this syntax that is not part of ES6, but possibly useful is:

.. code-block:: solidity

  import "filename" as symbolName;

which is equivalent to ``import * as symbolName from "filename";``.

If there is a naming collision, you can rename symbols while importing. For example,
the code below creates new global symbols ``alias`` and ``symbol2`` which reference
``symbol1`` and ``symbol2`` from inside ``"filename"``, respectively.

.. code-block:: solidity

    import {symbol1 as alias, symbol2} from "filename";

.. index:: virtual filesystem, source unit name, import; path, filesystem path, import callback, Remix IDE

Import Paths
------------

In order to be able to support reproducible builds on all platforms, the Solidity compiler has to
abstract away the details of the filesystem where source files are stored.
For this reason import paths do not refer directly to files in the host filesystem.
Instead the compiler maintains an internal database (*virtual filesystem* or *VFS* for short) where
each source unit is assigned a unique *source unit name* which is an opaque and unstructured identifier.
The import path specified in an import statement is translated into a source unit name and used to
find the corresponding source unit in this database.

Using the :ref:`Standard JSON <compiler-api>` API it is possible to directly provide the names and
content of all the source files as a part of the compiler input.
In this case source unit names are truly arbitrary.
If, however, you want the compiler to automatically find and load source code into the VFS, your
source unit names need to be structured in a way that makes it possible for an :ref:`import callback
<import-callback>` to locate them.
When using the command-line compiler the default import callback supports only loading source code
from the host filesystem, which means that your source unit names must be paths.
Some environments provide custom callbacks that are more versatile.
For example the `Remix IDE <https://remix.ethereum.org/>`_ provides one that
lets you `import files from HTTP, IPFS and Swarm URLs or refer directly to packages in NPM registry
<https://remix-ide.readthedocs.io/en/latest/import.html>`_.

For a complete description of the virtual filesystem and the path resolution logic used by the
compiler see :ref:`Path Resolution <path-resolution>`.

.. index:: ! comment, natspec

Comments
========

Single-line comments (``//``) and multi-line comments (``/*...*/``) are possible.

.. code-block:: solidity

    // This is a single-line comment.

    /*
    This is a
    multi-line comment.
    */

.. note::
  A single-line comment is terminated by any unicode line terminator
  (LF, VF, FF, CR, NEL, LS or PS) in UTF-8 encoding. The terminator is still part of
  the source code after the comment, so if it is not an ASCII symbol
  (these are NEL, LS and PS), it will lead to a parser error.

Additionally, there is another type of comment called a NatSpec comment,
which is detailed in the :ref:`style guide<style_guide_natspec>`. They are written with a
triple slash (``///``) or a double asterisk block (``/** ... */``) and
they should be used directly above function declarations or statements.
>>>>>>> 803585d2e676588f130e37329ecfbfc7bc51dcd7
