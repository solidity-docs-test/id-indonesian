<<<<<<< HEAD
.. index:: optimizer, optimiser, common subexpression elimination, constant propagation
.. _optimizer:

*********
Optimizer
*********

Kompiler Solidity menggunakan dua modul pengoptimal yang berbeda: Pengoptimal "lama"
yang beroperasi pada tingkat opcode dan pengoptimal "baru" yang beroperasi pada kode Yul IR.

Pengoptimal berbasis opcode menerapkan serangkaian `aturan penyederhanaan <https://github.com/ethereum/solidity/blob/develop/libevmasm/RuleList.h>`_
untuk opcode. Ini juga menggabungkan set kode yang sama dan menghapus kode yang tidak digunakan.

Pengoptimal berbasis Yul jauh lebih kuat, karena dapat bekerja di seluruh panggilan fungsi. Misalnya, arbitrary jump
tidak dimungkinkan di Yul, jadi dimungkinkan untuk menghitung efek samping dari setiap fungsi. Pertimbangkan dua panggilan fungsi,
di mana yang pertama tidak mengubah penyimpanan dan yang kedua mengubah penyimpanan.
Jika argumen dan nilai pengembaliannya tidak bergantung satu sama lain, kita dapat menyusun
ulang pemanggilan fungsi.
Demikian pula, jika suatu fungsi bebas efek samping dan hasilnya dikalikan dengan nol, Anda dapat
menghapus panggilan fungsi sepenuhnya.

Saat ini, parameter ``--optimize`` mengaktifkan pengoptimal berbasis opcode untuk bytecode yang dihasilkan
dan pengoptimal Yuluntuk kode Yul yang dihasilkan secara internal, misalnya untuk ABI coder v2.
Seseorang dapat menggunakan ``solc --ir-optimized --optimize`` untuk menghasilkan Yul IR
eksperimental yang dioptimalkan untuk sumber Soliditas. Demikian pula, seseorang dapat menggunakan ``solc --strict-assembly --optimize`` untuk mode Yul stand-alone.

Anda dapat menemukan detail lebih lanjut tentang modul pengoptimal dan langkah pengoptimalannya di bawah.

Manfaat Mengoptimalkan Kode Solidity
====================================

Secara keseluruhan, pengoptimal mencoba menyederhanakan ekspresi rumit, yang mengurangi ukuran kode dan biaya eksekusi,
yaitu, dapat mengurangi gas yang dibutuhkan untuk penerapan kontrak serta untuk panggilan eksternal yang dilakukan ke kontrak.
Ini juga mengkhususkan atau inline fungsi. Khususnya function inlining adalah operasi yang
dapat menyebabkan kode yang jauh lebih besar, tetapi sering dilakukan karena menghasilkan
peluang untuk lebih banyak penyederhanaan.


Perbedaan antara Kode yang Dioptimalkan dan Tidak Dioptimalkan
==============================================================

Umumnya, perbedaan yang paling terlihat adalah ekspresi konstan dievaluasi pada waktu kompilasi.
Ketika datang ke output ASM, kita juga dapat melihat pengurangan blok kode yang setara atau duplikat
(bandingkan output dari flag ``--asm`` dan ``--asm --optimize``). Namun,
ketika menyangkut Yul/representasi menengah, mungkin ada perbedaan
yang signifikan, misalnya, fungsi dapat disejajarkan, digabungkan, atau ditulis ulang untuk menghilangkan
redundansi, dll. (bandingkan output antara flag ``--ir`` dan
``--optimize --ir-optimized``).

.. _optimizer-parameter-runs:

Optimizer Parameter Runs
========================

Jumlah proses (``--optimize-runs``) menentukan secara kasar seberapa sering setiap opcode dari kode yang
di-deploy akan dieksekusi sepanjang masa kontrak. Ini berarti ini adalah parameter trade-off antara
ukuran kode (biaya penerapan) dan biaya eksekusi kode (biaya setelah penerapan).
Parameter "berjalan" dari "1" akan menghasilkan kode yang pendek tapi mahal. Sebaliknya, parameter "run"
yang lebih besar akan menghasilkan kode yang lebih lama tetapi lebih hemat gas. Nilai maksimum parameter
adalah ``2**32-1``.

.. note::

    Kesalahpahaman yang umum adalah bahwa parameter ini menentukan jumlah iterasi pengoptimal.
    Ini tidak benar: Pengoptimal akan selalu berjalan sesering yang masih dapat meningkatkan kode.

Modul Opcode-Based Optimizer
============================

Modul pengoptimal berbasis opcode beroperasi pada kode perakitan. Ini membagi
urutan instruksi menjadi blok dasar di ``JUMPs`` dan ``JUMPDESTs``.
Di dalam blok-blok ini, pengoptimal menganalisis instruksi dan mencatat setiap modifikasi pada stack,
memori, atau storage sebagai ekspresi yang terdiri dari instruksi dan
daftar argumen yang merupakan penunjuk ke ekspresi lain.

Selain itu, pengoptimal berbasis opcode
menggunakan komponen yang disebut "CommonSubexpressionEliminator" yang di antara
tugas-tugas lainnya, menemukan ekspresi yang selalu sama (pada setiap input) dan menggabungkannya
ke dalam kelas ekspresi. Ini pertama kali mencoba menemukan setiap ekspresi baru
dalam daftar ekspresi yang sudah dikenal. Jika tidak ada kecocokan seperti itu yang ditemukan,
ekspresi akan disederhanakan menurut aturan seperti ``constant + constant = sum_of_constants`` atau ``X * 1 = X``. Karena ini adalah
proses rekursif, kita juga dapat menerapkan aturan yang terakhir jika faktor kedua adalah ekspresi yang
lebih kompleks yang kita tahu selalu bernilai satu.

Langkah-langkah pengoptimal tertentu secara simbolis melacak lokasi penyimpanan dan memori. Misalnya, informasi
ini digunakan untuk menghitung hash Keccak-256 yang dapat dievaluasi selama waktu kompilasi. Pertimbangkan
urutannya:

.. code-block:: none

    PUSH 32
    PUSH 0
    CALLDATALOAD
    PUSH 100
    DUP2
    MSTORE
    KECCAK256

atau yul yang setara

.. code-block:: yul

    let x := calldataload(0)
    mstore(x, 100)
    let value := keccak256(x, 32)

Dalam hal ini, pengoptimal melacak nilai di lokasi memori ``calldataload(0)`` lalu
menyadari bahwa hash Keccak-256 dapat dievaluasi pada waktu kompilasi. Ini hanya berfungsi jika tidak ada
instruksi lain yang memodifikasi memori antara ``mstore`` dan ``keccak256``. Jadi jika ada
instruksi yang menulis ke memori (atau penyimpanan), maka kita perlu menghapus pengetahuan memori
saat ini (atau storage). Namun, ada pengecualian untuk penghapusan ini, ketika kita dapat dengan mudah melihat
instruksi tidak menulis ke lokasi tertentu.

Sebagai contoh,

.. code-block:: yul

    let x := calldataload(0)
    mstore(x, 100)
    // Current knowledge memory location x -> 100
    let y := add(x, 32)
    // Does not clear the knowledge that x -> 100, since y does not write to [x, x + 32)
    mstore(y, 200)
    // This Keccak-256 can now be evaluated
    let value := keccak256(x, 32)

Oleh karena itu, modifikasi lokasi penyimpanan dan memori, misalnya lokasi ``l``, harus menghapus
pengetahuan tentang penyimpanan atau lokasi memori yang mungkin sama dengan ``l``. Lebih khusus lagi, untuk
penyimpanan, pengoptimal harus menghapus semua pengetahuan tentang lokasi simbolis, yang mungkin sama dengan ``l``
dan untuk memori, pengoptimal harus menghapus semua pengetahuan tentang lokasi simbolis yang mungkin tidak
berjarak setidaknya 32 byte. Jika ``m`` menunjukkan lokasi arbitrer, maka keputusan penghapusan ini dilakukan dengan
menghitung nilai ``sub(l, m)``. Untuk penyimpanan, jika nilai ini dievaluasi ke literal yang bukan nol, maka
pengetahuan tentang ``m`` akan disimpan. Untuk memori, jika nilai dievaluasi ke literal antara ``32`` dan ``2**256 - 32``, maka pengetahuan tentang ``m`` akan disimpan.
Dalam semua kasus lain, pengetahuan tentang ``m`` akan dihapus.

Setelah proses ini, kita tahu ekspresi mana yang harus ada di stack
di akhir, dan memiliki daftar modifikasi pada memori dan penyimpanan. Informasi ini
disimpan bersama dengan blok dasar dan digunakan untuk menghubungkannya. Selanjutnya,
pengetahuan tentang konfigurasi stack, penyimpanan, dan memori diteruskan ke
blok berikutnya.

Jika kita mengetahui target dari semua instruksi ``JUMP`` dan ``JUMPI``,
kita dapat membangun grafik aliran kendali program yang lengkap. Jika hanya ada satu
target yang tidak kita ketahui (hal ini dapat terjadi karena pada prinsipnya, target jump dapat
dihitung dari input), kita harus menghapus semua pengetahuan tentang state input
dari suatu blok karena dapat menjadi target yang tidak diketahui ``JUMP``. Jika modul pengoptimal berbasis opcode
menemukan ``JUMPI`` yang kondisinya dievaluasi menjadi konstan, modul tersebut mengubahnya
menjadi lompatan tanpa syarat.

Sebagai langkah terakhir, kode di setiap blok dibuat ulang. Pengoptimal membuat
grafik dependency dari ekspresi pada tumpukan di akhir blok,
dan menghapus setiap operasi yang bukan bagian dari grafik ini. Ini menghasilkan kode
yang menerapkan modifikasi pada memori dan penyimpanan dalam urutan yang dibuat
dalam kode asli (menjatuhkan modifikasi yang ditemukan tidak diperlukan). Akhirnya,
itu menghasilkan semua nilai yang diperlukan untuk berada dalam stack di tempat yang benar.

Langkah-langkah ini diterapkan pada setiap blok dasar dan kode yang baru dibuat digunakan
sebagai pengganti jika lebih kecil. Jika blok dasar dipecah pada ``JUMPI`` dan selama analisis,
kondisinya dievaluasi menjadi konstanta, ``JUMPI`` diganti berdasarkan nilai konstanta. Jadi kode seperti

.. code-block:: solidity

    uint x = 7;
    data[7] = 9;
    if (data[x] != x + 2) // this condition is never true
      return 2;
    else
      return 1;

disederhanakan menjadi ini:

.. code-block:: solidity

    data[7] = 9;
    return 1;

Simple Inlining
---------------

Sejak Solidity versi 0.8.2, ada langkah pengoptimal lain yang menggantikan lompatan tertentu ke blok yang
berisi instruksi "sederhana" yang diakhiri dengan "lompatan" dengan salinan instruksi ini.
Ini sesuai dengan inlining fungsi Solidity atau Yul yang sederhana dan kecil. Secara khusus, urutan
``PUSHTAG(tag) JUMP`` dapat diganti, setiap kali ``JUMP`` ditandai sebagai lompat "ke" suatu fungsi
dan di belakang ``tag`` terdapat blok dasar (seperti dijelaskan di atas untuk "CommonSubexpressionEliminator")
yang diakhiri dengan ``JUMP`` lain yang ditandai sebagai lompatan "keluar dari" suatu fungsi.

Secara khusus, pertimbangkan contoh prototipe assembly berikut yang dihasilkan untuk
panggilan ke fungsi Solidity internal:

.. code-block:: text

      tag_return
      tag_f
      jump      // in
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

Selama isi fungsi adalah blok dasar kontinu, "Inliner" dapat menggantikan ``tag_f jump`` dengan
blok di ``tag_f`` menghasilkan:

.. code-block:: text

      tag_return
      ...body of function f...
      jump
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

Sekarang idealnya, langkah-langkah pengoptimal lain yang dijelaskan di atas akan mengakibatkan dorongan tag kembali dipindahkan
menuju lompatan yang tersisa menghasilkan:

.. code-block:: text

      ...body of function f...
      tag_return
      jump
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

Dalam situasi ini "PeepholeOptimizer" akan menghapus lompatan kembali. Idealnya, semua ini bisa dilakukan
untuk semua referensi ke ``tag_f`` membiarkannya tidak digunakan, s.t. itu dapat dihapus, menghasilkan:

.. code-block:: text

    ...body of function f...
    ...opcodes after call to f...

Jadi panggilan ke fungsi ``f`` disejajarkan dan definisi asli ``f`` dapat dihapus.

Inlining seperti ini dicoba, setiap kali heuristik menunjukkan bahwa inlining lebih murah selama
masa kontrak daripada tidak inlining. Heuristik ini bergantung pada ukuran badan fungsi, jumlah
referensi lain ke tagnya (mendekati jumlah panggilan ke fungsi) dan
jumlah eksekusi kontrak yang diharapkan (parameter pengoptimal global "berjalan").


Modul Yul-Based Optimizer
=========================

Pengoptimal berbasis Yul terdiri dari beberapa tahap dan komponen yang semuanya mengubah AST
dengan cara yang setara secara semantik. Tujuannya adalah untuk mendapatkan kode yang lebih pendek atau setidaknya
hanya sedikit lebih panjang tetapi akan memungkinkan
langkah pengoptimalan lebih lanjut.

.. warning::

    Karena pengoptimal sedang dalam pengembangan yang berat, informasi di sini mungkin sudah usang.
    Jika Anda mengandalkan fungsi tertentu, hubungi tim secara langsung.

Pengoptimal saat ini mengikuti strategi murni serakah dan tidak melakukan backtracking.

Semua komponen modul pengoptimal berbasis Yul dijelaskan di bawah ini.
Langkah-langkah transformasi berikut adalah komponen utama:

- SSA Transform
- Common Subexpression Eliminator
- Expression Simplifier
- Redundant Assign Eliminator
- Full Inliner

Optimizer Step
--------------

Ini adalah daftar semua langkah pengoptimal berbasis Yul yang diurutkan berdasarkan abjad. Anda dapat menemukan informasi
lebih lanjut tentang masing-masing langkah dan urutannya di bawah ini.

- :ref:`block-flattener`.
- :ref:`circular-reference-pruner`.
- :ref:`common-subexpression-eliminator`.
- :ref:`conditional-simplifier`.
- :ref:`conditional-unsimplifier`.
- :ref:`control-flow-simplifier`.
- :ref:`dead-code-eliminator`.
- :ref:`equivalent-function-combiner`.
- :ref:`expression-joiner`.
- :ref:`expression-simplifier`.
- :ref:`expression-splitter`.
- :ref:`for-loop-condition-into-body`.
- :ref:`for-loop-condition-out-of-body`.
- :ref:`for-loop-init-rewriter`.
- :ref:`expression-inliner`.
- :ref:`full-inliner`.
- :ref:`function-grouper`.
- :ref:`function-hoister`.
- :ref:`function-specializer`.
- :ref:`literal-rematerialiser`.
- :ref:`load-resolver`.
- :ref:`loop-invariant-code-motion`.
- :ref:`redundant-assign-eliminator`.
- :ref:`reasoning-based-simplifier`.
- :ref:`rematerialiser`.
- :ref:`SSA-reverser`.
- :ref:`SSA-transform`.
- :ref:`structural-simplifier`.
- :ref:`unused-function-parameter-pruner`.
- :ref:`unused-pruner`.
- :ref:`var-decl-initializer`.

Meimilih Optimizations
-----------------------

Secara default, pengoptimal menerapkan urutan langkah pengoptimalan yang telah ditentukan sebelumnya
ke rakitan yang dihasilkan. Anda dapat mengganti urutan ini dan menyediakan urutan Anda sendiri menggunakan
opsi ``--yul-optimizations``:

.. code-block:: bash

    solc --optimize --ir-optimized --yul-optimizations 'dhfoD[xarrscLMcCTU]uljmul'

Urutan di dalam ``[...]`` akan diterapkan beberapa kali dalam satu lingkaran hingga kode Yul tetap tidak berubah
atau hingga jumlah putaran maksimum (saat ini 12) telah tercapai.

Singkatan yang tersedia tercantum dalam `Yul optimizer docs <yul.rst#optimization-step-sequence>`_.

Preprocessing
-------------

Komponen preprocessing melakukan transformasi untuk membuat program
menjadi bentuk normal tertentu yang lebih mudah untuk dikerjakan. Bentuk normal
ini disimpan selama sisa proses optimasi.

.. _disambiguator:

Disambiguator
^^^^^^^^^^^^^

Disambiguator mengambil AST dan mengembalikan salinan baru di mana semua pengidentifikasi
memiliki nama unik di AST input. Ini adalah prasyarat untuk semua tahap pengoptimal lainnya.
Salah satu manfaatnya adalah pencarian identifier tidak perlu memperhitungkan cakupan yang
menyederhanakan analisis yang diperlukan untuk langkah-langkah lain.

Semua tahapan selanjutnya memiliki properti bahwa semua nama tetap unik. Ini berarti jika
identifier baru perlu diperkenalkan, nama unik baru akan dibuat.

.. _function-hoister:

FunctionHoister
^^^^^^^^^^^^^^^

Function hoister memindahkan semua definisi fungsi ke ujung blok paling atas. Ini adalah
transformasi ekuivalen semantik asalkan dilakukan setelah tahap disambiguasi. Alasannya
adalah bahwa memindahkan definisi ke blok tingkat yang lebih tinggi tidak dapat mengurangi visibilitasnya
dan tidak mungkin untuk merujuk variabel yang didefinisikan dalam fungsi yang berbeda.

Manfaat dari tahap ini adalah definisi fungsi dapat dicari dengan lebih mudah
dan fungsi dapat dioptimalkan secara terpisah tanpa harus melintasi AST sepenuhnya.

.. _function-grouper:

FunctionGrouper
^^^^^^^^^^^^^^^

Fungsi grouper harus diterapkan setelah disambiguator dan function hoister.
Efeknya adalah semua elemen teratas yang bukan definisi fungsi dipindahkan
menjadi satu blok yang merupakan pernyataan pertama dari blok root.

Setelah langkah ini, sebuah program memiliki bentuk normal berikut:

.. code-block:: text

    { I F... }

Di mana ``I`` adalah blok (berpotensi kosong) yang tidak mengandung definisi fungsi apa pun (bahkan tidak secara rekursif)
dan ``F`` adalah daftar definisi fungsi sehingga tidak ada fungsi yang berisi definisi fungsi.

Manfaat dari tahap ini adalah kita selalu tahu di mana daftar fungsi dimulai.

.. _for-loop-condition-into-body:

ForLoopConditionIntoBody
^^^^^^^^^^^^^^^^^^^^^^^^

Transformasi ini memindahkan kondisi loop-iterasi dari for-loop ke badan loop.
Kita membutuhkan transformasi ini karena :ref:`expression-splitter` tidak akan
berlaku untuk ekspresi kondisi iterasi (``C`` dalam contoh berikut).

.. code-block:: text

    for { Init... } C { Post... } {
        Body...
    }

diubah menjadi

.. code-block:: text

    for { Init... } 1 { Post... } {
        if iszero(C) { break }
        Body...
    }

Transformasi ini juga dapat berguna saat dipasangkan dengan ``LoopInvariantCodeMotion``, karena
invarian dalam kondisi loop-invarian kemudian dapat diambil di luar loop.

.. _for-loop-init-rewriter:

ForLoopInitRewriter
^^^^^^^^^^^^^^^^^^^

Transformasi ini memindahkan bagian inisialisasi dari for-loop ke sebelum loop:

.. code-block:: text

    for { Init... } C { Post... } {
        Body...
    }

diubah menjadi

.. code-block:: text

    Init...
    for {} C { Post... } {
        Body...
    }

Ini memudahkan proses pengoptimalan lainnya karena kita dapat mengabaikan
aturan pelingkupan rumit dari blok untuk inisialisasi loop.

.. _var-decl-initializer:

VarDeclInitializer
^^^^^^^^^^^^^^^^^^
Langkah ini menulis ulang deklarasi variabel sehingga semuanya diinisialisasi.
Deklarasi seperti ``let x, y`` dipecah menjadi beberapa pernyataan deklarasi

Hanya mendukung inisialisasi dengan nol literal untuk saat ini.

Pseudo-SSA Transformation
-------------------------

Tujuan dari komponen ini adalah untuk membuat program menjadi bentukyang
lebih panjang, sehingga komponen lain dapat lebih mudah bekerja dengannya.
Representasi akhir akan mirip dengan bentuk static-single-assignment (SSA),
dengan perbedaan bahwa itu tidak menggunakan fungsi "phi" eksplisit yang
menggabungkan nilai dari cabang aliran kontrol yang berbeda karena fitur
seperti itu tidak ada dalam bahasa Yul. Sebagai gantinya, ketika aliran
kontrol bergabung, jika variabel ditetapkan kembali di salah satu cabang,
variabel SSA baru dideklarasikan untuk mempertahankan nilainya saat ini,
sehingga ekspresi berikut masih hanya perlu merujuk variabel SSA.

Contoh transformasinya adalah sebagai berikut:

.. code-block:: yul

    {
        let a := calldataload(0)
        let b := calldataload(0x20)
        if gt(a, 0) {
            b := mul(b, 0x20)
        }
        a := add(a, 1)
        sstore(a, add(b, 0x20))
    }


Ketika semua langkah transformasi berikut diterapkan, program akan terlihat sebagai berikut:

.. code-block:: yul

    {
        let _1 := 0
        let a_9 := calldataload(_1)
        let a := a_9
        let _2 := 0x20
        let b_10 := calldataload(_2)
        let b := b_10
        let _3 := 0
        let _4 := gt(a_9, _3)
        if _4
        {
            let _5 := 0x20
            let b_11 := mul(b_10, _5)
            b := b_11
        }
        let b_12 := b
        let _6 := 1
        let a_13 := add(a_9, _6)
        let _7 := 0x20
        let _8 := add(b_12, _7)
        sstore(a_13, _8)
    }

Perhatikan bahwa satu-satunya variabel yang ditetapkan ulang dalam cuplikan ini adalah ``b``.
Penetapan ulang ini tidak dapat dihindari karena ``b`` memiliki nilai yang berbeda tergantung
pada aliran kontrol. Semua variabel lain tidak pernah mengubah nilainya setelah didefinisikan.
Keuntungan dari properti ini adalah bahwa variabel dapat dengan bebas dipindahkan dan referensi
untuk mereka dapat ditukar dengan nilai awal mereka (dan sebaliknya), selama nilai-nilai ini masih
berlaku dalam konteks baru.

Tentu saja, kode di sini masih jauh dari optimal. Sebaliknya, itu jauh lebih lama. Harapannya adalah
kode ini akan lebih mudah untuk dikerjakan dan selanjutnya, ada langkah-langkah pengoptimal yang membatalkan
perubahan ini dan membuat kode lebih kompak lagi di akhir.

.. _expression-splitter:

ExpressionSplitter
^^^^^^^^^^^^^^^^^^

Pembagi ekspresi mengubah ekspresi seperti ``add(mload(0x123), mul(mload(0x456), 0x20))``
menjadi urutan deklarasi variabel unik yang diberi sub-ekspresi dari ekspresi itu sehingga
setiap pemanggilan fungsi memiliki hanya variabel atau literal sebagai argumen.

Di atas akan diubah menjadi

.. code-block:: yul

    {
        let _1 := mload(0x123)
        let _2 := mul(_1, 0x20)
        let _3 := mload(0x456)
        let z := add(_3, _2)
    }

Perhatikan bahwa transformasi ini tidak mengubah urutan opcode atau panggilan fungsi.

Ini tidak diterapkan pada kondisi iterasi loop, karena aliran kontrol loop tidak mengizinkan "outlining"
ini dari ekspresi dalam dalam semua kasus. Kita dapat menghindari batasan ini dengan menerapkan
:ref:`for-loop-condition-into-body` untuk memindahkan kondisi iterasi ke dalam loop body.

Program akhir harus dalam bentuk sedemikian rupa (dengan pengecualian kondisi loop)
panggilan fungsi tidak dapat muncul bersarang di dalam ekspresi
dan semua argumen pemanggilan fungsi harus berupa literal atau variabel.

Manfaat dari formulir ini adalah lebih mudah untuk mengurutkan ulang urutan opcode
dan juga lebih mudah untuk melakukan inlining panggilan fungsi. Selain itu, lebih sederhana
untuk mengganti bagian individu dari ekspresi atau mengatur ulang "pohon ekspresi".
Kekurangannya adalah bahwa kode tersebut jauh lebih sulit untuk dibaca oleh manusia.

.. _SSA-transform:

SSATransform
^^^^^^^^^^^^

Tahap ini mencoba mengganti penugasan yang berulang-ulang menjadi
variabel yang ada dengan deklarasi variabel baru sebanyak
mungkin.
Penugasan kembali masih ada, tetapi semua referensi ke
variabel yang ditugaskan kembali digantikan oleh variabel yang baru dideklarasikan.

Contoh:

.. code-block:: yul

    {
        let a := 1
        mstore(a, 2)
        a := 3
    }

diubah menjadi

.. code-block:: yul

    {
        let a_1 := 1
        let a := a_1
        mstore(a_1, 2)
        let a_3 := 3
        a := a_3
    }

Semantik yang tepat:

Untuk setiap variabel ``a`` yang ditetapkan ke suatu tempat dalam kode
(variabel yang dideklarasikan dengan nilai dan tidak pernah ditetapkan ulang
tidak dimodifikasi) lakukan transformasi berikut:

- ganti ``let a := v`` dengan ``let a_i := v   let a := a_i``
- ganti ``a := v`` dengan ``let a_i := v   a := a_i`` dimana ``i`` adalah angka sedemikian rupa sehingga ``a_i`` belum digunakan.

Selanjutnya, selalu catat nilai ``i`` saat ini yang digunakan untuk ``a`` dan ganti masing-masing
referensi ke ``a`` dengan ``a_i``.
Nilai mapping saat ini dihapus untuk variabel ``a`` di akhir setiap blok
di mana itu ditugaskan ke dan di akhir blok for loop init jika ditugaskan
di dalam for loop body atau post block.
Jika nilai variabel dihapus sesuai dengan aturan di atas dan variabel dideklarasikan di luar
blok, variabel SSA baru akan dibuat di lokasi di mana aliran kontrol bergabung,
ini termasuk awal dari loop post/body block dan lokasi tepat setelahnya
Pernyataan If/Switch/ForLoop/Block.

Setelah tahap ini, Redundant Assign Eliminator direkomendasikan untuk
menghapus tugas perantara yang tidak perlu.

Tahap ini memberikan hasil terbaik jika Expression Splitter dan Common Subexpression Eliminator
dijalankan tepat sebelum itu, karena itu tidak menghasilkan jumlah variabel yang berlebihan.
Di sisi lain, Eliminator Subekspresi Umum bisa lebih efisien jika dijalankan setelah
transformasi SSA.

.. _redundant-assign-eliminator:

RedundantAssignEliminator
^^^^^^^^^^^^^^^^^^^^^^^^^

Transformasi SSA selalu menghasilkan penugasan dalam bentuk ``a := a_i``, meskipun
ini mungkin tidak diperlukan dalam banyak kasus, seperti contoh berikut:

.. code-block:: yul

    {
        let a := 1
        a := mload(a)
        a := sload(a)
        sstore(a, 1)
    }

Transformasi SSA mengonversi cuplikan ini menjadi yang berikut:

.. code-block:: yul

    {
        let a_1 := 1
        let a := a_1
        let a_2 := mload(a_1)
        a := a_2
        let a_3 := sload(a_2)
        a := a_3
        sstore(a_3, 1)
    }

Redundant Assign Eliminator menghapus ketiga penetapan ke ``a``, karena
nilai ``a`` tidak digunakan dan dengan demikian mengubah snippet ini menjadi bentuk strict SSA:

.. code-block:: yul

    {
        let a_1 := 1
        let a_2 := mload(a_1)
        let a_3 := sload(a_2)
        sstore(a_3, 1)
    }

Tentu saja bagian-bagian rumit untuk menentukan apakah suatu penugasan berlebihan atau tidak terhubung
dengan aliran kontrol yang bergabung.

Komponen bekerja sebagai berikut secara rinci:

AST dilalui dua kali: dalam langkah pengumpulan informasi dan dalam langkah
penghapusan yang sebenarnya. Selama pengumpulan informasi, kami mempertahankan
mapping dari pernyataan penugasan ke tiga state "unused", "undecided" dan "used"
yang menandakan apakah nilai yang ditetapkan akan digunakan nanti dengan referensi ke variabel.

Ketika sebuah assignment dikunjungi, assignment itu ditambahkan ke mapping dalam status "undecided" (lihat komentar tentang for loop di bawah) dan setiap tugas lainnya ke variabel yang sama yang masih dalam status "undecided" diubah menjadi "unused".
Ketika sebuah variabel direferensikan, status penugasan apa pun ke variabel itu yang masih dalam status "undecided" diubah menjadi "used".

Pada titik di mana aliran kontrol terpecah, salinan mapping diserahkan ke setiap cabang. Pada titik di mana aliran kontrol bergabung, dua pemetaan yang berasal dari dua cabang digabungkan dengan cara berikut:
Pernyataan yang hanya dalam satu pemetaan atau memiliki status yang sama digunakan tidak berubah.
Nilai-nilai yang bertentangan diselesaikan dengan cara berikut:

- "unused", "undecided" -> "undecided"
- "unused", "used" -> "used"
- "undecided, "used" -> "used"

Untuk for-loop, condition, bodi dan post-part dikunjungi dua kali, dengan memperhitungkan aliran
kontrol penyambungan pada condition.
Dengan kata lain, kami membuat tiga jalur aliran kontrol: Zero run dari loop, satu run dan dua run
dan kemudian menggabungkannya di akhir.

Mensimulasikan putaran ketiga atau bahkan lebih tidak diperlukan, yang dapat dilihat sebagai berikut:

Keadaan penugasan pada awal iterasi secara deterministik akan menghasilkan keadaan penugasan tersebut
pada akhir iterasi.
Biarkan fungsi mapping keadaan ini disebut ``f``. Kombinasi dari tiga state berbeda ``unused``, ``undecided``
dan ``used`` seperti yang dijelaskan di atas adalah operasi ``max`` di mana ``unused = 0``, ``undecided = 1` ` dan ``digunakan = 2``.

Cara yang tepat adalah dengan menghitung

.. code-block:: none

    max(s, f(s), f(f(s)), f(f(f(s))), ...)

sebagai status setelah loop. Karena ``f`` hanya memiliki rentang tiga nilai yang berbeda,
iterasi itu harus mencapai siklus setelah paling banyak tiga iterasi,
dan dengan demikian ``f(f(f(s)))`` harus sama dengan salah satu dari ``s``, ``f(s)``, atau ``f(f(s))``
dan dengan demikian

.. code-block:: none

    max(s, f(s), f(f(s))) = max(s, f(s), f(f(s)), f(f(f(s))), ...).

Singkatnya, menjalankan loop paling banyak dua kali sudah cukup karena hanya ada tiga
state yang berbeda.

Untuk pernyataan switch yang memiliki kasus "default", tidak ada bagian aliran
kontrol yang melewatkan switch.

Ketika sebuah variabel keluar dari ruang lingkup, semua pernyataan masih dalam "undecided"
status diubah menjadi "unused", kecuali variabelnya adalah pengembalian
parameter suatu fungsi - di sana, statusnya berubah menjadi "used".

Dalam traversal kedua, semua penetapan yang berada dalam status "unused" dihapus.

Langkah ini biasanya dijalankan tepat setelah transformasi SSA selesai
generasi pseudo-SSA.

Tool
----

Movability
^^^^^^^^^^

Movability adalah properti dari sebuah ekspresi. Ini secara kasar berarti bahwa ekspresi
bebas efek samping dan evaluasinya hanya bergantung pada nilai variabel
dan call-constant state dari environment. Sebagian besar ekspresi dapat dipindahkan.
Bagian berikut membuat ekspresi tidak dapat dipindahkan:

- panggilan fungsi (mungkin santai di masa mendatang jika semua pernyataan dalam fungsi dapat dipindahkan)
- opcode yang (dapat) memiliki efek samping (seperti ``call`` atau ``selfdestruct``)
- opcode yang membaca atau menulis memori, penyimpanan, atau informasi status eksternal
- opcode yang bergantung pada PC saat ini, ukuran memori, atau ukuran data yang dikembalikan

DataflowAnalyzer
^^^^^^^^^^^^^^^^

Dataflow Analyzer bukanlah langkah pengoptimal itu sendiri tetapi digunakan sebagai
alat oleh komponen lain. Saat melintasi AST, ia melacak nilai saat ini dari setiap variabel,
selama nilai itu adalah ekspresi yang dapat dipindahkan. Ini merekam variabel yang merupakan
bagian dari ekspresi yang saat ini ditetapkan untuk satu sama lain variabel. Pada setiap penugasan
ke variabel ``a``, nilai tersimpan saat ini dari ``a`` diperbarui dan semua nilai tersimpan dari
semua variabel ``b`` dihapus setiap kali ``a`` adalah bagian dari yang saat ini disimpan ekspresi untuk ``b``.

Pada gabungan aliran kontrol, pengetahuan tentang variabel dihapus jika variabel tersebut telah
atau akan ditetapkan di salah satu jalur aliran kontrol. Misalnya, saat memasuki perulangan for,
semua variabel dihapus yang akan ditetapkan selama blok isi atau pos.

Expression-Scale Simplifications
--------------------------------

These simplification passes change expressions and replace them by equivalent
and hopefully simpler expressions.

.. _common-subexpression-eliminator:

CommonSubexpressionEliminator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Langkah ini menggunakan Dataflow Analyzer dan mengganti subekspresi yang secara sintaksis cocok
dengan nilai variabel saat ini dengan referensi ke variabel tersebut. Ini adalah transformasi ekuivalensi
karena subekspresi tersebut harus dapat dipindahkan.

Semua subekspresi yang merupakan pengidentifikasi itu sendiri diganti dengan nilainya saat ini jika
nilainya adalah pengidentifikasi.

Kombinasi kedua aturan di atas memungkinkan untuk menghitung penomoran nilai lokal, yang berarti
bahwa jika dua variabel memiliki nilai yang sama, salah satunya akan selalu tidak digunakan. Pemangkas
yang Tidak Digunakan atau
Redundant Assign Eliminator kemudian akan dapat sepenuhnya menghilangkan variabel tersebut.

Langkah ini sangat efisien jika pemisah ekspresi dijalankan sebelumnya. Jika kode dalam bentuk pseudo-SSA,
nilai-nilai variabel tersedia untuk waktu yang lebih lama dan dengan demikian kami memiliki peluang ekspresi yang lebih tinggi untuk dapat diganti.

Penyederhanaan ekspresi akan dapat melakukan penggantian yang lebih baik jika eliminator subekspresi umum dijalankan tepat sebelumnya.

.. _expression-simplifier:

Expression Simplifier
^^^^^^^^^^^^^^^^^^^^^

Expression Simplifier menggunakan Dataflow Analyzer dan menggunakan daftar transformasi
ekivalensi pada ekspresi seperti ``X + 0 -> X`` untuk menyederhanakan kode.

Ia mencoba mencocokkan pola seperti ``X + 0`` pada setiap subekspresi.
Selama prosedur pencocokan, ini menyelesaikan variabel ke ekspresi yang saat ini ditetapkan
untuk dapat mencocokkan pola bersarang lebih dalam bahkan ketika kode dalam bentuk pseudo-SSA.

Beberapa pola seperti ``X - X -> 0`` hanya dapat diterapkan selama ekspresi ``X`` dapat dipindahkan,
karena jika tidak, akan menghilangkan potensi efek sampingnya.
Karena referensi variabel selalu dapat dipindahkan, bahkan jika nilainya saat ini mungkin tidak,
Penyederhanaan Ekspresi sekali lagi lebih kuat dalam bentuk split atau pseudo-SSA.

.. _literal-rematerialiser:

LiteralRematerialiser
^^^^^^^^^^^^^^^^^^^^^

Untuk didokumentasikan.

.. _load-resolver:

LoadResolver
^^^^^^^^^^^^

Tahap pengoptimalan yang menggantikan ekspresi tipe ``sload(x)`` dan ``mload(x)``
dengan nilai yang saat ini disimpan dalam storage resp. memori, jika diketahui.

Bekerja paling baik jika kode dalam bentuk SSA.

Prasyarat: Disambiguator, ForLoopInitRewriter.

.. _reasoning-based-simplifier:

ReasoningBasedSimplifier
^^^^^^^^^^^^^^^^^^^^^^^^

Pengoptimal ini menggunakan pemecah SMT untuk memeriksa apakah kondisi ``if`` konstan.

- Jika `` constraints AND condition`` adalah UNSAT, kondisi tidak pernah benar dan seluruh tubuh dapat dihilangkan.
- Jika ``constraint AND NOT condition`` adalah UNSAT, kondisinya selalu benar dan dapat diganti dengan ``1``.

Penyederhanaan di atas hanya dapat diterapkan jika kondisinya bergerak.

Ini hanya efektif pada dialek EVM, tetapi aman digunakan pada dialek lain.

Prasyarat: Disambiguator, SSATransform.

Statement-Scale Simplifications
-------------------------------

.. _circular-reference-pruner:

CircularReferencesPruner
^^^^^^^^^^^^^^^^^^^^^^^^

This stage removes functions that call each other but are
neither externally referenced nor referenced from the outermost context.

.. _conditional-simplifier:

ConditionalSimplifier
^^^^^^^^^^^^^^^^^^^^^

Conditional Simplifier menyisipkan penetapan ke variabel kondisi jika nilainya dapat ditentukan
dari aliran kontrol.

Menghancurkan SSA form.

Saat ini, alat ini sangat terbatas, terutama karena kami belum memiliki dukungan
untuk tipe boolean. Karena kondisi hanya memeriksa ekspresi yang bukan nol,
kita tidak dapat menetapkan nilai tertentu.

Fitur saat ini:

- ganti kasus: masukkan "<kondisi> := <caseLabel>"
- setelah pernyataan if dengan penghentian aliran kontrol, masukkan "<condition> := 0"

Fitur masa depan:

- izinkan penggantian dengan "1"
- pertimbangkan penghentian fungsi yang ditentukan pengguna

Bekerja paling baik dengan formulir SSA dan jika penghapusan kode mati telah berjalan sebelumnya.

Prasyarat: Disambiguator.

.. _conditional-unsimplifier:

ConditionalUnsimplifier
^^^^^^^^^^^^^^^^^^^^^^^

Kebalikan dari Penyederhanaan Bersyarat.

.. _control-flow-simplifier:

ControlFlowSimplifier
^^^^^^^^^^^^^^^^^^^^^

Menyederhanakan beberapa struktur control-flow:

- ganti jika dengan badan kosong dengan pop(condition)
- hapus kotak switch default yang kosong
- hapus kotak switch kosong jika tidak ada case default
- ganti switch tanpa case dengan pop(expression)
- putar switch dengan case tunggal menjadi if
- ganti switch dengan hanya case default dengan pop(expression) dan body
- ganti switch dengan const expr dengan bodi case yang cocok
- ganti ``for`` dengan menghentikan aliran kontrol dan tanpa pemutusan/lanjutan lainnya dengan ``if``
- hapus ``leave`` di akhir fungsi.

Tak satu pun dari operasi ini bergantung pada aliran data. StructuralSimplifier melakukan tugas serupa yang bergantung pada aliran data.

ControlFlowSimplifier merekam ada atau tidaknya ``break``
dan pernyataan ``continue`` selama traversalnya.

Prasyarat: Disambiguator, FunctionHoister, ForLoopInitRewriter.
Penting: Memperkenalkan opcode EVM dan dengan demikian hanya dapat digunakan pada kode EVM untuk saat ini.

.. _dead-code-eliminator:

DeadCodeEliminator
^^^^^^^^^^^^^^^^^^

Tahap pengoptimalan ini menghapus kode yang tidak dapat dijangkau.

Kode yang tidak dapat dijangkau adalah kode apa pun dalam blok yang didahului oleh sebuah
leave, return, invalid, break, continue, selfdestruct atau revert.

Definisi fungsi dipertahankan seperti yang mungkin dipanggil oleh kode
sebelumnya dan dengan demikian dianggap dapat dijangkau.

Karena variabel yang dideklarasikan dalam blok init for loop memiliki cakupan yang diperluas ke badan loop,
kami membutuhkan ForLoopInitRewriter untuk dijalankan sebelum langkah ini.

Prasyarat: ForLoopInitRewriter, Function Hoister, Function Grouper

.. _unused-pruner:

UnusedPruner
^^^^^^^^^^^^

Langkah ini menghapus definisi semua fungsi yang tidak pernah direferensikan.

Itu juga menghapus deklarasi variabel yang tidak pernah direferensikan.
Jika deklarasi memberikan nilai yang tidak dapat dipindahkan, ekspresi dipertahankan,
tetapi nilainya dibuang.

Semua pernyataan ekspresi bergerak (ekspresi yang tidak ditetapkan) dihapus.

.. _structural-simplifier:

StructuralSimplifier
^^^^^^^^^^^^^^^^^^^^

Ini adalah langkah umum yang melakukan berbagai macam penyederhanaan pada
tingkat struktural:

- ganti pernyataan if dengan badan kosong dengan ``pop(condition)``
- ganti jika pernyataan dengan kondisi benar oleh tubuhnya
- hapus pernyataan if dengan kondisi salah
- putar switch dengan kasing tunggal menjadi if
- ganti switch dengan hanya case default dengan ``pop(expression)`` dan body
- ganti switch dengan ekspresi literal dengan mencocokkan badan case
- ganti for loop dengan kondisi false dengan bagian inisialisasinya

Komponen ini menggunakan Dataflow Analyzer.

.. _block-flattener:

BlockFlattener
^^^^^^^^^^^^^^

Tahap ini menghilangkan nested block dengan memasukkan pernyataan di
inner block pada tempat yang sesuai di outer block:

.. code-block:: yul

    {
        let x := 2
        {
            let y := 3
            mstore(x, y)
        }
    }

diubah menjadi

.. code-block:: yul

    {
        let x := 2
        let y := 3
        mstore(x, y)
    }

Selama kode disamarkan, ini tidak menimbulkan masalah karena
cakupan variabel hanya bisa tumbuh.

.. _loop-invariant-code-motion:

LoopInvariantCodeMotion
^^^^^^^^^^^^^^^^^^^^^^^
Pengoptimalan ini memindahkan deklarasi variabel SSA yang dapat dipindahkan ke luar loop.

Hanya pernyataan di tingkat atas dalam badan perulangan atau blok pos yang dipertimbangkan, yaitu deklarasi variabel di dalam cabang bersyarat tidak akan dipindahkan keluar dari perulangan.

Persyaratan:

- Disambiguator, ForLoopInitRewriter dan FunctionHoister harus dijalankan terlebih dahulu.
- Pemisah ekspresi dan transformasi SSA harus dijalankan terlebih dahulu untuk mendapatkan hasil yang lebih baik.


Function-Level Optimizations
----------------------------

.. _function-specializer:

FunctionSpecializer
^^^^^^^^^^^^^^^^^^^

Langkah ini mengkhususkan fungsi dengan argumen literalnya.

Jika suatu fungsi, katakanlah, ``fungsi f(a, b) { sstore (a, b) }``, dipanggil dengan argumen literal, untuk
contoh, ``f(x, 5)``, di mana ``x`` adalah pengidentifikasi, itu bisa dispesialisasikan dengan membuat yang baru
fungsi ``f_1`` yang hanya membutuhkan satu argumen, yaitu,

.. code-block:: yul

    function f_1(a_1) {
        let b_1 := 5
        sstore(a_1, b_1)
    }

Langkah pengoptimalan lainnya akan dapat membuat lebih banyak penyederhanaan fungsi. Langkah
pengoptimalan terutama berguna untuk fungsi yang tidak akan digariskan.

Prasyarat: Disambiguator, FunctionHoister

LiteralRematerialiser direkomendasikan sebagai prasyarat, meskipun tidak diperlukan untuk
ketepatan.

.. _unused-function-parameter-pruner:

UnusedFunctionParameterPruner
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Langkah ini menghapus parameter yang tidak digunakan dalam suatu fungsi.

Jika parameter tidak digunakan, seperti ``c`` dan ``y`` di, ``fungsi f(a,b,c) -> x, y { x := div(a,b) }``, kami
hapus parameter dan buat fungsi "tautan" baru sebagai berikut:

.. code-block:: yul

    function f(a,b) -> x { x := div(a,b) }
    function f2(a,b,c) -> x, y { x := f(a,b) }

dan ganti semua referensi ke ``f`` dengan ``f2``.
Inliner harus dijalankan setelahnya untuk memastikan bahwa semua referensi ke ``f2`` diganti oleh
``f``.

Prasyarat: Disambiguator, FunctionHoister, LiteralRematerialiser.

Langkah LiteralRematerialiser tidak diperlukan untuk kebenaran. Ini membantu menangani kasus-kasus seperti:
``function f(x) -> y { revert(y, y} }`` di mana literal ``y`` akan diganti dengan nilainya ``0``,
memungkinkan kita untuk menulis ulang fungsi.

.. _equivalent-function-combiner:

EquivalentFunctionCombiner
^^^^^^^^^^^^^^^^^^^^^^^^^^

Jika dua fungsi secara sintaksis setara, sementara memungkinkan variabel
mengganti nama tetapi tidak memesan ulang, maka referensi apa pun ke salah satu dari
fungsi digantikan oleh yang lain.

Penghapusan fungsi yang sebenarnya dilakukan oleh Pemangkas yang Tidak Digunakan.


Fungsi Inlining
---------------

.. _expression-inliner:

ExpressionInliner
^^^^^^^^^^^^^^^^^

Komponen pengoptimal ini melakukan inlining fungsi terbatas dengan inlining fungsi yang dapat
sebaris di dalam ekspresi fungsional, yaitu fungsi yang:

- mengembalikan nilai tunggal.
- memiliki body seperti ``r := <functional expression>``.
- tidak merujuk diri mereka sendiri atau ``r`` di sisi kanan.

Selanjutnya, untuk semua parameter, semua hal berikut harus benar:

- Argumennya bisa dipindahkan.
- Parameter direferensikan kurang dari dua kali di badan fungsi, atau argumennya agak murah
  ("biaya" paling banyak 1, seperti konstanta hingga 0xff).

Contoh: Fungsi yang akan digarisbawahi berbentuk ``fungsi f(...) -> r { r := E }`` dimana
``E`` adalah ekspresi yang tidak mereferensikan ``r`` dan semua argumen dalam pemanggilan fungsi adalah ekspresi yang dapat dipindahkan.

Hasil dari inlining ini selalu berupa ekspresi tunggal.

Komponen ini hanya dapat digunakan pada sumber dengan nama unik.

.. _full-inliner:

FullInliner
^^^^^^^^^^^

Full Inliner menggantikan panggilan tertentu dari fungsi tertentu
oleh tubuh fungsi. Ini tidak terlalu membantu dalam banyak kasus, karena
hanya meningkatkan ukuran kode tetapi tidak memiliki manfaat. Selain itu,
kode biasanya sangat mahal dan kita sering lebih suka memiliki kode yang
lebih pendek daripada kode yang lebih efisien. Namun, dalam kasus yang sama,
penyejajaran fungsi dapat memiliki efek positif pada langkah pengoptimal
berikutnya. Ini adalah kasus jika salah satu argumen fungsi adalah konstanta, misalnya.

Selama inlining, heuristik digunakan untuk mengetahui apakah fungsi memanggil
harus digarisbawahi atau tidak.
Heuristik saat ini tidak sejajar dengan fungsi "large" kecuali
fungsi yang dipanggil kecil. Fungsi yang hanya digunakan sekali adalah inline,
serta fungsi berukuran sedang, sedangkan pemanggilan fungsi dengan argumen konstan
memungkinkan fungsi yang sedikit lebih besar.


Di masa mendatang, kami mungkin menyertakan komponen lacak balik yang,
alih-alih langsung menyejajarkan fungsi, hanya mengkhususkannya,
yang berarti bahwa salinan fungsi dihasilkan di mana
parameter tertentu selalu diganti dengan konstanta. Setelah itu,
kita dapat menjalankan pengoptimal pada fungsi khusus ini. Jika
menghasilkan keuntungan besar, fungsi khusus dipertahankan,
jika tidak, fungsi asli digunakan sebagai gantinya.

Cleanup
-------

Pembersihan dilakukan pada akhir menjalankan pengoptimal. Ini mencoba
untuk menggabungkan ekspresi terpisah menjadi ekspresi yang sangat *nested* lagi dan juga
meningkatkan "compilability" untuk mesin stack dengan menghilangkan
variabel sebanyak mungkin.

.. _expression-joiner:

ExpressionJoiner
^^^^^^^^^^^^^^^^

Ini adalah operasi kebalikan dari pemisah ekspresi. Itu mengubah urutan deklarasi
variabel yang memiliki tepat satu referensi menjadi ekspresi kompleks.
Tahap ini sepenuhnya mempertahankan urutan panggilan fungsi dan eksekusi opcode.
Itu tidak menggunakan informasi apa pun mengenai komutatifitas opcode;
jika memindahkan nilai variabel ke tempat penggunaannya akan mengubah urutannya
dari setiap panggilan fungsi atau eksekusi opcode, transformasi tidak dilakukan.

Perhatikan bahwa komponen tidak akan memindahkan nilai yang ditetapkan dari assignment variabel
atau variabel yang direferensikan lebih dari satu kali.

Cuplikan ``let x := add(0, 2) let y := mul(x, mload(2))`` tidak ditransformasi,
karena akan menyebabkan urutan panggilan ke opcodes ``add`` dan
``mload`` untuk ditukar - meskipun ini tidak akan membuat perbedaan
karena ``add`` dapat dipindahkan.

Saat menyusun ulang opcode seperti itu, referensi variabel dan literal diabaikan.
Karena itu, cuplikan ``let x := add(0, 2) let y := mul(x, 3)`` adalah
ditransformasikan ke ``biarkan y := mul(add(0, 2), 3)``, meskipun opcode ``add``
akan dieksekusi setelah evaluasi literal ``3``.

.. _SSA-reverser:

SSAReverser
^^^^^^^^^^^

Ini adalah langkah kecil yang membantu membalikkan efek transformasi SSA
jika digabungkan dengan Common Subexpression Eliminator dan
Pemangkas yang tidak digunakan.

Formulir SSA yang kami hasilkan merusak pembuatan kode pada EVM dan
WebAssembly sama karena menghasilkan banyak variabel lokal. Itu akan
lebih baik hanya menggunakan kembali variabel yang ada dengan tugas daripada
deklarasi variabel baru.

Transformasi SSA menulis ulang

.. code-block:: yul

    let a := calldataload(0)
    mstore(a, 1)

ke

.. code-block:: yul

    let a_1 := calldataload(0)
    let a := a_1
    mstore(a_1, 1)
    let a_2 := calldataload(0x20)
    a := a_2

Masalahnya adalah sebagai ganti ``a``, variabel ``a_1`` digunakan
setiap kali ``a`` dirujuk. Pernyataan perubahan transformasi SSA
formulir ini hanya dengan menukar deklarasi dan penugasan. Di atas
cuplikan berubah menjadi

.. code-block:: yul

    let a := calldataload(0)
    let a_1 := a
    mstore(a_1, 1)
    a := calldataload(0x20)
    let a_2 := a

Ini adalah transformasi ekuivalensi yang sangat sederhana, tetapi saat kita menjalankan
Common Subexpression Eliminator, akan menggantikan semua kemunculan ``a_1``
oleh ``a`` (sampai ``a`` ditetapkan ulang). Pemangkas yang Tidak Digunakan kemudian akan
hilangkan variabel ``a_1`` sama sekali dan dengan demikian membalikkan sepenuhnya
transformasi SSA.

.. _stack-compressor:

StackCompressor
^^^^^^^^^^^^^^^

Satu masalah yang membuat pembuatan kode untuk Mesin Virtual Ethereum
sulit adalah kenyataan bahwa ada batas keras 16 slot untuk dicapai
ke bawah tumpukan ekspresi. Ini kurang lebih diterjemahkan menjadi batas
dari 16 variabel lokal. Kompresor tumpukan mengambil kode Yul dan
mengkompilasinya ke bytecode EVM. Kapan pun perbedaan tumpukan terlalu
besar, ini merekam fungsi tempat ini terjadi.

Untuk setiap fungsi yang menyebabkan masalah seperti itu, Rematerialiser
dipanggil dengan permintaan khusus untuk menghilangkan secara agresif
variabel diurutkan berdasarkan biaya nilainya.

Pada kegagalan, prosedur ini diulang beberapa kali.

.. _rematerialiser:

Rematerialiser
^^^^^^^^^^^^^^

Tahap rematerialisasi mencoba mengganti referensi variabel dengan ekspresi bahwa
terakhir ditugaskan ke variabel. Ini tentu saja hanya bermanfaat jika ungkapan ini
relatif murah untuk dievaluasi. Lebih jauh, itu hanya setara secara semantik jika
nilai ekspresi tidak berubah antara titik penugasan dan
titik penggunaan. Manfaat utama dari tahap ini adalah dapat menghemat slot tumpukan jika
mengarah ke variabel yang dihilangkan sepenuhnya (lihat di bawah), tetapi juga dapat
simpan opcode DUP pada EVM jika ekspresinya sangat murah.

Rematerialiser menggunakan Dataflow Analyzer untuk melacak nilai variabel saat ini,
yang selalu bergerak.
Jika nilainya sangat murah atau variabel tersebut secara eksplisit diminta untuk dihilangkan,
referensi variabel diganti dengan nilai saat ini.

.. _for-loop-condition-out-of-body:

ForLoopConditionOutOfBody
^^^^^^^^^^^^^^^^^^^^^^^^^

Membalikkan transformasi ForLoopConditionIntoBody.

Untuk setiap ``c`` bergerak, ternyata

.. code-block:: none

    for { ... } 1 { ... } {
    if iszero(c) { break }
    ...
    }

ke dalam

.. code-block:: none

    for { ... } c { ... } {
    ...
    }

dan ternyata

.. code-block:: none

    for { ... } 1 { ... } {
    if c { break }
    ...
    }

ke dalam

.. code-block:: none

    for { ... } iszero(c) { ... } {
    ...
    }

LiteralRematerialiser harus dijalankan sebelum langkah ini.


WebAssembly specific
--------------------

MainFunction
^^^^^^^^^^^^

Mengubah blok paling atas menjadi fungsi dengan nama tertentu ("main") yang tidak memiliki
input maupun output.

Tergantung pada Fungsi Kerapu.
=======
.. index:: optimizer, optimiser, common subexpression elimination, constant propagation
.. _optimizer:

*************
The Optimizer
*************

The Solidity compiler uses two different optimizer modules: The "old" optimizer
that operates at the opcode level and the "new" optimizer that operates on Yul IR code.

The opcode-based optimizer applies a set of `simplification rules <https://github.com/ethereum/solidity/blob/develop/libevmasm/RuleList.h>`_
to opcodes. It also combines equal code sets and removes unused code.

The Yul-based optimizer is much more powerful, because it can work across function
calls. For example, arbitrary jumps are not possible in Yul, so it is
possible to compute the side-effects of each function. Consider two function calls,
where the first does not modify storage and the second does modify storage.
If their arguments and return values do not depend on each other, we can reorder
the function calls. Similarly, if a function is
side-effect free and its result is multiplied by zero, you can remove the function
call completely.

Currently, the parameter ``--optimize`` activates the opcode-based optimizer for the
generated bytecode and the Yul optimizer for the Yul code generated internally, for example for ABI coder v2.
One can use ``solc --ir-optimized --optimize`` to produce an
optimized Yul IR for a Solidity source. Similarly, one can use ``solc --strict-assembly --optimize``
for a stand-alone Yul mode.

You can find more details on both optimizer modules and their optimization steps below.

Benefits of Optimizing Solidity Code
====================================

Overall, the optimizer tries to simplify complicated expressions, which reduces both code
size and execution cost, i.e., it can reduce gas needed for contract deployment as well as for external calls made to the contract.
It also specializes or inlines functions. Especially
function inlining is an operation that can cause much bigger code, but it is
often done because it results in opportunities for more simplifications.


Differences between Optimized and Non-Optimized Code
====================================================

Generally, the most visible difference is that constant expressions are evaluated at compile time.
When it comes to the ASM output, one can also notice a reduction of equivalent or duplicate
code blocks (compare the output of the flags ``--asm`` and ``--asm --optimize``). However,
when it comes to the Yul/intermediate-representation, there can be significant
differences, for example, functions may be inlined, combined, or rewritten to eliminate
redundancies, etc. (compare the output between the flags ``--ir`` and
``--optimize --ir-optimized``).

.. _optimizer-parameter-runs:

Optimizer Parameter Runs
========================

The number of runs (``--optimize-runs``) specifies roughly how often each opcode of the
deployed code will be executed across the life-time of the contract. This means it is a
trade-off parameter between code size (deploy cost) and code execution cost (cost after deployment).
A "runs" parameter of "1" will produce short but expensive code. In contrast, a larger "runs"
parameter will produce longer but more gas efficient code. The maximum value of the parameter
is ``2**32-1``.

.. note::

    A common misconception is that this parameter specifies the number of iterations of the optimizer.
    This is not true: The optimizer will always run as many times as it can still improve the code.

Opcode-Based Optimizer Module
=============================

The opcode-based optimizer module operates on assembly code. It splits the
sequence of instructions into basic blocks at ``JUMPs`` and ``JUMPDESTs``.
Inside these blocks, the optimizer analyzes the instructions and records every modification to the stack,
memory, or storage as an expression which consists of an instruction and
a list of arguments which are pointers to other expressions.

Additionally, the opcode-based optimizer
uses a component called "CommonSubexpressionEliminator" that, amongst other
tasks, finds expressions that are always equal (on every input) and combines
them into an expression class. It first tries to find each new
expression in a list of already known expressions. If no such matches are found,
it simplifies the expression according to rules like
``constant + constant = sum_of_constants`` or ``X * 1 = X``. Since this is
a recursive process, we can also apply the latter rule if the second factor
is a more complex expression which we know always evaluates to one.

Certain optimizer steps symbolically track the storage and memory locations. For example, this
information is used to compute Keccak-256 hashes that can be evaluated during compile time. Consider
the sequence:

.. code-block:: none

    PUSH 32
    PUSH 0
    CALLDATALOAD
    PUSH 100
    DUP2
    MSTORE
    KECCAK256

or the equivalent Yul

.. code-block:: yul

    let x := calldataload(0)
    mstore(x, 100)
    let value := keccak256(x, 32)

In this case, the optimizer tracks the value at a memory location ``calldataload(0)`` and then
realizes that the Keccak-256 hash can be evaluated at compile time. This only works if there is no
other instruction that modifies memory between the ``mstore`` and ``keccak256``. So if there is an
instruction that writes to memory (or storage), then we need to erase the knowledge of the current
memory (or storage). There is, however, an exception to this erasing, when we can easily see that
the instruction doesn't write to a certain location.

For example,

.. code-block:: yul

    let x := calldataload(0)
    mstore(x, 100)
    // Current knowledge memory location x -> 100
    let y := add(x, 32)
    // Does not clear the knowledge that x -> 100, since y does not write to [x, x + 32)
    mstore(y, 200)
    // This Keccak-256 can now be evaluated
    let value := keccak256(x, 32)

Therefore, modifications to storage and memory locations, of say location ``l``, must erase
knowledge about storage or memory locations which may be equal to ``l``. More specifically, for
storage, the optimizer has to erase all knowledge of symbolic locations, that may be equal to ``l``
and for memory, the optimizer has to erase all knowledge of symbolic locations that may not be at
least 32 bytes away. If ``m`` denotes an arbitrary location, then this decision on erasure is done
by computing the value ``sub(l, m)``. For storage, if this value evaluates to a literal that is
non-zero, then the knowledge about ``m`` will be kept. For memory, if the value evaluates to a
literal that is between ``32`` and ``2**256 - 32``, then the knowledge about ``m`` will be kept. In
all other cases, the knowledge about ``m`` will be erased.

After this process, we know which expressions have to be on the stack at
the end, and have a list of modifications to memory and storage. This information
is stored together with the basic blocks and is used to link them. Furthermore,
knowledge about the stack, storage and memory configuration is forwarded to
the next block(s).

If we know the targets of all ``JUMP`` and ``JUMPI`` instructions,
we can build a complete control flow graph of the program. If there is only
one target we do not know (this can happen as in principle, jump targets can
be computed from inputs), we have to erase all knowledge about the input state
of a block as it can be the target of the unknown ``JUMP``. If the opcode-based
optimizer module finds a ``JUMPI`` whose condition evaluates to a constant, it transforms it
to an unconditional jump.

As the last step, the code in each block is re-generated. The optimizer creates
a dependency graph from the expressions on the stack at the end of the block,
and it drops every operation that is not part of this graph. It generates code
that applies the modifications to memory and storage in the order they were
made in the original code (dropping modifications which were found not to be
needed). Finally, it generates all values that are required to be on the
stack in the correct place.

These steps are applied to each basic block and the newly generated code
is used as replacement if it is smaller. If a basic block is split at a
``JUMPI`` and during the analysis, the condition evaluates to a constant,
the ``JUMPI`` is replaced based on the value of the constant. Thus code like

.. code-block:: solidity

    uint x = 7;
    data[7] = 9;
    if (data[x] != x + 2) // this condition is never true
      return 2;
    else
      return 1;

simplifies to this:

.. code-block:: solidity

    data[7] = 9;
    return 1;

Simple Inlining
---------------

Since Solidity version 0.8.2, there is another optimizer step that replaces certain
jumps to blocks containing "simple" instructions ending with a "jump" by a copy of these instructions.
This corresponds to inlining of simple, small Solidity or Yul functions. In particular, the sequence
``PUSHTAG(tag) JUMP`` may be replaced, whenever the ``JUMP`` is marked as jump "into" a
function and behind ``tag`` there is a basic block (as described above for the
"CommonSubexpressionEliminator") that ends in another ``JUMP`` which is marked as a jump
"out of" a function.

In particular, consider the following prototypical example of assembly generated for a
call to an internal Solidity function:

.. code-block:: text

      tag_return
      tag_f
      jump      // in
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

As long as the body of the function is a continuous basic block, the "Inliner" can replace ``tag_f jump`` by
the block at ``tag_f`` resulting in:

.. code-block:: text

      tag_return
      ...body of function f...
      jump
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

Now ideally, the other optimizer steps described above will result in the return tag push being moved
towards the remaining jump resulting in:

.. code-block:: text

      ...body of function f...
      tag_return
      jump
    tag_return:
      ...opcodes after call to f...

    tag_f:
      ...body of function f...
      jump      // out

In this situation the "PeepholeOptimizer" will remove the return jump. Ideally, all of this can be done
for all references to ``tag_f`` leaving it unused, s.t. it can be removed, yielding:

.. code-block:: text

    ...body of function f...
    ...opcodes after call to f...

So the call to function ``f`` is inlined and the original definition of ``f`` can be removed.

Inlining like this is attempted, whenever a heuristics suggests that inlining is cheaper over the lifetime of a
contract than not inlining. This heuristics depends on the size of the function body, the
number of other references to its tag (approximating the number of calls to the function) and
the expected number of executions of the contract (the global optimizer parameter "runs").


Yul-Based Optimizer Module
==========================

The Yul-based optimizer consists of several stages and components that all transform
the AST in a semantically equivalent way. The goal is to end up either with code
that is shorter or at least only marginally longer but will allow further
optimization steps.

.. warning::

    Since the optimizer is under heavy development, the information here might be outdated.
    If you rely on a certain functionality, please reach out to the team directly.

The optimizer currently follows a purely greedy strategy and does not do any
backtracking.

All components of the Yul-based optimizer module are explained below.
The following transformation steps are the main components:

- SSA Transform
- Common Subexpression Eliminator
- Expression Simplifier
- Redundant Assign Eliminator
- Full Inliner

Optimizer Steps
---------------

This is a list of all steps the Yul-based optimizer sorted alphabetically. You can find more information
on the individual steps and their sequence below.

- :ref:`block-flattener`.
- :ref:`circular-reference-pruner`.
- :ref:`common-subexpression-eliminator`.
- :ref:`conditional-simplifier`.
- :ref:`conditional-unsimplifier`.
- :ref:`control-flow-simplifier`.
- :ref:`dead-code-eliminator`.
- :ref:`equal-store-eliminator`.
- :ref:`equivalent-function-combiner`.
- :ref:`expression-joiner`.
- :ref:`expression-simplifier`.
- :ref:`expression-splitter`.
- :ref:`for-loop-condition-into-body`.
- :ref:`for-loop-condition-out-of-body`.
- :ref:`for-loop-init-rewriter`.
- :ref:`expression-inliner`.
- :ref:`full-inliner`.
- :ref:`function-grouper`.
- :ref:`function-hoister`.
- :ref:`function-specializer`.
- :ref:`literal-rematerialiser`.
- :ref:`load-resolver`.
- :ref:`loop-invariant-code-motion`.
- :ref:`redundant-assign-eliminator`.
- :ref:`reasoning-based-simplifier`.
- :ref:`rematerialiser`.
- :ref:`SSA-reverser`.
- :ref:`SSA-transform`.
- :ref:`structural-simplifier`.
- :ref:`unused-function-parameter-pruner`.
- :ref:`unused-pruner`.
- :ref:`var-decl-initializer`.

Selecting Optimizations
-----------------------

By default the optimizer applies its predefined sequence of optimization steps to
the generated assembly. You can override this sequence and supply your own using
the ``--yul-optimizations`` option:

.. code-block:: bash

    solc --optimize --ir-optimized --yul-optimizations 'dhfoD[xarrscLMcCTU]uljmul'

The sequence inside ``[...]`` will be applied multiple times in a loop until the Yul code
remains unchanged or until the maximum number of rounds (currently 12) has been reached.

Available abbreviations are listed in the `Yul optimizer docs <yul.rst#optimization-step-sequence>`_.

Preprocessing
-------------

The preprocessing components perform transformations to get the program
into a certain normal form that is easier to work with. This normal
form is kept during the rest of the optimization process.

.. _disambiguator:

Disambiguator
^^^^^^^^^^^^^

The disambiguator takes an AST and returns a fresh copy where all identifiers have
unique names in the input AST. This is a prerequisite for all other optimizer stages.
One of the benefits is that identifier lookup does not need to take scopes into account
which simplifies the analysis needed for other steps.

All subsequent stages have the property that all names stay unique. This means if
a new identifier needs to be introduced, a new unique name is generated.

.. _function-hoister:

FunctionHoister
^^^^^^^^^^^^^^^

The function hoister moves all function definitions to the end of the topmost block. This is
a semantically equivalent transformation as long as it is performed after the
disambiguation stage. The reason is that moving a definition to a higher-level block cannot decrease
its visibility and it is impossible to reference variables defined in a different function.

The benefit of this stage is that function definitions can be looked up more easily
and functions can be optimized in isolation without having to traverse the AST completely.

.. _function-grouper:

FunctionGrouper
^^^^^^^^^^^^^^^

The function grouper has to be applied after the disambiguator and the function hoister.
Its effect is that all topmost elements that are not function definitions are moved
into a single block which is the first statement of the root block.

After this step, a program has the following normal form:

.. code-block:: text

    { I F... }

Where ``I`` is a (potentially empty) block that does not contain any function definitions (not even recursively)
and ``F`` is a list of function definitions such that no function contains a function definition.

The benefit of this stage is that we always know where the list of function begins.

.. _for-loop-condition-into-body:

ForLoopConditionIntoBody
^^^^^^^^^^^^^^^^^^^^^^^^

This transformation moves the loop-iteration condition of a for-loop into loop body.
We need this transformation because :ref:`expression-splitter` will not
apply to iteration condition expressions (the ``C`` in the following example).

.. code-block:: text

    for { Init... } C { Post... } {
        Body...
    }

is transformed to

.. code-block:: text

    for { Init... } 1 { Post... } {
        if iszero(C) { break }
        Body...
    }

This transformation can also be useful when paired with ``LoopInvariantCodeMotion``, since
invariants in the loop-invariant conditions can then be taken outside the loop.

.. _for-loop-init-rewriter:

ForLoopInitRewriter
^^^^^^^^^^^^^^^^^^^

This transformation moves the initialization part of a for-loop to before
the loop:

.. code-block:: text

    for { Init... } C { Post... } {
        Body...
    }

is transformed to

.. code-block:: text

    Init...
    for {} C { Post... } {
        Body...
    }

This eases the rest of the optimization process because we can ignore
the complicated scoping rules of the for loop initialisation block.

.. _var-decl-initializer:

VarDeclInitializer
^^^^^^^^^^^^^^^^^^
This step rewrites variable declarations so that all of them are initialized.
Declarations like ``let x, y`` are split into multiple declaration statements.

Only supports initializing with the zero literal for now.

Pseudo-SSA Transformation
-------------------------

The purpose of this components is to get the program into a longer form,
so that other components can more easily work with it. The final representation
will be similar to a static-single-assignment (SSA) form, with the difference
that it does not make use of explicit "phi" functions which combines the values
from different branches of control flow because such a feature does not exist
in the Yul language. Instead, when control flow merges, if a variable is re-assigned
in one of the branches, a new SSA variable is declared to hold its current value,
so that the following expressions still only need to reference SSA variables.

An example transformation is the following:

.. code-block:: yul

    {
        let a := calldataload(0)
        let b := calldataload(0x20)
        if gt(a, 0) {
            b := mul(b, 0x20)
        }
        a := add(a, 1)
        sstore(a, add(b, 0x20))
    }


When all the following transformation steps are applied, the program will look
as follows:

.. code-block:: yul

    {
        let _1 := 0
        let a_9 := calldataload(_1)
        let a := a_9
        let _2 := 0x20
        let b_10 := calldataload(_2)
        let b := b_10
        let _3 := 0
        let _4 := gt(a_9, _3)
        if _4
        {
            let _5 := 0x20
            let b_11 := mul(b_10, _5)
            b := b_11
        }
        let b_12 := b
        let _6 := 1
        let a_13 := add(a_9, _6)
        let _7 := 0x20
        let _8 := add(b_12, _7)
        sstore(a_13, _8)
    }

Note that the only variable that is re-assigned in this snippet is ``b``.
This re-assignment cannot be avoided because ``b`` has different values
depending on the control flow. All other variables never change their
value once they are defined. The advantage of this property is that
variables can be freely moved around and references to them
can be exchanged by their initial value (and vice-versa),
as long as these values are still valid in the new context.

Of course, the code here is far from being optimized. To the contrary, it is much
longer. The hope is that this code will be easier to work with and furthermore,
there are optimizer steps that undo these changes and make the code more
compact again at the end.

.. _expression-splitter:

ExpressionSplitter
^^^^^^^^^^^^^^^^^^

The expression splitter turns expressions like ``add(mload(0x123), mul(mload(0x456), 0x20))``
into a sequence of declarations of unique variables that are assigned sub-expressions
of that expression so that each function call has only variables
as arguments.

The above would be transformed into

.. code-block:: yul

    {
        let _1 := 0x20
        let _2 := 0x456
        let _3 := mload(_2)
        let _4 := mul(_3, _1)
        let _5 := 0x123
        let _6 := mload(_5)
        let z := add(_6, _4)
    }

Note that this transformation does not change the order of opcodes or function calls.

It is not applied to loop iteration-condition, because the loop control flow does not allow
this "outlining" of the inner expressions in all cases. We can sidestep this limitation by applying
:ref:`for-loop-condition-into-body` to move the iteration condition into loop body.

The final program should be in a form such that (with the exception of loop conditions)
function calls cannot appear nested inside expressions
and all function call arguments have to be variables.

The benefits of this form are that it is much easier to re-order the sequence of opcodes
and it is also easier to perform function call inlining. Furthermore, it is simpler
to replace individual parts of expressions or re-organize the "expression tree".
The drawback is that such code is much harder to read for humans.

.. _SSA-transform:

SSATransform
^^^^^^^^^^^^

This stage tries to replace repeated assignments to
existing variables by declarations of new variables as much as
possible.
The reassignments are still there, but all references to the
reassigned variables are replaced by the newly declared variables.

Example:

.. code-block:: yul

    {
        let a := 1
        mstore(a, 2)
        a := 3
    }

is transformed to

.. code-block:: yul

    {
        let a_1 := 1
        let a := a_1
        mstore(a_1, 2)
        let a_3 := 3
        a := a_3
    }

Exact semantics:

For any variable ``a`` that is assigned to somewhere in the code
(variables that are declared with value and never re-assigned
are not modified) perform the following transforms:

- replace ``let a := v`` by ``let a_i := v   let a := a_i``
- replace ``a := v`` by ``let a_i := v   a := a_i`` where ``i`` is a number such that ``a_i`` is yet unused.

Furthermore, always record the current value of ``i`` used for ``a`` and replace each
reference to ``a`` by ``a_i``.
The current value mapping is cleared for a variable ``a`` at the end of each block
in which it was assigned to and at the end of the for loop init block if it is assigned
inside the for loop body or post block.
If a variable's value is cleared according to the rule above and the variable is declared outside
the block, a new SSA variable will be created at the location where control flow joins,
this includes the beginning of loop post/body block and the location right after
If/Switch/ForLoop/Block statement.

After this stage, the Redundant Assign Eliminator is recommended to remove the unnecessary
intermediate assignments.

This stage provides best results if the Expression Splitter and the Common Subexpression Eliminator
are run right before it, because then it does not generate excessive amounts of variables.
On the other hand, the Common Subexpression Eliminator could be more efficient if run after the
SSA transform.

.. _redundant-assign-eliminator:

RedundantAssignEliminator
^^^^^^^^^^^^^^^^^^^^^^^^^

The SSA transform always generates an assignment of the form ``a := a_i``, even though
these might be unnecessary in many cases, like the following example:

.. code-block:: yul

    {
        let a := 1
        a := mload(a)
        a := sload(a)
        sstore(a, 1)
    }

The SSA transform converts this snippet to the following:

.. code-block:: yul

    {
        let a_1 := 1
        let a := a_1
        let a_2 := mload(a_1)
        a := a_2
        let a_3 := sload(a_2)
        a := a_3
        sstore(a_3, 1)
    }

The Redundant Assign Eliminator removes all the three assignments to ``a``, because
the value of ``a`` is not used and thus turn this
snippet into strict SSA form:

.. code-block:: yul

    {
        let a_1 := 1
        let a_2 := mload(a_1)
        let a_3 := sload(a_2)
        sstore(a_3, 1)
    }

Of course the intricate parts of determining whether an assignment is redundant or not
are connected to joining control flow.

The component works as follows in detail:

The AST is traversed twice: in an information gathering step and in the
actual removal step. During information gathering, we maintain a
mapping from assignment statements to the three states
"unused", "undecided" and "used" which signifies whether the assigned
value will be used later by a reference to the variable.

When an assignment is visited, it is added to the mapping in the "undecided" state
(see remark about for loops below) and every other assignment to the same variable
that is still in the "undecided" state is changed to "unused".
When a variable is referenced, the state of any assignment to that variable still
in the "undecided" state is changed to "used".

At points where control flow splits, a copy
of the mapping is handed over to each branch. At points where control flow
joins, the two mappings coming from the two branches are combined in the following way:
Statements that are only in one mapping or have the same state are used unchanged.
Conflicting values are resolved in the following way:

- "unused", "undecided" -> "undecided"
- "unused", "used" -> "used"
- "undecided, "used" -> "used"

For for-loops, the condition, body and post-part are visited twice, taking
the joining control-flow at the condition into account.
In other words, we create three control flow paths: Zero runs of the loop,
one run and two runs and then combine them at the end.

Simulating a third run or even more is unnecessary, which can be seen as follows:

A state of an assignment at the beginning of the iteration will deterministically
result in a state of that assignment at the end of the iteration. Let this
state mapping function be called ``f``. The combination of the three different
states ``unused``, ``undecided`` and ``used`` as explained above is the ``max``
operation where ``unused = 0``, ``undecided = 1`` and ``used = 2``.

The proper way would be to compute

.. code-block:: none

    max(s, f(s), f(f(s)), f(f(f(s))), ...)

as state after the loop. Since ``f`` just has a range of three different values,
iterating it has to reach a cycle after at most three iterations,
and thus ``f(f(f(s)))`` has to equal one of ``s``, ``f(s)``, or ``f(f(s))``
and thus

.. code-block:: none

    max(s, f(s), f(f(s))) = max(s, f(s), f(f(s)), f(f(f(s))), ...).

In summary, running the loop at most twice is enough because there are only three
different states.

For switch statements that have a "default"-case, there is no control-flow
part that skips the switch.

When a variable goes out of scope, all statements still in the "undecided"
state are changed to "unused", unless the variable is the return
parameter of a function - there, the state changes to "used".

In the second traversal, all assignments that are in the "unused" state are removed.

This step is usually run right after the SSA transform to complete
the generation of the pseudo-SSA.

Tools
-----

Movability
^^^^^^^^^^

Movability is a property of an expression. It roughly means that the expression
is side-effect free and its evaluation only depends on the values of variables
and the call-constant state of the environment. Most expressions are movable.
The following parts make an expression non-movable:

- function calls (might be relaxed in the future if all statements in the function are movable)
- opcodes that (can) have side-effects (like ``call`` or ``selfdestruct``)
- opcodes that read or write memory, storage or external state information
- opcodes that depend on the current PC, memory size or returndata size

DataflowAnalyzer
^^^^^^^^^^^^^^^^

The Dataflow Analyzer is not an optimizer step itself but is used as a tool
by other components. While traversing the AST, it tracks the current value of
each variable, as long as that value is a movable expression.
It records the variables that are part of the expression
that is currently assigned to each other variable. Upon each assignment to
a variable ``a``, the current stored value of ``a`` is updated and
all stored values of all variables ``b`` are cleared whenever ``a`` is part
of the currently stored expression for ``b``.

At control-flow joins, knowledge about variables is cleared if they have or would be assigned
in any of the control-flow paths. For instance, upon entering a
for loop, all variables are cleared that will be assigned during the
body or the post block.

Expression-Scale Simplifications
--------------------------------

These simplification passes change expressions and replace them by equivalent
and hopefully simpler expressions.

.. _common-subexpression-eliminator:

CommonSubexpressionEliminator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This step uses the Dataflow Analyzer and replaces subexpressions that
syntactically match the current value of a variable by a reference to
that variable. This is an equivalence transform because such subexpressions have
to be movable.

All subexpressions that are identifiers themselves are replaced by their
current value if the value is an identifier.

The combination of the two rules above allow to compute a local value
numbering, which means that if two variables have the same
value, one of them will always be unused. The Unused Pruner or the
Redundant Assign Eliminator will then be able to fully eliminate such
variables.

This step is especially efficient if the expression splitter is run
before. If the code is in pseudo-SSA form,
the values of variables are available for a longer time and thus we
have a higher chance of expressions to be replaceable.

The expression simplifier will be able to perform better replacements
if the common subexpression eliminator was run right before it.

.. _expression-simplifier:

Expression Simplifier
^^^^^^^^^^^^^^^^^^^^^

The Expression Simplifier uses the Dataflow Analyzer and makes use
of a list of equivalence transforms on expressions like ``X + 0 -> X``
to simplify the code.

It tries to match patterns like ``X + 0`` on each subexpression.
During the matching procedure, it resolves variables to their currently
assigned expressions to be able to match more deeply nested patterns
even when the code is in pseudo-SSA form.

Some of the patterns like ``X - X -> 0`` can only be applied as long
as the expression ``X`` is movable, because otherwise it would remove its potential side-effects.
Since variable references are always movable, even if their current
value might not be, the Expression Simplifier is again more powerful
in split or pseudo-SSA form.

.. _literal-rematerialiser:

LiteralRematerialiser
^^^^^^^^^^^^^^^^^^^^^

To be documented.

.. _load-resolver:

LoadResolver
^^^^^^^^^^^^

Optimisation stage that replaces expressions of type ``sload(x)`` and ``mload(x)`` by the value
currently stored in storage resp. memory, if known.

Works best if the code is in SSA form.

Prerequisite: Disambiguator, ForLoopInitRewriter.

.. _reasoning-based-simplifier:

ReasoningBasedSimplifier
^^^^^^^^^^^^^^^^^^^^^^^^

This optimizer uses SMT solvers to check whether ``if`` conditions are constant.

- If ``constraints AND condition`` is UNSAT, the condition is never true and the whole body can be removed.
- If ``constraints AND NOT condition`` is UNSAT, the condition is always true and can be replaced by ``1``.

The simplifications above can only be applied if the condition is movable.

It is only effective on the EVM dialect, but safe to use on other dialects.

Prerequisite: Disambiguator, SSATransform.

Statement-Scale Simplifications
-------------------------------

.. _circular-reference-pruner:

CircularReferencesPruner
^^^^^^^^^^^^^^^^^^^^^^^^

This stage removes functions that call each other but are
neither externally referenced nor referenced from the outermost context.

.. _conditional-simplifier:

ConditionalSimplifier
^^^^^^^^^^^^^^^^^^^^^

The Conditional Simplifier inserts assignments to condition variables if the value can be determined
from the control-flow.

Destroys SSA form.

Currently, this tool is very limited, mostly because we do not yet have support
for boolean types. Since conditions only check for expressions being nonzero,
we cannot assign a specific value.

Current features:

- switch cases: insert "<condition> := <caseLabel>"
- after if statement with terminating control-flow, insert "<condition> := 0"

Future features:

- allow replacements by "1"
- take termination of user-defined functions into account

Works best with SSA form and if dead code removal has run before.

Prerequisite: Disambiguator.

.. _conditional-unsimplifier:

ConditionalUnsimplifier
^^^^^^^^^^^^^^^^^^^^^^^

Reverse of Conditional Simplifier.

.. _control-flow-simplifier:

ControlFlowSimplifier
^^^^^^^^^^^^^^^^^^^^^

Simplifies several control-flow structures:

- replace if with empty body with pop(condition)
- remove empty default switch case
- remove empty switch case if no default case exists
- replace switch with no cases with pop(expression)
- turn switch with single case into if
- replace switch with only default case with pop(expression) and body
- replace switch with const expr with matching case body
- replace ``for`` with terminating control flow and without other break/continue by ``if``
- remove ``leave`` at the end of a function.

None of these operations depend on the data flow. The StructuralSimplifier
performs similar tasks that do depend on data flow.

The ControlFlowSimplifier does record the presence or absence of ``break``
and ``continue`` statements during its traversal.

Prerequisite: Disambiguator, FunctionHoister, ForLoopInitRewriter.
Important: Introduces EVM opcodes and thus can only be used on EVM code for now.

.. _dead-code-eliminator:

DeadCodeEliminator
^^^^^^^^^^^^^^^^^^

This optimization stage removes unreachable code.

Unreachable code is any code within a block which is preceded by a
leave, return, invalid, break, continue, selfdestruct or revert.

Function definitions are retained as they might be called by earlier
code and thus are considered reachable.

Because variables declared in a for loop's init block have their scope extended to the loop body,
we require ForLoopInitRewriter to run before this step.

Prerequisite: ForLoopInitRewriter, Function Hoister, Function Grouper

.. _equal-store-eliminator:

EqualStoreEliminator
^^^^^^^^^^^^^^^^^^^^

This steps removes ``mstore(k, v)`` and ``sstore(k, v)`` calls if
there was a previous call to ``mstore(k, v)`` / ``sstore(k, v)``,
no other store in between and the values of ``k`` and ``v`` did not change.

This simple step is effective if run after the SSA transform and the
Common Subexpression Eliminator, because SSA will make sure that the variables
will not change and the Common Subexpression Eliminator re-uses exactly the same
variable if the value is known to be the same.

Prerequisites: Disambiguator, ForLoopInitRewriter

.. _unused-pruner:

UnusedPruner
^^^^^^^^^^^^

This step removes the definitions of all functions that are never referenced.

It also removes the declaration of variables that are never referenced.
If the declaration assigns a value that is not movable, the expression is retained,
but its value is discarded.

All movable expression statements (expressions that are not assigned) are removed.

.. _structural-simplifier:

StructuralSimplifier
^^^^^^^^^^^^^^^^^^^^

This is a general step that performs various kinds of simplifications on
a structural level:

- replace if statement with empty body by ``pop(condition)``
- replace if statement with true condition by its body
- remove if statement with false condition
- turn switch with single case into if
- replace switch with only default case by ``pop(expression)`` and body
- replace switch with literal expression by matching case body
- replace for loop with false condition by its initialization part

This component uses the Dataflow Analyzer.

.. _block-flattener:

BlockFlattener
^^^^^^^^^^^^^^

This stage eliminates nested blocks by inserting the statement in the
inner block at the appropriate place in the outer block. It depends on the
FunctionGrouper and does not flatten the outermost block to keep the form
produced by the FunctionGrouper.

.. code-block:: yul

    {
        {
            let x := 2
            {
                let y := 3
                mstore(x, y)
            }
        }
    }

is transformed to

.. code-block:: yul

    {
        {
            let x := 2
            let y := 3
            mstore(x, y)
        }
    }

As long as the code is disambiguated, this does not cause a problem because
the scopes of variables can only grow.

.. _loop-invariant-code-motion:

LoopInvariantCodeMotion
^^^^^^^^^^^^^^^^^^^^^^^
This optimization moves movable SSA variable declarations outside the loop.

Only statements at the top level in a loop's body or post block are considered, i.e variable
declarations inside conditional branches will not be moved out of the loop.

Requirements:

- The Disambiguator, ForLoopInitRewriter and FunctionHoister must be run upfront.
- Expression splitter and SSA transform should be run upfront to obtain better result.


Function-Level Optimizations
----------------------------

.. _function-specializer:

FunctionSpecializer
^^^^^^^^^^^^^^^^^^^

This step specializes the function with its literal arguments.

If a function, say, ``function f(a, b) { sstore (a, b) }``, is called with literal arguments, for
example, ``f(x, 5)``, where ``x`` is an identifier, it could be specialized by creating a new
function ``f_1`` that takes only one argument, i.e.,

.. code-block:: yul

    function f_1(a_1) {
        let b_1 := 5
        sstore(a_1, b_1)
    }

Other optimization steps will be able to make more simplifications to the function. The
optimization step is mainly useful for functions that would not be inlined.

Prerequisites: Disambiguator, FunctionHoister

LiteralRematerialiser is recommended as a prerequisite, even though it's not required for
correctness.

.. _unused-function-parameter-pruner:

UnusedFunctionParameterPruner
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This step removes unused parameters in a function.

If a parameter is unused, like ``c`` and ``y`` in, ``function f(a,b,c) -> x, y { x := div(a,b) }``, we
remove the parameter and create a new "linking" function as follows:

.. code-block:: yul

    function f(a,b) -> x { x := div(a,b) }
    function f2(a,b,c) -> x, y { x := f(a,b) }

and replace all references to ``f`` by ``f2``.
The inliner should be run afterwards to make sure that all references to ``f2`` are replaced by
``f``.

Prerequisites: Disambiguator, FunctionHoister, LiteralRematerialiser.

The step LiteralRematerialiser is not required for correctness. It helps deal with cases such as:
``function f(x) -> y { revert(y, y} }`` where the literal ``y`` will be replaced by its value ``0``,
allowing us to rewrite the function.

.. _equivalent-function-combiner:

EquivalentFunctionCombiner
^^^^^^^^^^^^^^^^^^^^^^^^^^

If two functions are syntactically equivalent, while allowing variable
renaming but not any re-ordering, then any reference to one of the
functions is replaced by the other.

The actual removal of the function is performed by the Unused Pruner.


Function Inlining
-----------------

.. _expression-inliner:

ExpressionInliner
^^^^^^^^^^^^^^^^^

This component of the optimizer performs restricted function inlining by inlining functions that can be
inlined inside functional expressions, i.e. functions that:

- return a single value.
- have a body like ``r := <functional expression>``.
- neither reference themselves nor ``r`` in the right hand side.

Furthermore, for all parameters, all of the following need to be true:

- The argument is movable.
- The parameter is either referenced less than twice in the function body, or the argument is rather cheap
  ("cost" of at most 1, like a constant up to 0xff).

Example: The function to be inlined has the form of ``function f(...) -> r { r := E }`` where
``E`` is an expression that does not reference ``r`` and all arguments in the function call are movable expressions.

The result of this inlining is always a single expression.

This component can only be used on sources with unique names.

.. _full-inliner:

FullInliner
^^^^^^^^^^^

The Full Inliner replaces certain calls of certain functions
by the function's body. This is not very helpful in most cases, because
it just increases the code size but does not have a benefit. Furthermore,
code is usually very expensive and we would often rather have shorter
code than more efficient code. In same cases, though, inlining a function
can have positive effects on subsequent optimizer steps. This is the case
if one of the function arguments is a constant, for example.

During inlining, a heuristic is used to tell if the function call
should be inlined or not.
The current heuristic does not inline into "large" functions unless
the called function is tiny. Functions that are only used once
are inlined, as well as medium-sized functions, while function
calls with constant arguments allow slightly larger functions.


In the future, we may include a backtracking component
that, instead of inlining a function right away, only specializes it,
which means that a copy of the function is generated where
a certain parameter is always replaced by a constant. After that,
we can run the optimizer on this specialized function. If it
results in heavy gains, the specialized function is kept,
otherwise the original function is used instead.

Cleanup
-------

The cleanup is performed at the end of the optimizer run. It tries
to combine split expressions into deeply nested ones again and also
improves the "compilability" for stack machines by eliminating
variables as much as possible.

.. _expression-joiner:

ExpressionJoiner
^^^^^^^^^^^^^^^^

This is the opposite operation of the expression splitter. It turns a sequence of
variable declarations that have exactly one reference into a complex expression.
This stage fully preserves the order of function calls and opcode executions.
It does not make use of any information concerning the commutativity of the opcodes;
if moving the value of a variable to its place of use would change the order
of any function call or opcode execution, the transformation is not performed.

Note that the component will not move the assigned value of a variable assignment
or a variable that is referenced more than once.

The snippet ``let x := add(0, 2) let y := mul(x, mload(2))`` is not transformed,
because it would cause the order of the call to the opcodes ``add`` and
``mload`` to be swapped - even though this would not make a difference
because ``add`` is movable.

When reordering opcodes like that, variable references and literals are ignored.
Because of that, the snippet ``let x := add(0, 2) let y := mul(x, 3)`` is
transformed to ``let y := mul(add(0, 2), 3)``, even though the ``add`` opcode
would be executed after the evaluation of the literal ``3``.

.. _SSA-reverser:

SSAReverser
^^^^^^^^^^^

This is a tiny step that helps in reversing the effects of the SSA transform
if it is combined with the Common Subexpression Eliminator and the
Unused Pruner.

The SSA form we generate is detrimental to code generation on the EVM and
WebAssembly alike because it generates many local variables. It would
be better to just re-use existing variables with assignments instead of
fresh variable declarations.

The SSA transform rewrites

.. code-block:: yul

    let a := calldataload(0)
    mstore(a, 1)

to

.. code-block:: yul

    let a_1 := calldataload(0)
    let a := a_1
    mstore(a_1, 1)
    let a_2 := calldataload(0x20)
    a := a_2

The problem is that instead of ``a``, the variable ``a_1`` is used
whenever ``a`` was referenced. The SSA transform changes statements
of this form by just swapping out the declaration and the assignment. The above
snippet is turned into

.. code-block:: yul

    let a := calldataload(0)
    let a_1 := a
    mstore(a_1, 1)
    a := calldataload(0x20)
    let a_2 := a

This is a very simple equivalence transform, but when we now run the
Common Subexpression Eliminator, it will replace all occurrences of ``a_1``
by ``a`` (until ``a`` is re-assigned). The Unused Pruner will then
eliminate the variable ``a_1`` altogether and thus fully reverse the
SSA transform.

.. _stack-compressor:

StackCompressor
^^^^^^^^^^^^^^^

One problem that makes code generation for the Ethereum Virtual Machine
hard is the fact that there is a hard limit of 16 slots for reaching
down the expression stack. This more or less translates to a limit
of 16 local variables. The stack compressor takes Yul code and
compiles it to EVM bytecode. Whenever the stack difference is too
large, it records the function this happened in.

For each function that caused such a problem, the Rematerialiser
is called with a special request to aggressively eliminate specific
variables sorted by the cost of their values.

On failure, this procedure is repeated multiple times.

.. _rematerialiser:

Rematerialiser
^^^^^^^^^^^^^^

The rematerialisation stage tries to replace variable references by the expression that
was last assigned to the variable. This is of course only beneficial if this expression
is comparatively cheap to evaluate. Furthermore, it is only semantically equivalent if
the value of the expression did not change between the point of assignment and the
point of use. The main benefit of this stage is that it can save stack slots if it
leads to a variable being eliminated completely (see below), but it can also
save a DUP opcode on the EVM if the expression is very cheap.

The Rematerialiser uses the Dataflow Analyzer to track the current values of variables,
which are always movable.
If the value is very cheap or the variable was explicitly requested to be eliminated,
the variable reference is replaced by its current value.

.. _for-loop-condition-out-of-body:

ForLoopConditionOutOfBody
^^^^^^^^^^^^^^^^^^^^^^^^^

Reverses the transformation of ForLoopConditionIntoBody.

For any movable ``c``, it turns

.. code-block:: none

    for { ... } 1 { ... } {
    if iszero(c) { break }
    ...
    }

into

.. code-block:: none

    for { ... } c { ... } {
    ...
    }

and it turns

.. code-block:: none

    for { ... } 1 { ... } {
    if c { break }
    ...
    }

into

.. code-block:: none

    for { ... } iszero(c) { ... } {
    ...
    }

The LiteralRematerialiser should be run before this step.


WebAssembly specific
--------------------

MainFunction
^^^^^^^^^^^^

Changes the topmost block to be a function with a specific name ("main") which has no
inputs nor outputs.

Depends on the Function Grouper.
>>>>>>> 43f29c00da02e19ff10d43f7eb6955d627c57728
