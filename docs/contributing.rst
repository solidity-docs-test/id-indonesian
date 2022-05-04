<<<<<<< HEAD
#############
Berkontribusi
#############

Bantuan selalu diterima dan ada banyak pilihan bagaimana Anda dapat berkontribusi pada Solidity.

Secara khusus, kami menghargai dukungan di bidang-bidang berikut:

* Melaporkan masalah.
* Memperbaiki dan menanggapi `Solidity's GitHub issues
  <https://github.com/ethereum/solidity/issues>`_, terutama yang ditandai sebagai
  `"good first issue" <https://github.com/ethereum/solidity/labels/good%20first%20issue>`_ yang mana
   dimaksudkan sebagai masalah pengantar untuk kontributor eksternal.
* Memperbaiki dokumentasi.
* Menerjemahkan dokumentasi ke lebih banyak bahasa.
* Menanggapi pertanyaan dari pengguna lain di `StackExchange
  <https://ethereum.stackexchange.com>`_ dan `Solidity Gitter Chat
  <https://gitter.im/ethereum/solidity>`_.
* Terlibat dalam proses desain bahasa dengan mengusulkan perubahan bahasa atau fitur baru di `Forum Solidity <https://forum.soliditylang.org/>`_ dan memberikan masukan.

Untuk memulai, Anda dapat mencoba :ref:`building-from-source` untuk membiasakan
diri Anda dengan komponen Solidity dan proses build. Juga, mungkin
berguna untuk menjadi ahli dalam menulis kontrak pintar di Solidity.

Harap diperhatikan bahwa proyek ini dirilis dengan `Kode Etik Kontributor <https://raw.githubusercontent.com/ethereum/solidity/develop/CODE_OF_CONDUCT.md>`_. Dengan berpartisipasi dalam proyek ini - dalam masalah, permintaan tarik, atau saluran Gitter - Anda setuju untuk mematuhi persyaratannya.

Panggilan Tim
=============

Jika Anda memiliki masalah atau menarik permintaan untuk didiskusikan, atau tertarik
untuk mendengar apa yang sedang dikerjakan oleh tim dan kontributor, Anda dapat bergabung
dengan panggilan tim publik kami:

- Senin pukul 15:00 CET/CEST.
- Rabu pukul 14:00 CET/CEST.

Kedua panggilan berlangsung di `Jitsi <https://meet.komputing.org/solidity>`_.

Cara Melaporkan Masalah
=======================

Untuk melaporkan masalah, silakan gunakan
`GitHub issue tracker <https://github.com/ethereum/solidity/issues>`_. Saat
melaporkan masalah, harap sebutkan detail berikut:

* Versi solidity.
* Kode sumber (jika ada).
* Sistem operasi.
* Langkah-langkah untuk mereproduksi masalah.
* Perilaku aktual vs. yang diharapkan.

Mengurangi kode sumber yang menyebabkan masalah seminimal mungkin selalu
sangat membantu dan terkadang bahkan memperjelas kesalahpahaman.

Workflow untuk Pull Requests
============================

Untuk berkontribusi, harap keluar dari cabang ``develop`` dan buat
perubahan di sana. Pesan komit Anda harus merinci *mengapa* Anda melakukan perubahan
selain *apa* yang Anda lakukan (kecuali itu adalah perubahan kecil).

Jika Anda perlu menarik perubahan apa pun dari ``develop`` setelah membuat fork (untuk
misalnya, untuk menyelesaikan potensi konflik penggabungan), harap hindari menggunakan ``git merge``
dan sebagai gantinya, ``git rebase`` cabang Anda. Ini akan membantu kami meninjau perubahan Anda
lebih mudah.

Selain itu, jika Anda menulis fitur baru, pastikan Anda menambahkan kasus uji yang sesuai
di bawah ``test/`` (lihat di bawah).

Namun, jika Anda membuat perubahan yang lebih besar, silakan berkonsultasi dengan `Solidity Development Gitter channel
<https://gitter.im/ethereum/solidity-dev>`_ (berbeda dari yang disebutkan di atas, yang ini adalah
berfokus pada kompiler dan pengembangan bahasa daripada penggunaan bahasa) terlebih dahulu.

Fitur baru dan perbaikan bug harus ditambahkan ke file ``Changelog.md``: harap
ikuti gaya entri sebelumnya, jika berlaku.

Terakhir, pastikan Anda menghormati `gaya pengkodean
<https://github.com/ethereum/solidity/blob/develop/CODING_STYLE.md>`_
untuk proyek ini. Juga, meskipun kami melakukan pengujian CI, harap uji kode Anda dan
pastikan itu dibangun secara lokal sebelum mengirimkan permintaan pull.

Terima kasih untuk bantuannya!

Menjalankan Tes Compiler
==========================

Prasyarat
-------------

Untuk menjalankan semua pengujian kompiler, Anda mungkin ingin menginstal beberapa dependensi secara
opsional (`evmone <https://github.com/ethereum/evmone/releases>`_,
`libz3 <https://github.com/Z3Prover/z3>`_, dan
`libhera <https://github.com/ewasm/hera>`_).

Di macOS, beberapa skrip pengujian mengharapkan GNU coreutils diinstal.
Ini paling mudah dilakukan dengan menggunakan Homebrew: ``brew install coreutils``.

Menjalankan Tes
-----------------

Solidity mencakup berbagai jenis tes, kebanyakan dibundel ke dalam aplikasi ``soltest``
`Boost C++ Test Framework <https://www.boost.org/doc/libs/release/libs/test/doc/html/index.html>`_.
Jalankan ``build/test/soltest`` atau wrapper ``scripts/soltest.sh`` cukup untuk sebagian besar perubahan.

Skrip ``./scripts/tests.sh`` menjalankan sebagian besar pengujian Solidity secara otomatis,
termasuk yang dibundel ke dalam `Boost C++ Test Framework <https://www.boost.org/doc/libs/release/libs/test/doc/html/index.html>`_
aplikasi ``soltest`` (atau pembungkusnya ``scripts/soltest.sh``), serta pengujian baris perintah dan
tes kompilasi.

Sistem pengujian secara otomatis mencoba menemukan lokasi
`evmone <https://github.com/ethereum/evmone/releases>`_ untuk menjalankan tes semantik.

Library ``evmone`` harus ditempatkan di direktori ``deps`` atau ``deps/lib`` relatif terhadap
direktori kerja saat ini, ke induknya atau induknya. Atau lokasi eksplisit
untuk objek bersama ``evmone`` dapat ditentukan melalui environment variable ``ETH_EVMONE``.

``evmone`` diperlukan terutama untuk menjalankan tes semantik dan gas.
Jika Anda belum menginstalnya, Anda dapat melewati pengujian ini dengan meneruskan flag ``--no-semantic-tests``
ke ``scripts/soltest.sh``.

Menjalankan tes Ewasm dinonaktifkan secara default dan dapat diaktifkan secara eksplisit
melalui ``./scripts/soltest.sh --ewasm`` dan membutuhkan `hera <https://github.com/ewasm/hera>`_
untuk dapat ditemukan oleh ``soltest``.
Mekanisme untuk menemukan library ``hera`` sama seperti untuk ``evmone``, kecuali
variabel untuk menentukan lokasi eksplisit disebut ``ETH_HERA``.

Library ``evmone`` dan ``hera`` harus diakhiri dengan nama file
ekstensi ``.so`` di Linux, ``.dll`` di sistem Windows dan ``.dylib`` di macOS.

Untuk menjalankan pengujian SMT, library ``libz3`` harus diinstal dan dapat ditemukan
oleh ``cmake`` selama tahap konfigurasi kompiler.

Jika library ``libz3`` tidak diinstal pada sistem Anda, Anda harus menonaktifkan
Tes SMT dengan mengekspor ``SMT_FLAGS=--no-smt`` sebelum menjalankan ``./scripts/tests.sh`` atau
menjalankan ``./scripts/soltest.sh --no-smt``.
Pengujian ini adalah ``libsolidity/smtCheckerTests`` dan ``libsolidity/smtCheckerTestsJSON``.

.. note ::

    Untuk mendapatkan daftar semua pengujian unit yang dijalankan oleh Soltest, jalankan ``./build/test/soltest --list_content=HRF``.

Untuk hasil yang lebih cepat, Anda dapat menjalankan subset dari, atau tes tertentu.

Untuk menjalankan subset pengujian, Anda dapat menggunakan filter:
``./scripts/soltest.sh -t TestSuite/TestName``,
dimana ``TestName`` bisa menjadi wildcard ``*``.

Atau, misalnya, untuk menjalankan semua tes untuk yul disambiguator:
``./scripts/soltest.sh -t "yulOptimizerTests/disambiguator/*" --no-smt``.

``./build/test/soltest --help`` memiliki bantuan ekstensif pada semua opsi yang tersedia.

Lihat terutama:

- `show_progress (-p) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/show_progress.html>`_ to show test completion,
- `run_test (-t) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/run_test.html>`_ to run specific tests cases, and
- `report-level (-r) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/report_level.html>`_ give a more detailed report.

.. note ::

    Mereka yang bekerja di lingkungan Windows yang ingin menjalankan set dasar di atas
    tanpa libz3. Menggunakan Git Bash, Anda menggunakan: ``./build/test/Release/soltest.exe -- --no-smt``.
    Jika Anda menjalankan ini di Command Prompt biasa, gunakan ``.\build\test\Release\soltest.exe -- --no-smt``.

Jika Anda ingin men-debug menggunakan GDB, pastikan Anda membangun yang berbeda dari "biasa".
Misalnya, Anda dapat menjalankan perintah berikut di folder ``build`` Anda:
.. code-block:: bash

   cmake -DCMAKE_BUILD_TYPE=Debug ..
   make

Ini menciptakan simbol sehingga saat Anda men-debug pengujian menggunakan flag ``--debug``,
Anda memiliki akses ke fungsi dan variabel di mana Anda dapat memecahkan atau mencetaknya.

CI menjalankan tes tambahan (termasuk ``solc-js`` dan menguji frameworks Solidity
pihak ketiga) yang memerlukan kompilasi target Emscripten.

Menulis dan Menjalankan Tes Sintaks
-----------------------------------

Tes sintaks memeriksa bahwa kompiler menghasilkan pesan kesalahan yang benar untuk kode yang tidak valid
dan menerima kode yang valid dengan benar.
Mereka disimpan dalam file individual di dalam folder ``tests/libsolidity/syntaxTests``.
File-file ini harus berisi anotasi, yang menyatakan hasil yang diharapkan dari pengujian masing-masing.
Test suite mengkompilasi dan memeriksanya terhadap ekspektasi yang diberikan.

Misalnya: ``./test/libsolidity/syntaxTests/double_stateVariable_declaration.sol``

.. code-block:: solidity

    contract test {
        uint256 variable;
        uint128 variable;
    }
    // ----
    // DeclarationError: (36-52): Identifier already declared.

Pengujian sintaks harus berisi setidaknya kontrak yang sedang diuji itu sendiri, diikuti oleh pemisah ``// ----``. Komentar yang mengikuti pemisah digunakan untuk menggambarkan
kesalahan atau peringatan kompiler yang diharapkan. Rentang angka menunjukkan lokasi di sumber tempat kesalahan terjadi.
Jika Anda ingin kontrak dikompilasi tanpa kesalahan atau peringatan, Anda dapat pergi
keluar pemisah dan komentar yang mengikutinya.

Dalam contoh di atas, variabel state ``variabel`` dideklarasikan dua kali, dimana itu tidak diperbolehkan. Ini menghasilkan ``DeclarationError`` yang menyatakan bahwa pengidentifikasi sudah dideklarasikan.

Alat ``isoltest`` digunakan untuk pengujian ini dan Anda dapat menemukannya di ``./build/test/tools/``. Ini adalah alat interaktif yang memungkinkan
mengedit kontrak yang gagal menggunakan editor teks pilihan Anda. Mari kita coba menghentikan pengujian ini dengan menghapus deklarasi kedua dari ``variabel``:

.. code-block:: solidity

    contract test {
        uint256 variable;
    }
    // ----
    // DeclarationError: (36-52): Identifier already declared.

Menjalankan ``./build/test/tools/isoltest`` lagi menghasilkan kegagalan pengujian:

.. code-block:: text

    syntaxTests/double_stateVariable_declaration.sol: FAIL
        Contract:
            contract test {
                uint256 variable;
            }

        Expected result:
            DeclarationError: (36-52): Identifier already declared.
        Obtained result:
            Success


``isoltest`` mencetak hasil yang diharapkan di sebelah hasil yang diperoleh, dan juga
menyediakan cara untuk mengedit, memperbarui atau melewati file kontrak saat ini, atau keluar dari aplikasi.

Ini menawarkan beberapa opsi untuk tes yang gagal:

- ``edit``: ``isoltest`` mencoba membuka kontrak di editor sehingga Anda dapat menyesuaikannya. Itu bisa menggunakan editor yang diberikan pada baris perintah (sebagai ``isoltest --editor /path/to/editor``), dalam environment variable ``EDITOR`` atau hanya ``/usr/bin/editor`` ( dalam urutan itu).
- ``update``: Memperbarui ekspektasi untuk kontrak yang sedang diuji. Ini memperbarui anotasi dengan menghapus harapan yang tidak terpenuhi dan menambahkan harapan yang hilang. Tes kemudian dijalankan lagi.
- ``skip``: Melewati eksekusi tes khusus ini.
- ``quit``: Keluar ``isoltest``.

Semua opsi ini berlaku untuk kontrak saat ini, mengharapkan ``quit`` yang menghentikan seluruh proses pengujian.

Secara otomatis memperbarui tes di atas mengubahnya menjadi

.. code-block:: solidity

    contract test {
        uint256 variable;
    }
    // ----

dan menjalankan kembali tes. Sekarang lewat lagi:

.. code-block:: text

    Re-running test case...
    syntaxTests/double_stateVariable_declaration.sol: OK


.. note::

    Pilih nama untuk file kontrak yang menjelaskan apa yang sedang diuji, mis. ``double_variable_declaration.sol``.
    Jangan memasukkan lebih dari satu kontrak ke dalam satu file, kecuali Anda sedang menguji inheritance atau cross-contract calls.
    Setiap file harus menguji satu aspek dari fitur baru Anda.


Menjalankan Fuzzer melalui AFL
==============================

Fuzzing adalah teknik yang menjalankan program pada input yang kurang lebih acak untuk menemukan status eksekusi yang
luar biasa (kesalahan segmentasi, pengecualian, dll). Fuzzers modern pintar dan menjalankan pencarian
terarah di dalam input. Kami memiliki biner khusus yang disebut ``solfuzzer`` yang mengambil kode sumber
sebagai input dan gagal setiap kali menemukan kesalahan kompiler internal, kesalahan segmentasi atau serupa, tetapi
tidak gagal jika misalnya, kode berisi kesalahan. Dengan cara ini, alat fuzzing dapat menemukan masalah internal di kompiler.

Kami terutama menggunakan `AFL <https://lcamtuf.coredump.cx/afl/>`_ untuk fuzzing. Anda perlu mengunduh dan
menginstal paket AFL dari repositori Anda (afl, afl-clang) atau buat secara manual.
Selanjutnya, buat Solidity (atau hanya biner ``solfuzzer``) dengan AFL sebagai kompiler Anda:

.. code-block:: bash

    cd build
    # if needed
    make clean
    cmake .. -DCMAKE_C_COMPILER=path/to/afl-gcc -DCMAKE_CXX_COMPILER=path/to/afl-g++
    make solfuzzer

Pada tahap ini Anda seharusnya dapat melihat pesan yang mirip dengan berikut ini:

.. code-block:: text

    Scanning dependencies of target solfuzzer
    [ 98%] Building CXX object test/tools/CMakeFiles/solfuzzer.dir/fuzzer.cpp.o
    afl-cc 2.52b by <lcamtuf@google.com>
    afl-as 2.52b by <lcamtuf@google.com>
    [+] Instrumented 1949 locations (64-bit, non-hardened mode, ratio 100%).
    [100%] Linking CXX executable solfuzzer

Jika pesan instrumentasi tidak muncul, coba alihkan tanda cmake yang menunjuk ke biner dentang AFL:

.. code-block:: bash

    # if previously failed
    make clean
    cmake .. -DCMAKE_C_COMPILER=path/to/afl-clang -DCMAKE_CXX_COMPILER=path/to/afl-clang++
    make solfuzzer

Jika tidak, saat eksekusi, fuzzer berhenti dengan kesalahan yang mengatakan biner tidak diinstrumentasi:

.. code-block:: text

    afl-fuzz 2.52b by <lcamtuf@google.com>
    ... (truncated messages)
    [*] Validating target binary...

    [-] Looks like the target binary is not instrumented! The fuzzer depends on
        compile-time instrumentation to isolate interesting test cases while
        mutating the input data. For more information, and for tips on how to
        instrument binaries, please see /usr/share/doc/afl-doc/docs/README.

        When source code is not available, you may be able to leverage QEMU
        mode support. Consult the README for tips on how to enable this.
        (It is also possible to use afl-fuzz as a traditional, "dumb" fuzzer.
        For that, you can use the -n option - but expect much worse results.)

    [-] PROGRAM ABORT : No instrumentation detected
             Location : check_binary(), afl-fuzz.c:6920


Selanjutnya, Anda memerlukan beberapa contoh file sumber. Ini membuatnya lebih mudah bagi fuzzer
untuk menemukan kesalahan. Anda dapat menyalin beberapa file dari pengujian sintaks atau mengekstrak file pengujian
dari dokumentasi atau tes lainnya:

.. code-block:: bash

    mkdir /tmp/test_cases
    cd /tmp/test_cases
    # extract from tests:
    path/to/solidity/scripts/isolate_tests.py path/to/solidity/test/libsolidity/SolidityEndToEndTest.cpp
    # extract from documentation:
    path/to/solidity/scripts/isolate_tests.py path/to/solidity/docs

Dokumentasi AFL menyatakan bahwa corpus (file input awal) tidak boleh
terlalu besar. File itu sendiri tidak boleh lebih besar dari 1 kB dan harus ada
paling banyak satu file input per fungsionalitas, jadi lebih baik mulai dengan jumlah kecil.
Ada juga alat yang disebut ``afl-cmin`` yang dapat memangkas file input
yang menghasilkan perilaku biner yang serupa.

Sekarang jalankan fuzzer (``-m`` menambah ukuran memori hingga 60 MB):

.. code-block:: bash

    afl-fuzz -m 60 -i /tmp/test_cases -o /tmp/fuzzer_reports -- /path/to/solfuzzer

Fuzzer membuat file sumber yang menyebabkan kegagalan di ``/tmp/fuzzer_reports``.
Seringkali ditemukan banyak file sumber serupa yang menghasilkan kesalahan yang sama. Kamu bisa
menggunakan alat ``scripts/uniqueErrors.sh`` untuk memfilter kesalahan unik.

Whisker
=======

*Whisker* adalah sistem templating string yang mirip dengan `Mustache <https://mustache.github.io>`_. Ini digunakan oleh
compiler di berbagai tempat untuk membantu keterbacaan, dan dengan demikian pemeliharaan dan verifikasi, kode.

Sintaksnya hadir dengan perbedaan substansial pada Kumis. Penanda template ``{{`` dan ``}}`` adalah
diganti dengan ``<`` dan ``>`` untuk membantu penguraian dan menghindari konflik dengan :ref:`yul`
(Simbol ``<`` dan ``>`` tidak valid dalam perakitan sebaris, sedangkan ``{`` dan ``}`` digunakan untuk membatasi blok).
Keterbatasan lain adalah bahwa daftar hanya diselesaikan satu kedalaman dan tidak berulang. Ini mungkin berubah di masa depan.

Spesifikasi kasarnya adalah sebagai berikut:

Setiap kemunculan ``<name>`` diganti dengan nilai string dari variabel yang disediakan ``name`` tanpa
pelolosan dan tanpa penggantian berulang. Suatu area dapat dibatasi dengan ``<#name>...</name>``. Itu digantikan
oleh sebanyak mungkin rangkaian isinya karena ada set variabel yang dipasok ke sistem templat,
setiap kali mengganti item ``<inner>`` dengan nilainya masing-masing. Variabel tingkat atas juga dapat digunakan
di dalam area tersebut.

Ada juga kondisional dari bentuk ``<?name>...<!name>...</name>``, di mana penggantian template
berlanjut secara rekursif baik di segmen pertama atau kedua tergantung pada nilai boolean
parameter ``name``. Jika ``<?+name>...<!+name>...</+name>`` digunakan, maka pemeriksaannya adalah apakah
parameter string ``name`` tidak kosong.

.. _documentation-style:

Panduan Gaya Dokumentasi
=========================

Di bagian berikut, Anda akan menemukan rekomendasi gaya yang secara khusus berfokus pada kontribusi
dokumentasi untuk Solidity.

Bahasa Inggris
----------------

Gunakan bahasa Inggris, dengan ejaan bahasa Inggris British lebih disukai, kecuali menggunakan nama proyek atau merek. Cobalah untuk mengurangi
penggunaan bahasa gaul dan referensi lokal, buat bahasa Anda sejelas mungkin untuk semua pembaca. Di bawah ini adalah beberapa referensi untuk membantu:

* `Bahasa Inggris teknis yang disederhanakan <https://en.wikipedia.org/wiki/Simplified_Technical_English>`_
* `Bahasa Inggris Internasional <https://en.wikipedia.org/wiki/International_English>`_
* `ejaan bahasa Inggris british <https://en.oxforddictionaries.com/spelling/british-and-spelling>`_


.. note::

    Walaupun dokumentasi resmi Solidity ditulis dalam bahasa Inggris, ada kontribusi komunitas :ref:`translations`
    dalam bahasa lain yang tersedia. Silakan merujuk ke `panduan terjemahan <https://github.com/solidity-docs/translation-guide>`_
    untuk informasi tentang bagaimana berkontribusi pada terjemahan komunitas.

Judul Kasus untuk Judul
-----------------------

Gunakan `title case <https://titlecase.com>`_ untuk heading. Ini berarti huruf besar semua kata utama
dalam judul, tetapi bukan artikel, konjungsi, dan preposisi kecuali mereka memulai
judul.

Misalnya, berikut ini semua benar:

* Judul Kasus untuk Headings.
* Untuk Headings Gunakan Title Case.
* Nama Variabel Lokal dan state.
* Urutan Tata Letak.

Perluas Kontraksi
-------------------

Gunakan kontraksi yang diperluas untuk kata-kata, misalnya:

* "Do not" dari pada "Don't".
* "Can not" dari pada "Can't".

Suara Aktif dan Pasif
------------------------

Suara aktif biasanya direkomendasikan untuk dokumentasi gaya tutorial karena
membantu pembaca memahami siapa atau apa yang melakukan tugas. Namun, sebagai
Dokumentasi solidity adalah campuran dari tutorial dan konten referensi, suara
pasif terkadang lebih dapat diterapkan.

Sebagai ringkasan:

* Gunakan suara pasif untuk referensi teknis, misalnya definisi bahasa dan internal VM Ethereum.
* Gunakan suara aktif saat menjelaskan rekomendasi tentang cara menerapkan aspek Solidity.

Misalnya, di bawah ini dalam bentuk pasif karena menentukan aspek Solidity:

  Fungsi dapat dideklarasikan ``pure`` dalam hal ini mereka berjanji untuk tidak membaca
  dari atau mengubah state.

Sebagai contoh, di bawah ini adalah suara aktif saat membahas aplikasi Solidity:

  Saat menjalankan kompiler, Anda dapat menentukan cara menemukan elemen pertama
  dari sebuah jalur, dan juga pemetaan ulang awalan jalur.

Istilah Umum
------------

* "Function parameters" dan "return variables", bukan parameter input dan output.

Contoh Kode
-------------

Proses CI menguji semua contoh kode yang diformat blok kode yang dimulai dengan ``pragma solidity``, ``contract``, ``library``
atau ``interface`` menggunakan skrip ``./test/cmdlineTests.sh`` saat Anda membuat PR. Jika Anda menambahkan contoh kode baru,
pastikan mereka bekerja dan lulus tes sebelum membuat PR.

Pastikan bahwa semua contoh kode dimulai dengan versi ``pragma`` yang memiliki rentang terbesar di mana kode kontrak berlaku.
Misalnya ``pragma solidity >=0.4.0 <0.9.0;``.

Menjalankan Tes Dokumentasi
---------------------------

Pastikan kontribusi Anda lulus uji dokumentasi kami dengan menjalankan ``./scripts/docs.sh`` yang menginstal dependensi
yang diperlukan untuk dokumentasi dan memeriksa masalah apa pun seperti tautan rusak atau masalah sintaksis.

Desain Bahasa Solidity
========================

Untuk secara aktif terlibat dalam proses desain bahasa dan berbagi ide Anda tentang masa depan Solidity,
silakan bergabung dengan `forum Solidity <https://forum.soliditylang.org/>`_.

Forum Solidity berfungsi sebagai tempat untuk mengusulkan dan mendiskusikan fitur bahasa baru dan implementasinya di
tahap awal ide atau modifikasi fitur yang ada.

Segera setelah proposal menjadi lebih nyata,
implementasinya juga akan dibahas di `Repositori GitHub Solidity <https://github.com/ethereum/solidity>`_
dalam bentuk masalah.

Selain forum dan diskusi masalah, kami secara teratur menyelenggarakan panggilan diskusi desain bahasa di mana topik, masalah, atau implementasi
fitur yang dipilih diperdebatkan secara rinci. Undangan untuk panggilan tersebut dibagikan melalui forum.

Kami juga membagikan survei umpan balik dan konten lain yang relevan dengan desain bahasa di forum.

Jika Anda ingin mengetahui posisi tim dalam hal atau menerapkan fitur baru, Anda dapat mengikuti status implementasi di `Solidity Github project <https://github.com/ethereum/solidity/projects/43>`_.
Masalah dalam backlog desain memerlukan spesifikasi lebih lanjut dan akan dibahas dalam panggilan desain bahasa atau dalam panggilan tim reguler. Anda dapat
melihat perubahan yang akan datang untuk rilis terbaru berikutnya dengan mengubah dari default branch (`develop`) ke `breaking branch <https://github.com/ethereum/solidity/tree/breaking>`_.

Untuk kasus dan pertanyaan ad-hoc, Anda dapat menghubungi kami melalui `Solidity-dev Gitter channel <https://gitter.im/ethereum/solidity-dev>`_,
ruang obrolan khusus untuk percakapan seputar kompiler Solidity dan pengembangan bahasa.

Kami senang mendengar pendapat Anda tentang bagaimana kami dapat meningkatkan proses desain bahasa menjadi lebih kolaboratif dan transparan.
=======
############
Contributing
############

Help is always welcome and there are plenty of options how you can contribute to Solidity.

In particular, we appreciate support in the following areas:

* Reporting issues.
* Fixing and responding to `Solidity's GitHub issues
  <https://github.com/ethereum/solidity/issues>`_, especially those tagged as
  `"good first issue" <https://github.com/ethereum/solidity/labels/good%20first%20issue>`_ which are
  meant as introductory issues for external contributors.
* Improving the documentation.
* Translating the documentation into more languages.
* Responding to questions from other users on `StackExchange
  <https://ethereum.stackexchange.com>`_ and the `Solidity Gitter Chat
  <https://gitter.im/ethereum/solidity>`_.
* Getting involved in the language design process by proposing language changes or new features in the `Solidity forum <https://forum.soliditylang.org/>`_ and providing feedback.

To get started, you can try :ref:`building-from-source` in order to familiarize
yourself with the components of Solidity and the build process. Also, it may be
useful to become well-versed at writing smart-contracts in Solidity.

Please note that this project is released with a `Contributor Code of Conduct <https://raw.githubusercontent.com/ethereum/solidity/develop/CODE_OF_CONDUCT.md>`_. By participating in this project - in the issues, pull requests, or Gitter channels - you agree to abide by its terms.

Team Calls
==========

If you have issues or pull requests to discuss, or are interested in hearing what
the team and contributors are working on, you can join our public team calls:

- Mondays at 3pm CET/CEST.
- Wednesdays at 2pm CET/CEST.

Both calls take place on `Jitsi <https://meet.ethereum.org/solidity>`_.

How to Report Issues
====================

To report an issue, please use the
`GitHub issues tracker <https://github.com/ethereum/solidity/issues>`_. When
reporting issues, please mention the following details:

* Solidity version.
* Source code (if applicable).
* Operating system.
* Steps to reproduce the issue.
* Actual vs. expected behaviour.

Reducing the source code that caused the issue to a bare minimum is always
very helpful and sometimes even clarifies a misunderstanding.

Workflow for Pull Requests
==========================

In order to contribute, please fork off of the ``develop`` branch and make your
changes there. Your commit messages should detail *why* you made your change
in addition to *what* you did (unless it is a tiny change).

If you need to pull in any changes from ``develop`` after making your fork (for
example, to resolve potential merge conflicts), please avoid using ``git merge``
and instead, ``git rebase`` your branch. This will help us review your change
more easily.

Additionally, if you are writing a new feature, please ensure you add appropriate
test cases under ``test/`` (see below).

However, if you are making a larger change, please consult with the `Solidity Development Gitter channel
<https://gitter.im/ethereum/solidity-dev>`_ (different from the one mentioned above, this one is
focused on compiler and language development instead of language usage) first.

New features and bugfixes should be added to the ``Changelog.md`` file: please
follow the style of previous entries, when applicable.

Finally, please make sure you respect the `coding style
<https://github.com/ethereum/solidity/blob/develop/CODING_STYLE.md>`_
for this project. Also, even though we do CI testing, please test your code and
ensure that it builds locally before submitting a pull request.

Thank you for your help!

Running the Compiler Tests
==========================

Prerequisites
-------------

For running all compiler tests you may want to optionally install a few
dependencies (`evmone <https://github.com/ethereum/evmone/releases>`_,
`libz3 <https://github.com/Z3Prover/z3>`_, and
`libhera <https://github.com/ewasm/hera>`_).

On macOS some of the testing scripts expect GNU coreutils to be installed.
This can be easiest accomplished using Homebrew: ``brew install coreutils``.

On Windows systems make sure that you have a privilege to create symlinks,
otherwise several tests may fail.
Administrators should have that privilege, but you may also
`grant it to other users <https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/create-symbolic-links#policy-management>`_
or
`enable Developer Mode <https://docs.microsoft.com/en-us/windows/apps/get-started/enable-your-device-for-development>`_.

Running the Tests
-----------------

Solidity includes different types of tests, most of them bundled into the
`Boost C++ Test Framework <https://www.boost.org/doc/libs/release/libs/test/doc/html/index.html>`_ application ``soltest``.
Running ``build/test/soltest`` or its wrapper ``scripts/soltest.sh`` is sufficient for most changes.

The ``./scripts/tests.sh`` script executes most Solidity tests automatically,
including those bundled into the `Boost C++ Test Framework <https://www.boost.org/doc/libs/release/libs/test/doc/html/index.html>`_
application ``soltest`` (or its wrapper ``scripts/soltest.sh``), as well as command line tests and
compilation tests.

The test system automatically tries to discover the location of
the `evmone <https://github.com/ethereum/evmone/releases>`_ for running the semantic tests.

The ``evmone`` library must be located in the ``deps`` or ``deps/lib`` directory relative to the
current working directory, to its parent or its parent's parent. Alternatively an explicit location
for the ``evmone`` shared object can be specified via the ``ETH_EVMONE`` environment variable.

``evmone`` is needed mainly for running semantic and gas tests.
If you do not have it installed, you can skip these tests by passing the ``--no-semantic-tests``
flag to ``scripts/soltest.sh``.

Running Ewasm tests is disabled by default and can be explicitly enabled
via ``./scripts/soltest.sh --ewasm`` and requires `hera <https://github.com/ewasm/hera>`_
to be found by ``soltest``.
The mechanism for locating the ``hera`` library is the same as for ``evmone``, except that the
variable for specifying an explicit location is called ``ETH_HERA``.

The ``evmone`` and ``hera`` libraries should both end with the file name
extension ``.so`` on Linux, ``.dll`` on Windows systems and ``.dylib`` on macOS.

For running SMT tests, the ``libz3`` library must be installed and locatable
by ``cmake`` during compiler configure stage.

If the ``libz3`` library is not installed on your system, you should disable the
SMT tests by exporting ``SMT_FLAGS=--no-smt`` before running ``./scripts/tests.sh`` or
running ``./scripts/soltest.sh --no-smt``.
These tests are ``libsolidity/smtCheckerTests`` and ``libsolidity/smtCheckerTestsJSON``.

.. note ::

    To get a list of all unit tests run by Soltest, run ``./build/test/soltest --list_content=HRF``.

For quicker results you can run a subset of, or specific tests.

To run a subset of tests, you can use filters:
``./scripts/soltest.sh -t TestSuite/TestName``,
where ``TestName`` can be a wildcard ``*``.

Or, for example, to run all the tests for the yul disambiguator:
``./scripts/soltest.sh -t "yulOptimizerTests/disambiguator/*" --no-smt``.

``./build/test/soltest --help`` has extensive help on all of the options available.

See especially:

- `show_progress (-p) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/show_progress.html>`_ to show test completion,
- `run_test (-t) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/run_test.html>`_ to run specific tests cases, and
- `report-level (-r) <https://www.boost.org/doc/libs/release/libs/test/doc/html/boost_test/utf_reference/rt_param_reference/report_level.html>`_ give a more detailed report.

.. note ::

    Those working in a Windows environment wanting to run the above basic sets
    without libz3. Using Git Bash, you use: ``./build/test/Release/soltest.exe -- --no-smt``.
    If you are running this in plain Command Prompt, use ``.\build\test\Release\soltest.exe -- --no-smt``.

If you want to debug using GDB, make sure you build differently than the "usual".
For example, you could run the following command in your ``build`` folder:
.. code-block:: bash

   cmake -DCMAKE_BUILD_TYPE=Debug ..
   make

This creates symbols so that when you debug a test using the ``--debug`` flag,
you have access to functions and variables in which you can break or print with.

The CI runs additional tests (including ``solc-js`` and testing third party Solidity
frameworks) that require compiling the Emscripten target.

Writing and Running Syntax Tests
--------------------------------

Syntax tests check that the compiler generates the correct error messages for invalid code
and properly accepts valid code.
They are stored in individual files inside the ``tests/libsolidity/syntaxTests`` folder.
These files must contain annotations, stating the expected result(s) of the respective test.
The test suite compiles and checks them against the given expectations.

For example: ``./test/libsolidity/syntaxTests/double_stateVariable_declaration.sol``

.. code-block:: solidity

    contract test {
        uint256 variable;
        uint128 variable;
    }
    // ----
    // DeclarationError: (36-52): Identifier already declared.

A syntax test must contain at least the contract under test itself, followed by the separator ``// ----``. The comments that follow the separator are used to describe the
expected compiler errors or warnings. The number range denotes the location in the source where the error occurred.
If you want the contract to compile without any errors or warning you can leave
out the separator and the comments that follow it.

In the above example, the state variable ``variable`` was declared twice, which is not allowed. This results in a ``DeclarationError`` stating that the identifier was already declared.

The ``isoltest`` tool is used for these tests and you can find it under ``./build/test/tools/``. It is an interactive tool which allows
editing of failing contracts using your preferred text editor. Let's try to break this test by removing the second declaration of ``variable``:

.. code-block:: solidity

    contract test {
        uint256 variable;
    }
    // ----
    // DeclarationError: (36-52): Identifier already declared.

Running ``./build/test/tools/isoltest`` again results in a test failure:

.. code-block:: text

    syntaxTests/double_stateVariable_declaration.sol: FAIL
        Contract:
            contract test {
                uint256 variable;
            }

        Expected result:
            DeclarationError: (36-52): Identifier already declared.
        Obtained result:
            Success


``isoltest`` prints the expected result next to the obtained result, and also
provides a way to edit, update or skip the current contract file, or quit the application.

It offers several options for failing tests:

- ``edit``: ``isoltest`` tries to open the contract in an editor so you can adjust it. It either uses the editor given on the command line (as ``isoltest --editor /path/to/editor``), in the environment variable ``EDITOR`` or just ``/usr/bin/editor`` (in that order).
- ``update``: Updates the expectations for contract under test. This updates the annotations by removing unmet expectations and adding missing expectations. The test is then run again.
- ``skip``: Skips the execution of this particular test.
- ``quit``: Quits ``isoltest``.

All of these options apply to the current contract, expect ``quit`` which stops the entire testing process.

Automatically updating the test above changes it to

.. code-block:: solidity

    contract test {
        uint256 variable;
    }
    // ----

and re-run the test. It now passes again:

.. code-block:: text

    Re-running test case...
    syntaxTests/double_stateVariable_declaration.sol: OK


.. note::

    Choose a name for the contract file that explains what it tests, e.g. ``double_variable_declaration.sol``.
    Do not put more than one contract into a single file, unless you are testing inheritance or cross-contract calls.
    Each file should test one aspect of your new feature.


Running the Fuzzer via AFL
==========================

Fuzzing is a technique that runs programs on more or less random inputs to find exceptional execution
states (segmentation faults, exceptions, etc). Modern fuzzers are clever and run a directed search
inside the input. We have a specialized binary called ``solfuzzer`` which takes source code as input
and fails whenever it encounters an internal compiler error, segmentation fault or similar, but
does not fail if e.g., the code contains an error. This way, fuzzing tools can find internal problems in the compiler.

We mainly use `AFL <https://lcamtuf.coredump.cx/afl/>`_ for fuzzing. You need to download and
install the AFL packages from your repositories (afl, afl-clang) or build them manually.
Next, build Solidity (or just the ``solfuzzer`` binary) with AFL as your compiler:

.. code-block:: bash

    cd build
    # if needed
    make clean
    cmake .. -DCMAKE_C_COMPILER=path/to/afl-gcc -DCMAKE_CXX_COMPILER=path/to/afl-g++
    make solfuzzer

At this stage you should be able to see a message similar to the following:

.. code-block:: text

    Scanning dependencies of target solfuzzer
    [ 98%] Building CXX object test/tools/CMakeFiles/solfuzzer.dir/fuzzer.cpp.o
    afl-cc 2.52b by <lcamtuf@google.com>
    afl-as 2.52b by <lcamtuf@google.com>
    [+] Instrumented 1949 locations (64-bit, non-hardened mode, ratio 100%).
    [100%] Linking CXX executable solfuzzer

If the instrumentation messages did not appear, try switching the cmake flags pointing to AFL's clang binaries:

.. code-block:: bash

    # if previously failed
    make clean
    cmake .. -DCMAKE_C_COMPILER=path/to/afl-clang -DCMAKE_CXX_COMPILER=path/to/afl-clang++
    make solfuzzer

Otherwise, upon execution the fuzzer halts with an error saying binary is not instrumented:

.. code-block:: text

    afl-fuzz 2.52b by <lcamtuf@google.com>
    ... (truncated messages)
    [*] Validating target binary...

    [-] Looks like the target binary is not instrumented! The fuzzer depends on
        compile-time instrumentation to isolate interesting test cases while
        mutating the input data. For more information, and for tips on how to
        instrument binaries, please see /usr/share/doc/afl-doc/docs/README.

        When source code is not available, you may be able to leverage QEMU
        mode support. Consult the README for tips on how to enable this.
        (It is also possible to use afl-fuzz as a traditional, "dumb" fuzzer.
        For that, you can use the -n option - but expect much worse results.)

    [-] PROGRAM ABORT : No instrumentation detected
             Location : check_binary(), afl-fuzz.c:6920


Next, you need some example source files. This makes it much easier for the fuzzer
to find errors. You can either copy some files from the syntax tests or extract test files
from the documentation or the other tests:

.. code-block:: bash

    mkdir /tmp/test_cases
    cd /tmp/test_cases
    # extract from tests:
    path/to/solidity/scripts/isolate_tests.py path/to/solidity/test/libsolidity/SolidityEndToEndTest.cpp
    # extract from documentation:
    path/to/solidity/scripts/isolate_tests.py path/to/solidity/docs

The AFL documentation states that the corpus (the initial input files) should not be
too large. The files themselves should not be larger than 1 kB and there should be
at most one input file per functionality, so better start with a small number of.
There is also a tool called ``afl-cmin`` that can trim input files
that result in similar behaviour of the binary.

Now run the fuzzer (the ``-m`` extends the size of memory to 60 MB):

.. code-block:: bash

    afl-fuzz -m 60 -i /tmp/test_cases -o /tmp/fuzzer_reports -- /path/to/solfuzzer

The fuzzer creates source files that lead to failures in ``/tmp/fuzzer_reports``.
Often it finds many similar source files that produce the same error. You can
use the tool ``scripts/uniqueErrors.sh`` to filter out the unique errors.

Whiskers
========

*Whiskers* is a string templating system similar to `Mustache <https://mustache.github.io>`_. It is used by the
compiler in various places to aid readability, and thus maintainability and verifiability, of the code.

The syntax comes with a substantial difference to Mustache. The template markers ``{{`` and ``}}`` are
replaced by ``<`` and ``>`` in order to aid parsing and avoid conflicts with :ref:`yul`
(The symbols ``<`` and ``>`` are invalid in inline assembly, while ``{`` and ``}`` are used to delimit blocks).
Another limitation is that lists are only resolved one depth and they do not recurse. This may change in the future.

A rough specification is the following:

Any occurrence of ``<name>`` is replaced by the string-value of the supplied variable ``name`` without any
escaping and without iterated replacements. An area can be delimited by ``<#name>...</name>``. It is replaced
by as many concatenations of its contents as there were sets of variables supplied to the template system,
each time replacing any ``<inner>`` items by their respective value. Top-level variables can also be used
inside such areas.

There are also conditionals of the form ``<?name>...<!name>...</name>``, where template replacements
continue recursively either in the first or the second segment depending on the value of the boolean
parameter ``name``. If ``<?+name>...<!+name>...</+name>`` is used, then the check is whether
the string parameter ``name`` is non-empty.

.. _documentation-style:

Documentation Style Guide
=========================

In the following section you find style recommendations specifically focusing on documentation
contributions to Solidity.

English Language
----------------

Use English, with British English spelling preferred, unless using project or brand names. Try to reduce the usage of
local slang and references, making your language as clear to all readers as possible. Below are some references to help:

* `Simplified technical English <https://en.wikipedia.org/wiki/Simplified_Technical_English>`_
* `International English <https://en.wikipedia.org/wiki/International_English>`_
* `British English spelling <https://en.oxforddictionaries.com/spelling/british-and-spelling>`_


.. note::

    While the official Solidity documentation is written in English, there are community contributed :ref:`translations`
    in other languages available. Please refer to the `translation guide <https://github.com/solidity-docs/translation-guide>`_
    for information on how to contribute to the community translations.

Title Case for Headings
-----------------------

Use `title case <https://titlecase.com>`_ for headings. This means capitalise all principal words in
titles, but not articles, conjunctions, and prepositions unless they start the
title.

For example, the following are all correct:

* Title Case for Headings.
* For Headings Use Title Case.
* Local and State Variable Names.
* Order of Layout.

Expand Contractions
-------------------

Use expanded contractions for words, for example:

* "Do not" instead of "Don't".
* "Can not" instead of "Can't".

Active and Passive Voice
------------------------

Active voice is typically recommended for tutorial style documentation as it
helps the reader understand who or what is performing a task. However, as the
Solidity documentation is a mixture of tutorials and reference content, passive
voice is sometimes more applicable.

As a summary:

* Use passive voice for technical reference, for example language definition and internals of the Ethereum VM.
* Use active voice when describing recommendations on how to apply an aspect of Solidity.

For example, the below is in passive voice as it specifies an aspect of Solidity:

  Functions can be declared ``pure`` in which case they promise not to read
  from or modify the state.

For example, the below is in active voice as it discusses an application of Solidity:

  When invoking the compiler, you can specify how to discover the first element
  of a path, and also path prefix remappings.

Common Terms
------------

* "Function parameters" and "return variables", not input and output parameters.

Code Examples
-------------

A CI process tests all code block formatted code examples that begin with ``pragma solidity``, ``contract``, ``library``
or ``interface`` using the ``./test/cmdlineTests.sh`` script when you create a PR. If you are adding new code examples,
ensure they work and pass tests before creating the PR.

Ensure that all code examples begin with a ``pragma`` version that spans the largest where the contract code is valid.
For example ``pragma solidity >=0.4.0 <0.9.0;``.

Running Documentation Tests
---------------------------

Make sure your contributions pass our documentation tests by running ``./docs/docs.sh`` that installs dependencies
needed for documentation and checks for any problems such as broken links or syntax issues.

Solidity Language Design
========================

To actively get involved in the language design process and share your ideas concerning the future of Solidity,
please join the `Solidity forum <https://forum.soliditylang.org/>`_.

The Solidity forum serves as the place to propose and discuss new language features and their implementation in
the early stages of ideation or modifications of existing features.

As soon as proposals get more tangible, their
implementation will also be discussed in the `Solidity GitHub repository <https://github.com/ethereum/solidity>`_
in the form of issues.

In addition to the forum and issue discussions, we regularly host language design discussion calls in which selected
topics, issues or feature implementations are debated in detail. The invitation to those calls is shared via the forum.

We are also sharing feedback surveys and other content that is relevant to language design in the forum.

If you want to know where the team is standing in terms or implementing new features, you can follow the implementation status in the `Solidity Github project <https://github.com/ethereum/solidity/projects/43>`_.
Issues in the design backlog need further specification and will either be discussed in a language design call or in a regular team call. You can
see the upcoming changes for the next breaking release by changing from the default branch (`develop`) to the `breaking branch <https://github.com/ethereum/solidity/tree/breaking>`_.

For ad-hoc cases and questions you can reach out to us via the `Solidity-dev Gitter channel <https://gitter.im/ethereum/solidity-dev>`_, a
dedicated chatroom for conversations around the Solidity compiler and language development.

We are happy to hear your thoughts on how we can improve the language design process to be even more collaborative and transparent.
>>>>>>> 84c64edfeea316f2d5bc6b4f0f392301a53061b6
