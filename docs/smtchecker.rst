<<<<<<< HEAD
.. _formal_verification:

##################################
SMTChecker dan Verifikasi Formal
##################################

Dengan menggunakan verifikasi formal, dimungkinkan untuk melakukan pembuktian matematis
otomatis bahwa kode sumber Anda memenuhi spesifikasi formal tertentu.
Spesifikasinya masih formal (seperti kode sumber), tetapi biasanya jauh lebih sederhana.

Perhatikan bahwa verifikasi formal itu sendiri hanya dapat membantu Anda memahami perbedaan
antara apa yang Anda lakukan (spesifikasi) dan bagaimana Anda melakukannya (implementasi
sebenarnya). Anda masih perlu memeriksa apakah spesifikasinya adalah apa yang Anda inginkan
dan Anda tidak melewatkan efek yang tidak diinginkan.

Solidity menerapkan pendekatan verifikasi formal berdasarkan
`SMT (Satisfiability Modulo Theories) <https://en.wikipedia.org/wiki/Satisfiability_modulo_theories>`_ dan
penyelesaian `Horn <https://en.wikipedia.org/wiki/Horn-satisfiability>`_ .
Modul SMTChecker secara otomatis mencoba membuktikan bahwa kode memenuhi spesifikasi
yang diberikan oleh pernyataan ``require`` dan ``assert``. Artinya, ia menganggap
pernyataan ``require`` sebagai asumsi dan mencoba membuktikan bahwa kondisi di dalam
pernyataan ``assert`` selalu benar. Jika kegagalan pernyataan ditemukan, contoh tandingan
dapat diberikan kepada pengguna yang menunjukkan bagaimana pernyataan dapat dilanggar.
Jika tidak ada peringatan yang diberikan oleh SMTChecker untuk suatu properti, berarti properti
tersebut aman.

Target verifikasi lain yang diperiksa SMTChecker pada waktu kompilasi adalah:

- Aritmatika underflow dan overflow.
- Division by zero.
- Kondisi Trivial dan unreachable code.
- Memunculkan array kosong.
- Akses indeks di luar batas.
- Dana tidak cukup untuk transfer.

Semua target di atas secara otomatis diperiksa secara default jika semua mesin
diaktifkan, kecuali underflow dan overflow untuk Solidity >=0.8.7.

Peringatan potensial yang dilaporkan SMTChecker adalah:

- ``<failing  property> happens here.``. Ini berarti bahwa SMTChecker membuktikan bahwa properti tertentu gagal. Sebuah contoh tandingan dapat diberikan, namun dalam situasi yang kompleks mungkin juga tidak menunjukkan contoh tandingan. Hasil ini mungkin juga positif palsu dalam kasus tertentu, ketika penyandian SMT menambahkan abstraksi untuk kode Solidity yang sulit atau tidak mungkin untuk diungkapkan.
- ``<failing property> might happen here``. Ini berarti bahwa solver tidak dapat membuktikan kedua kasus dalam batas waktu yang diberikan. Karena hasilnya tidak diketahui, SMTChecker melaporkan potensi kegagalan untuk kesehatan. Ini dapat diselesaikan dengan meningkatkan batas waktu kueri, tetapi masalahnya mungkin juga terlalu sulit untuk dipecahkan oleh mesin.

Untuk mengaktifkan SMTChecker, Anda harus memilih :ref:`mesin mana yang harus dijalankan<smtchecker_engines>`,
di mana defaultnya adalah tidak ada mesin. Memilih mesin memungkinkan SMTChecker pada semua file.

.. note::

    Sebelum Solidity 0.8.4, cara default untuk mengaktifkan SMTChecker adalah melalui
    ``pragma experimental SMTTChecker;`` dan hanya kontrak yang berisi pragma yang akan
    dianalisis. Pragma itu sudah tidak digunakan lagi, dan meskipun masih memungkinkan
    SMTChecker untuk kompatibilitas ke belakang, pragma itu akan dihapus di Solidity 0.9.0.
    Perhatikan juga bahwa sekarang menggunakan pragma bahkan hanya dalam satu file akan
    memungkinkan SMTChecker untuk semua file.

.. note::

    Kurangnya peringatan untuk target verifikasi menunjukkan bukti kebenaran matematis yang
    tak terbantahkan, dengan asumsi tidak ada bug di SMTChecker dan pemecah yang mendasarinya.
    Perlu diingat bahwa masalah ini *sangat sulit* dan terkadang *tidak mungkin* untuk diselesaikan
    secara otomatis dalam kasus umum. Oleh karena itu, beberapa properti mungkin tidak dapat
    diselesaikan atau mungkin mengarah pada kesalahan positif untuk kontrak besar. Setiap properti
    yang telah terbukti harus dilihat sebagai pencapaian penting. Untuk pengguna tingkat lanjut,
    lihat :ref:`SMTChecker Tuning <smtchecker_options>` untuk mempelajari beberapa opsi yang mungkin
    membantu membuktikan properti yang lebih kompleks.

********
Tutorial
********

Overflow
========

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Overflow {
        uint immutable x;
        uint immutable y;

        function add(uint _x, uint _y) internal pure returns (uint) {
            return _x + _y;
        }

        constructor(uint _x, uint _y) {
            (x, y) = (_x, _y);
        }

        function stateAdd() public view returns (uint) {
            return add(x, y);
        }
    }

Kontrak di atas menunjukkan contoh cek overflow.
SMTChecker tidak memeriksa underflow dan overflow secara default untuk Solidity >=0.8.7,
jadi kita perlu menggunakan opsi baris perintah ``--model-checker-targets "underflow,overflow"``
atau opsi JSON ``settings.modelChecker.targets = ["underflow", "overflow"]``.
Lihat :ref:`bagian ini untuk konfigurasi target<smtchecker_targets>`.
Di sini, ia melaporkan hal berikut:

.. code-block:: text

    Warning: CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
    Counterexample:
    x = 1, y = 115792089237316195423570985008687907853269984665640564039457584007913129639935
     = 0

    Transaction trace:
    Overflow.constructor(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935)
    State: x = 1, y = 115792089237316195423570985008687907853269984665640564039457584007913129639935
    Overflow.stateAdd()
        Overflow.add(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935) -- internal call
     --> o.sol:9:20:
      |
    9 |             return _x + _y;
      |                    ^^^^^^^

Jika kita menambahkan pernyataan ``require`` yang memfilter kasus overflow,
SMTChecker membuktikan bahwa tidak ada overflow yang dapat dijangkau (dengan tidak melaporkan peringatan):

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Overflow {
        uint immutable x;
        uint immutable y;

        function add(uint _x, uint _y) internal pure returns (uint) {
            return _x + _y;
        }

        constructor(uint _x, uint _y) {
            (x, y) = (_x, _y);
        }

        function stateAdd() public view returns (uint) {
            require(x < type(uint128).max);
            require(y < type(uint128).max);
            return add(x, y);
        }
    }


Assert
======

Sebuah pernyataan mewakili invarian dalam kode Anda: sebuah properti yang harus benar
*untuk semua transaksi, termasuk semua nilai input dan penyimpanan*, jika tidak ada bug.

Kode di bawah ini mendefinisikan fungsi ``f`` yang menjamin tidak ada overflow.
Fungsi ``inv`` mendefinisikan spesifikasi bahwa ``f`` meningkat secara monoton:
untuk setiap kemungkinan pasangan ``(_a, _b)``, jika ``_b > _a`` maka ``f(_b) > f(_a)``.
Karena ``f`` memang meningkat secara monoton, SMTChecker membuktikan bahwa properti kita benar.
Anda didorong untuk bermain dengan properti dan definisi fungsi untuk melihat hasil apa yang keluar!

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Monotonic {
        function f(uint _x) internal pure returns (uint) {
            require(_x < type(uint128).max);
            return _x * 42;
        }

        function inv(uint _a, uint _b) public pure {
            require(_b > _a);
            assert(f(_b) > f(_a));
        }
    }

Kami juga dapat menambahkan pernyataan di dalam loop untuk memverifikasi properti yang lebih rumit.
Kode berikut mencari elemen maksimum dari array angka yang tidak
dibatasi, dan menegaskan properti bahwa elemen yang ditemukan harus lebih besar atau
sama dengan setiap elemen dalam array.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Max {
        function max(uint[] memory _a) public pure returns (uint) {
            uint m = 0;
            for (uint i = 0; i < _a.length; ++i)
                if (_a[i] > m)
                    m = _a[i];

            for (uint i = 0; i < _a.length; ++i)
                assert(m >= _a[i]);

            return m;
        }
    }

Perhatikan bahwa dalam contoh ini SMTChecker akan secara otomatis mencoba membuktikan tiga properti:

1. ``++i``di loop pertama tidak overflow.
2. ``++i`` di loop kedua tidak overflow.
3. assertion selalu true.

.. note::

    Properti melibatkan loop, yang membuatnya *jauh* lebih sulit dari sebelumnya
    contoh, jadi waspadalah terhadap loop!

Semua properti benar terbukti aman. Jangan ragu untuk mengubah
properties dan/atau tambahkan batasan pada array untuk melihat hasil yang berbeda.
Misalnya, mengubah kode menjadi

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Max {
        function max(uint[] memory _a) public pure returns (uint) {
            require(_a.length >= 5);
            uint m = 0;
            for (uint i = 0; i < _a.length; ++i)
                if (_a[i] > m)
                    m = _a[i];

            for (uint i = 0; i < _a.length; ++i)
                assert(m > _a[i]);

            return m;
        }
    }

memberi kita:

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:

    _a = [0, 0, 0, 0, 0]
     = 0

    Transaction trace:
    Test.constructor()
    Test.max([0, 0, 0, 0, 0])
      --> max.sol:14:4:
       |
    14 |            assert(m > _a[i]);


State Properties
================

Sejauh ini contoh-contoh hanya menunjukkan penggunaan SMTTChecker di atas kode murni,
membuktikan properti tentang operasi atau algoritma tertentu.
Jenis properti umum dalam kontrak pintar adalah properti yang melibatkan
status kontrak. Beberapa transaksi mungkin diperlukan untuk membuat *assertion*
gagal untuk properti seperti itu.

Sebagai contoh, perhatikan grid 2D di mana kedua sumbu memiliki koordinat dalam rentang (-2^128, 2^128 - 1).
Mari kita tempatkan robot pada posisi (0, 0). Robot hanya bisa bergerak secara diagonal, selangkah demi selangkah,
dan tidak bisa bergerak di luar grid. Mesin state robot dapat diwakili oleh kontrak pintar
di bawah.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Robot {
        int x = 0;
        int y = 0;

        modifier wall {
            require(x > type(int128).min && x < type(int128).max);
            require(y > type(int128).min && y < type(int128).max);
            _;
        }

        function moveLeftUp() wall public {
            --x;
            ++y;
        }

        function moveLeftDown() wall public {
            --x;
            --y;
        }

        function moveRightUp() wall public {
            ++x;
            ++y;
        }

        function moveRightDown() wall public {
            ++x;
            --y;
        }

        function inv() public view {
            assert((x + y) % 2 == 0);
        }
    }

Fungsi ``inv`` merepresentasikan invarian dari mesin state bahwa ``x + y``
harus genap.
SMTChecker berhasil membuktikan bahwa terlepas dari berapa banyak perintah yang kita berikan kepada
robot, bahkan jika jumlahnya tak terhingga, invarian *tidak akan pernah* gagal. Pembaca yang
tertarik mungkin ingin membuktikan fakta itu secara manual. Petunjuk: invarian ini adalah
induktif.

Kita juga dapat mengelabui SMTChecker agar memberi kita jalur ke posisi tertentu
yang menurut kita dapat dijangkau. Kita dapat menambahkan properti yang (2, 4) *not*
reachable, dengan menambahkan fungsi berikut.

.. code-block:: Solidity

    function reach_2_4() public view {
        assert(!(x == 2 && y == 4));
    }

Properti ini salah, dan sambil membuktikan bahwa properti itu salah,
SMTChecker memberi tahu kita dengan tepat *bagaimana* mencapainya (2, 4):

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:
    x = 2, y = 4

    Transaction trace:
    Robot.constructor()
    State: x = 0, y = 0
    Robot.moveLeftUp()
    State: x = (- 1), y = 1
    Robot.moveRightUp()
    State: x = 0, y = 2
    Robot.moveRightUp()
    State: x = 1, y = 3
    Robot.moveRightUp()
    State: x = 2, y = 4
    Robot.reach_2_4()
      --> r.sol:35:4:
       |
    35 |            assert(!(x == 2 && y == 4));
       |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^

Perhatikan bahwa jalur di atas belum tentu deterministik, karena ada
jalur lain yang bisa dijangkau (2, 4). Pilihan jalur mana yang ditampilkan
mungkin berubah tergantung pada pemecah yang digunakan, versinya, atau hanya secara acak.

External Call dan Reentrancy
=============================

Setiap panggilan eksternal diperlakukan sebagai panggilan ke kode yang tidak dikenal oleh SMTChecker.
Alasan di balik itu adalah bahwa meskipun kode kontrak yang dipanggil tersedia pada
waktu kompilasi, tidak ada jaminan bahwa kontrak yang digunakan memang akan sama
dengan kontrak dari mana antarmuka berasal pada waktu kompilasi.

Dalam beberapa kasus, dimungkinkan untuk secara otomatis menyimpulkan properti atas
variabel state yang masih benar bahkan jika kode yang dipanggil secara eksternal dapat
melakukan apa saja, termasuk memasukkan kembali kontrak pemanggil.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    interface Unknown {
        function run() external;
    }

    contract Mutex {
        uint x;
        bool lock;

        Unknown immutable unknown;

        constructor(Unknown _u) {
            require(address(_u) != address(0));
            unknown = _u;
        }

        modifier mutex {
            require(!lock);
            lock = true;
            _;
            lock = false;
        }

        function set(uint _x) mutex public {
            x = _x;
        }

        function run() mutex public {
            uint xPre = x;
            unknown.run();
            assert(xPre == x);
        }
    }

Contoh di atas menunjukkan kontrak yang menggunakan flag mutex untuk melarang reentrancy.
Solver dapat menyimpulkan bahwa ketika ``unknown.run()`` dipanggil, kontrak
sudah "dikunci", jadi tidak mungkin mengubah nilai ``x``,
terlepas dari apa yang dilakukan kode yang tidak dikenal.

Jika kita "lupa" untuk menggunakan pengubah ``mutex`` pada fungsi ``set``,
SMTChecker dapat mensintesis perilaku kode yang dipanggil secara eksternal
sehingga pernyataan gagal:

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:
    x = 1, lock = true, unknown = 1

    Transaction trace:
    Mutex.constructor(1)
    State: x = 0, lock = false, unknown = 1
    Mutex.run()
        unknown.run() -- untrusted external call, synthesized as:
            Mutex.set(1) -- reentrant call
      --> m.sol:32:3:
       |
    32 | 		assert(xPre == x);
       | 		^^^^^^^^^^^^^^^^^


.. _smtchecker_options:

*****************************
SMTChecker Options and Tuning
*****************************

Timeout
=======

SMTChecker menggunakan batas sumber daya hardcoded (``rlimit``) yang dipilih per pemecah,
yang tidak secara tepat terkait dengan waktu. Kami memilih opsi ``rlimit`` sebagai default
karena memberikan lebih banyak jaminan determinisme daripada waktu di dalam solver.

Opsi ini diterjemahkan secara kasar menjadi "batas waktu beberapa detik" per kueri. Tentu saja
banyak sifat yang sangat kompleks dan membutuhkan banyak waktu untuk diselesaikan, di mana
determinisme tidak menjadi masalah. Jika SMTChecker tidak berhasil menyelesaikan properti
kontrak dengan default ``rlimit``, batas waktu dapat diberikan dalam milidetik melalui opsi
CLI ``--model-checker-timeout <time>`` atau opsi JSON ``settings.modelChecker.timeout=<time>``,
di mana 0 berarti tidak ada batas waktu.

.. _smtchecker_targets:

Target Verifikasi
=================

Jenis target verifikasi yang dibuat oleh SMTChecker juga dapat
dikustomisasi melalui opsi CLI ``--model-checker-target <targets>`` atau JSON
opsi ``settings.modelChecker.targets=<targets>``.
Dalam kasus CLI, ``<targets>`` adalah daftar tanpa spasi-koma-dipisahkan dari satu atau
lebih banyak target verifikasi, dan array dari satu atau lebih target sebagai string dalam
input JSON.
Kata kunci yang mewakili target adalah:

- Pernyataan: ``assert``.
- Aritmatika underflow: ``underflow``.
- Aritmatika overflow: ``overflow``.
- Pembagian dengan nol: ``divByZero``.
- Kondisi sepele dan kode yang tidak dapat dijangkau: ``constantCondition``.
- Memunculkan array kosong: ``popEmptyArray``.
- Akses indeks array/byte tetap di luar batas: ``outOfBounds``.
- Dana tidak mencukupi untuk transfer: ``balance``.
- Semua hal di atas: ``default`` (khusus CLI).

Subset umum dari target mungkin, misalnya:
``--model-checker-targets assert,overflow``.

Semua target diperiksa secara default, kecuali underflow dan overflow untuk Solidity >=0.8.7.

Tidak ada heuristik yang tepat tentang bagaimana dan kapan harus membagi target verifikasi,
tetapi dapat berguna terutama ketika berhadapan dengan kontrak besar.

Unproved Targets
================

Jika ada target yang belum terbukti, SMTTChecker mengeluarkan satu peringatan yang menyatakan:
berapa banyak target yang belum terbukti. Jika pengguna ingin melihat semua target
spesifik yang belum terbukti, opsi CLI ``--model-checker-show-unproved`` dan
opsi JSON ``settings.modelChecker.showUnproved = true`` dapat digunakan.

Kontrak Terverifikasi
=====================

Secara default, semua kontrak yang dapat di-deploy dalam sumber yang diberikan dianalisis secara
terpisah sebagai kontrak yang akan di-deploy. Artinya, jika suatu kontrak memiliki banyak
*inheritance parents* langsung dan tidak langsung, semuanya akan dianalisis sendiri-sendiri,
meskipun hanya yang paling turunan yang akan diakses langsung di blockchain. Hal ini menyebabkan
beban yang tidak perlu pada SMTChecker dan solver. Untuk membantu kasus seperti ini, pengguna
dapat menentukan kontrak mana yang harus dianalisis sebagai kontrak yang diterapkan.
Kontrak induk tentu saja masih dianalisis, tetapi hanya dalam konteks kontrak yang paling diturunkan,
mengurangi kerumitan pengkodean dan kueri yang dihasilkan. Perhatikan bahwa kontrak abstrak secara
default tidak dianalisis sebagai kontrak yang paling diturunkan oleh SMTChecker.

Kontrak yang dipilih dapat diberikan melalui daftar yang dipisahkan koma (whitespace
tidak diperbolehkan) dari pasangan <source>:<contract> di CLI:
``--model-checker-contracts "<source1.sol:contract1>,<source2.sol:contract2>,<source2.sol:contract3>"``,
dan melalui objek ``settings.modelChecker.contracts`` di :ref:`JSON input<compiler-api>`,
yang memiliki bentuk sebagai berikut:

.. code-block:: json

    "contracts": {
        "source1.sol": ["contract1"],
        "source2.sol": ["contract2", "contract3"]
    }

Invarian Inductive Inferred yang Dilaporkan
===========================================

Untuk properti yang terbukti aman dengan mesin CHC,
SMTChecker dapat mengambil invarian induktif yang disimpulkan oleh Horn
solver sebagai bagian dari pembuktian.
Saat ini dua jenis invarian dapat dilaporkan kepada pengguna:

- Contract Invariants: ini adalah properti di atas variabel state kontrak yang benar sebelum dan sesudah setiap
  kemungkinan transaksi yang mungkin pernah dijalankan oleh kontrak. Misalnya, ``x >= y``, di mana ``x`` dan ``y`` adalah variabel status kontrak.
- Reentrancy Properties: mereka mewakili perilaku kontrak di hadapan panggilan eksternal ke kode yang tidak dikenal.
  Properti ini dapat mengekspresikan hubungan antara nilai variabel state sebelum dan sesudah panggilan eksternal, di mana panggilan eksternal bebas
  untuk melakukan apa saja, termasuk membuat panggilan masuk kembali ke kontrak yang dianalisis. Variabel prima mewakili nilai variabel state setelah panggilan eksternal tersebut. Contoh: ``lock -> x = x'``.

Pengguna dapat memilih jenis invarian yang akan dilaporkan menggunakan opsi CLI ``--model-checker-invariants "contract,reentrancy"`` atau sebagai array di bidang ``settings.modelChecker.invariants`` di : ref:`JSON input<compiler-api>`.
Secara default, SMTChecker tidak melaporkan invarian.

Division dan Modulo dengan Slack Variables
==========================================

Spacer, Horn solver default yang digunakan oleh SMTTChecker, sering kali tidak menyukai operasi division
dan modulo di dalam aturan Horn. Karena itu, secara default divisi Solidity dan operasi modulo
dikodekan menggunakan batasan ``a = b * d + m`` di mana ``d = a / b`` dan ``m = a % b``.
Namun, solver lain, seperti Eldarica, lebih menyukai operasi sintaksis yang tepat.
Command line flag ``--model-checker-div-mod-no-slacks`` dan opsi JSON
``settings.modelChecker.divModNoSlacks`` dapat digunakan untuk mengaktifkan pengkodean
tergantung pada preferensi solver yang digunakan.

Abstraksi Fungsi Natspec
========================

Fungsi tertentu termasuk metode matematika umum seperti ``pow``
dan ``sqrt`` mungkin terlalu rumit untuk dianalisis dengan cara yang sepenuhnya otomatis.
Fungsi-fungsi ini dapat dijelaskan dengan tag Natspec yang menunjukkan ke
SMTChecker bahwa fungsi-fungsi ini harus diabstraksikan. Ini berarti bahwa
badan fungsi tidak digunakan, dan ketika dipanggil, fungsi akan:

- Kembalikan nilai nondeterministik, dan pertahankan variabel status tidak berubah jika fungsi yang diabstraksi adalah tampilan/murni, atau juga atur variabel status ke nilai nondeterministik sebaliknya. Ini dapat digunakan melalui anotasi ``/// @custom:smtchecker abstract-function-nondet``.
- Bertindak sebagai fungsi yang tidak diinterpretasikan. Ini berarti bahwa semantik fungsi (diberikan oleh tubuh) diabaikan, dan satu-satunya properti yang dimiliki fungsi ini adalah bahwa dengan input yang sama, itu menjamin output yang sama. Ini sedang dalam pengembangan dan akan tersedia melalui anotasi ``/// @custom:smtchecker abstract-function-uf``.

.. _smtchecker_engines:

Model Checking Engines
======================

Modul SMTChecker mengimplementasikan dua mesin penalaran yang berbeda, sebuah Bounded
Model Checker (BMC) dan sistem Constrained Horn Clauses (CHC). Kedua
mesin sedang dalam pengembangan, dan memiliki karakteristik yang berbeda.
Mesinnya independen dan setiap peringatan properti menyatakan dari mesin mana
itu datang. Perhatikan bahwa semua contoh di atas dengan contoh tandingan
dilaporkan oleh CHC, mesin yang lebih kuat.

Secara default kedua mesin digunakan, di mana CHC berjalan lebih dulu, dan setiap properti yang
tidak terbukti diteruskan ke BMC. Anda dapat memilih mesin tertentu melalui opsi
CLI ``--model-checker-engine {all,bmc,chc,none}`` atau opsi JSON
``settings.modelChecker.engine={all,bmc,chc,none}``.

Bounded Model Checker (BMC)
---------------------------

Mesin BMC menganalisis fungsi secara terpisah, yaitu, tidak memerlukan
perilaku kontrak secara keseluruhan atas beberapa transaksi ketika
menganalisis setiap fungsi. Loop juga diabaikan dalam mesin ini saat ini.
Panggilan fungsi internal disejajarkan asalkan tidak rekursif, secara langsung
atau tidak langsung. Panggilan fungsi eksternal disejajarkan jika memungkinkan. Pengetahuan
yang berpotensi dipengaruhi oleh reentrancy akan dihapus.

Karakteristik di atas membuat BMC rentan melaporkan false positive,
tetapi juga ringan dan harus dapat dengan cepat menemukan bug lokal kecil.

Constrained Horn Clauses (CHC)
------------------------------

Control Flow Graph (CFG) kontrak dimodelkan sebagai sistem klausa Horn,
di mana siklus hidup kontrak diwakili oleh loop yang dapat mengunjungi
setiap fungsi publik/eksternal secara non-deterministik. Dengan cara ini,
perilaku seluruh kontrak atas jumlah transaksi yang tidak terbatas diperhitungkan
saat menganalisis fungsi apa pun. Loop didukung penuh oleh mesin ini. Panggilan
fungsi internal didukung, dan panggilan fungsi eksternal menganggap kode yang
dipanggil tidak diketahui dan dapat melakukan apa saja.

Mesin CHC jauh lebih bertenaga daripada BMC dalam hal apa yang dapat dibuktikannya,
dan mungkin memerlukan lebih banyak sumber daya komputasi.

SMT dan Horn solvers
====================

Kedua mesin yang dirinci di atas menggunakan pembuktian teorema otomatis sebagai backend
logisnya. BMC menggunakan SMT solver, sedangkan CHC menggunakan Horn solver.
Seringkali alat yang sama dapat bertindak sebagai keduanya, seperti yang terlihat di
`z3 <https://github.com/Z3Prover/z3>`_, yang terutama merupakan pemecah SMT dan membuat
`Spacer <https://spacer.bitbucket.io/>`_ tersedia sebagai Horn solver, dan
`Eldarica <https://github.com/uuverifiers/eldarica>`_ yang melakukan keduanya.

Pengguna dapat memilih pemecah mana yang harus digunakan, jika tersedia, melalui opsi
CLI ``--model-checker-solvers {all,cvc4,smtlib2,z3}`` atau opsi JSON
``settings.modelChecker.solvers=[smtlib2,z3]``, di mana:

- ``cvc4`` hanya tersedia jika biner ``solc`` dikompilasi dengannya. Hanya BMC yang menggunakan ``cvc4``.
- ``smtlib2`` mengeluarkan kueri SMT/Horn dalam format `smtlib2 <http://smtlib.cs.uiowa.edu/>`_.
   Ini dapat digunakan bersama dengan kompiler `callback mechanism <https://github.com/ethereum/solc-js>`_ sehingga
   solver binary apa pun dari sistem dapat digunakan untuk secara sinkron mengembalikan hasil kueri ke kompiler.
   Saat ini satu-satunya cara untuk menggunakan Eldarica, misalnya, karena tidak memiliki C++ API.
   Ini dapat digunakan oleh BMC dan CHC tergantung pada pemecah yang dipanggil.
- ``z3`` tersedia

  - jika ``solc`` dikompilasi dengannya;
  - jika library ``z3`` dinamis versi 4.8.x diinstal di sistem Linux (dari Solidity 0.7.6);
  - secara statis di ``soljson.js`` (dari Solidity 0.6.9), yaitu, biner Javascript dari compiler.

Karena BMC dan CHC menggunakan ``z3``, dan ``z3`` tersedia di lebih banyak variasi lingkungan,
termasuk di browser, sebagian besar pengguna hampir tidak perlu khawatir tentang opsi ini. Pengguna
yang lebih mahir mungkin menerapkan opsi ini untuk mencoba pemecah alternatif pada masalah yang
lebih kompleks.

Harap dicatat bahwa kombinasi tertentu dari mesin dan pemecah yang dipilih akan menyebabkan
SMTChecker tidak melakukan apa-apa, misalnya memilih CHC dan ``cvc4``.

*******************************
Abstraction dan False Positives
*******************************

SMTChecker mengimplementasikan abstraksi dengan cara yang tidak lengkap dan sehat: Jika ada bug
dilaporkan, itu mungkin false positive yang diperkenalkan oleh abstraksi (karena
menghapus pengetahuan atau menggunakan tipe yang tidak tepat). Jika ditentukan bahwa
target verifikasi aman, memang aman, yaitu tidak ada yang false
negatif (kecuali ada bug di SMTTChecker).

Jika target tidak dapat dibuktikan, Anda dapat mencoba membantu solver dengan
menggunakan opsi tuning di bagian sebelumnya.
Jika Anda yakin dengan false positive, tambahkan pernyataan ``require`` dalam kode
dengan lebih banyak informasi juga dapat memberikan lebih banyak kekuatan untuk pemecah.

SMT Encoding dan Types
======================

Pengkodean SMTChecker mencoba setepat mungkin, memetakan tipe Solidity
dan ekspresi ke representasi `SMT-LIB <http://smtlib.cs.uiowa.edu/>`_ terdekat,
seperti yang ditunjukkan pada tabel di bawah.

+-----------------------+--------------------------------+-----------------------------+
|Solidity type          |SMT sort                        |Theories                     |
+=======================+================================+=============================+
|Boolean                |Bool                            |Bool                         |
+-----------------------+--------------------------------+-----------------------------+
|intN, uintN, address,  |Integer                         |LIA, NIA                     |
|bytesN, enum, contract |                                |                             |
+-----------------------+--------------------------------+-----------------------------+
|array, mapping, bytes, |Tuple                           |Datatypes, Arrays, LIA       |
|string                 |(Array elements, Integer length)|                             |
+-----------------------+--------------------------------+-----------------------------+
|struct                 |Tuple                           |Datatypes                    |
+-----------------------+--------------------------------+-----------------------------+
|other types            |Integer                         |LIA                          |
+-----------------------+--------------------------------+-----------------------------+

Jenis yang belum didukung diabstraksikan oleh satu 256-bit unsigned
integer, di mana operasi mereka yang tidak didukung diabaikan.

Untuk detail lebih lanjut tentang bagaimana pengkodean SMT bekerja secara internal, lihat makalah
`Verifikasi Smart Kontrak Solidity berbasis SMT <https://github.com/leonardoalt/text/blob/master/solidity_isola_2018/main.pdf>`_.

Function Calls
==============

Di mesin BMC, panggilan fungsi ke kontrak yang sama (atau kontrak dasar) di-inlined
jika memungkinkan, yaitu saat implementasinya tersedia. Panggilan ke fungsi dalam kontrak lain
tidak di-inlined meskipun kodenya tersedia, karena kami tidak dapat menjamin bahwa kode yang
diterapkan sebenarnya sama.

Mesin CHC membuat klausa Horn nonlinier yang menggunakan ringkasan fungsi yang dipanggil
untuk mendukung panggilan fungsi internal. Panggilan fungsi eksternal diperlakukan
sebagai panggilan ke kode yang tidak dikenal, termasuk reentrant call yang potensial.

Fungsi pure yang kompleks diabstraksikan oleh fungsi yang tidak ditafsirkan (UF) di atas
argumen.

+-----------------------------------+--------------------------------------+
|Functions                          |Perilaku BMC/CHC                      |
+===================================+======================================+
|``assert``                         |Target verifikasi.                    |
+-----------------------------------+--------------------------------------+
|``require``                        |Asumsi.                               |
+-----------------------------------+--------------------------------------+
|internal call                      |BMC: Inline function call.            |
|                                   |CHC: Function summaries.              |
+-----------------------------------+--------------------------------------+
|external call to known code        |BMC: Inline function call atau        |
|                                   |menghapus knowledge tentang variabel  |
|                                   |state dan local storage references.   |
|                                   |CHC: Mwngasumsikan kode yang dipanggil|
|                                   |adalah unknown. Cobalah untuk         |
|                                   |menyimpulkan invarian yang bertahan   |
|                                   |setelah panggilan return.             |
+-----------------------------------+--------------------------------------+
|Storage array push/pop             |Didukung secara tepat.                |
|                                   |Memeriksa apakah itu memunculkan      |
|                                   |array kosong.                         |
+-----------------------------------+--------------------------------------+
|ABI functions                      |Diabstraksikan dengan UF.             |
+-----------------------------------+--------------------------------------+
|``addmod``, ``mulmod``             |Didukung secara tepat.                |
+-----------------------------------+--------------------------------------+
|``gasleft``, ``blockhash``,        |Diabstraksikan dengan UF.             |
|``keccak256``, ``ecrecover``       |                                      |
|``ripemd160``                      |                                      |
+-----------------------------------+--------------------------------------+
|pure functions without             |Diabstraksikan dengan UF.             |
|implementation (external or        |                                      |
|complex)                           |                                      |
+-----------------------------------+--------------------------------------+
|external functions without         |BMC: menghapus state knowledge dan    |
|implementation                     |anggap hasilnya nondeterminisc.       |
|                                   |CHC: Ringkasan Nondeterministic.      |
|                                   |Cobalah untuk menyimpulkan invarian   |
|                                   |yang bertahan setelah call returns.   |
+-----------------------------------+--------------------------------------+
|transfer                           |BMC: Memeriksa apakah                 |
|                                   |saldo kontrak mencukupi.              |
|                                   |CHC: belum melakukan pemeriksaan.     |
+-----------------------------------+--------------------------------------+
|others                             |Saat ini tidak didukung               |
+-----------------------------------+--------------------------------------+

Menggunakan abstraksi berarti kehilangan pengetahuan yang tepat, tetapi dalam banyak kasus
itu tidak berarti kehilangan kekuatan pembuktian.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Recover
    {
        function f(
            bytes32 hash,
            uint8 _v1, uint8 _v2,
            bytes32 _r1, bytes32 _r2,
            bytes32 _s1, bytes32 _s2
        ) public pure returns (address) {
            address a1 = ecrecover(hash, _v1, _r1, _s1);
            require(_v1 == _v2);
            require(_r1 == _r2);
            require(_s1 == _s2);
            address a2 = ecrecover(hash, _v2, _r2, _s2);
            assert(a1 == a2);
            return a1;
        }
    }

Dalam contoh di atas, SMTChecker tidak cukup ekspresif untuk benar-benar menghitung ``ecrecover``,
tetapi dengan memodelkan pemanggilan fungsi sebagai fungsi yang tidak diinterpretasikan, kita tahu
bahwa nilai yang dikembalikan adalah sama ketika dipanggil pada parameter yang setara. Ini cukup untuk
membuktikan bahwa pernyataan di atas selalu benar.

Mengabstraksi panggilan fungsi dengan UF dapat dilakukan untuk fungsi yang diketahui deterministik,
dan dapat dengan mudah dilakukan untuk fungsi pure. Namun sulit untuk melakukan ini dengan fungsi
eksternal umum, karena mereka mungkin bergantung pada variabel state.

Reference Types dan Aliasing
============================

Solidity mengimplementasikan aliasing untuk tipe referensi dengan
:ref:`lokasi data <data-location>` yang sama.
Itu berarti satu variabel dapat dimodifikasi melalui referensi ke area data
yang sama.
SMTChecker tidak melacak referensi mana yang merujuk ke data yang sama.
Ini menyiratkan bahwa setiap kali referensi lokal atau variabel state tipe referensi ditetapkan,
semua pengetahuan tentang variabel dengan tipe dan lokasi data yang sama dihapus.
Jika tipenya nested, penghapusan pengetahuan juga mencakup semua tipe dasar awalan.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Aliasing
    {
        uint[] array1;
        uint[][] array2;
        function f(
            uint[] memory a,
            uint[] memory b,
            uint[][] memory c,
            uint[] storage d
        ) internal {
            array1[0] = 42;
            a[0] = 2;
            c[0][0] = 2;
            b[0] = 1;
            // Erasing knowledge about memory references should not
            // erase knowledge about state variables.
            assert(array1[0] == 42);
            // However, an assignment to a storage reference will erase
            // storage knowledge accordingly.
            d[0] = 2;
            // Fails as false positive because of the assignment above.
            assert(array1[0] == 42);
            // Fails because `a == b` is possible.
            assert(a[0] == 2);
            // Fails because `c[i] == b` is possible.
            assert(c[0][0] == 2);
            assert(d[0] == 2);
            assert(b[0] == 1);
        }
        function g(
            uint[] memory a,
            uint[] memory b,
            uint[][] memory c,
            uint x
        ) public {
            f(a, b, c, array2[x]);
        }
    }

Setelah assignment ke ``b[0]``, kita perlu menghapus pengetahuan tentang ``a`` karena
memiliki tipe yang sama (``uint[]``) dan lokasi data (memori). Kita juga perlu
pengetahuan yang jelas tentang ``c``, karena tipe dasarnya juga terletak di ``uint[]``
dalam memori. Ini menyiratkan bahwa beberapa ``c[i]`` dapat merujuk ke data yang sama dengan
``b`` atau ``a``.

Perhatikan bahwa kita tidak menghapus pengetahuan tentang ``array`` dan ``d`` karena keduanya
terletak di penyimpanan, meskipun mereka juga memiliki tipe ``uint[]``. Namun,
jika ``d`` ditetapkan, kita perlu menghapus pengetahuan tentang ``array`` dan
sebaliknya.

Saldo Kontrak
=============

Kontrak dapat di-deploy dengan dana yang dikirim ke sana, jika ``msg.value`` > 0 saat
transaksi deployment.
Namun, alamat kontrak mungkin sudah memiliki dana sebelum deployment,
yang disimpan oleh kontrak.
Oleh karena itu, SMTChecker mengasumsikan bahwa ``address(this).balance >= msg.value``
di konstruktor agar konsisten dengan aturan EVM.
Saldo kontrak juga dapat meningkat tanpa memicu panggilan ke
kontrak, jika

- ``selfdestruct`` dieksekusi oleh kontrak lain dengan kontrak yang dianalisis
  sebagai target sisa dana,
- kontraknya adalah coinbase (yaitu, ``block.coinbase``) dari beberapa blok.

Untuk memodelkan ini dengan benar, SMTChecker mengasumsikan bahwa pada setiap transaksi baru
saldo kontrak dapat bertambah dengan setidaknya ``msg.value``.

**********************
Asumsi Dunia Nyata
**********************

Beberapa skenario dapat diekspresikan dalam Solidity dan EVM, tetapi diharapkan untuk
tidak pernah terjadi dalam praktik.
Salah satu kasus tersebut adalah panjang array penyimpanan dinamis yang meluap selama
push: Jika operasi ``push`` diterapkan ke array dengan panjang 2^256 - 1, panjangnya
akan overflow secara diam-diam.
Namun, ini tidak mungkin terjadi dalam praktiknya, karena operasi yang diperlukan untuk menumbuhkan
array ke titik itu akan membutuhkan waktu miliaran tahun untuk dieksekusi.
Asumsi serupa lainnya yang diambil oleh SMTChecker adalah bahwa saldo alamat
tidak pernah bisa overflow.

Ide serupa disampaikan di `EIP-1985 <https://eips.ethereum.org/EIPS/eip-1985>`_.
=======
.. _formal_verification:

##################################
SMTChecker and Formal Verification
##################################

Using formal verification it is possible to perform an automated mathematical
proof that your source code fulfills a certain formal specification.
The specification is still formal (just as the source code), but usually much
simpler.

Note that formal verification itself can only help you understand the
difference between what you did (the specification) and how you did it
(the actual implementation). You still need to check whether the specification
is what you wanted and that you did not miss any unintended effects of it.

Solidity implements a formal verification approach based on
`SMT (Satisfiability Modulo Theories) <https://en.wikipedia.org/wiki/Satisfiability_modulo_theories>`_ and
`Horn <https://en.wikipedia.org/wiki/Horn-satisfiability>`_ solving.
The SMTChecker module automatically tries to prove that the code satisfies the
specification given by ``require`` and ``assert`` statements. That is, it considers
``require`` statements as assumptions and tries to prove that the conditions
inside ``assert`` statements are always true.  If an assertion failure is
found, a counterexample may be given to the user showing how the assertion can
be violated. If no warning is given by the SMTChecker for a property,
it means that the property is safe.

The other verification targets that the SMTChecker checks at compile time are:

- Arithmetic underflow and overflow.
- Division by zero.
- Trivial conditions and unreachable code.
- Popping an empty array.
- Out of bounds index access.
- Insufficient funds for a transfer.

All the targets above are automatically checked by default if all engines are
enabled, except underflow and overflow for Solidity >=0.8.7.

The potential warnings that the SMTChecker reports are:

- ``<failing  property> happens here.``. This means that the SMTChecker proved that a certain property fails. A counterexample may be given, however in complex situations it may also not show a counterexample. This result may also be a false positive in certain cases, when the SMT encoding adds abstractions for Solidity code that is either hard or impossible to express.
- ``<failing property> might happen here``. This means that the solver could not prove either case within the given timeout. Since the result is unknown, the SMTChecker reports the potential failure for soundness. This may be solved by increasing the query timeout, but the problem might also simply be too hard for the engine to solve.

To enable the SMTChecker, you must select :ref:`which engine should run<smtchecker_engines>`,
where the default is no engine. Selecting the engine enables the SMTChecker on all files.

.. note::

    Prior to Solidity 0.8.4, the default way to enable the SMTChecker was via
    ``pragma experimental SMTChecker;`` and only the contracts containing the
    pragma would be analyzed. That pragma has been deprecated, and although it
    still enables the SMTChecker for backwards compatibility, it will be removed
    in Solidity 0.9.0. Note also that now using the pragma even in a single file
    enables the SMTChecker for all files.

.. note::

    The lack of warnings for a verification target represents an undisputed
    mathematical proof of correctness, assuming no bugs in the SMTChecker and
    the underlying solver. Keep in mind that these problems are
    *very hard* and sometimes *impossible* to solve automatically in the
    general case.  Therefore, several properties might not be solved or might
    lead to false positives for large contracts. Every proven property should
    be seen as an important achievement. For advanced users, see :ref:`SMTChecker Tuning <smtchecker_options>`
    to learn a few options that might help proving more complex
    properties.

********
Tutorial
********

Overflow
========

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Overflow {
        uint immutable x;
        uint immutable y;

        function add(uint x_, uint y_) internal pure returns (uint) {
            return x_ + y_;
        }

        constructor(uint x_, uint y_) {
            (x, y) = (x_, y_);
        }

        function stateAdd() public view returns (uint) {
            return add(x, y);
        }
    }

The contract above shows an overflow check example.
The SMTChecker does not check underflow and overflow by default for Solidity >=0.8.7,
so we need to use the command line option ``--model-checker-targets "underflow,overflow"``
or the JSON option ``settings.modelChecker.targets = ["underflow", "overflow"]``.
See :ref:`this section for targets configuration<smtchecker_targets>`.
Here, it reports the following:

.. code-block:: text

    Warning: CHC: Overflow (resulting value larger than 2**256 - 1) happens here.
    Counterexample:
    x = 1, y = 115792089237316195423570985008687907853269984665640564039457584007913129639935
     = 0

    Transaction trace:
    Overflow.constructor(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935)
    State: x = 1, y = 115792089237316195423570985008687907853269984665640564039457584007913129639935
    Overflow.stateAdd()
        Overflow.add(1, 115792089237316195423570985008687907853269984665640564039457584007913129639935) -- internal call
     --> o.sol:9:20:
      |
    9 |             return x_ + y_;
      |                    ^^^^^^^

If we add ``require`` statements that filter out overflow cases,
the SMTChecker proves that no overflow is reachable (by not reporting warnings):

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Overflow {
        uint immutable x;
        uint immutable y;

        function add(uint x_, uint y_) internal pure returns (uint) {
            return x_ + y_;
        }

        constructor(uint x_, uint y_) {
            (x, y) = (x_, y_);
        }

        function stateAdd() public view returns (uint) {
            require(x < type(uint128).max);
            require(y < type(uint128).max);
            return add(x, y);
        }
    }


Assert
======

An assertion represents an invariant in your code: a property that must be true
*for all transactions, including all input and storage values*, otherwise there is a bug.

The code below defines a function ``f`` that guarantees no overflow.
Function ``inv`` defines the specification that ``f`` is monotonically increasing:
for every possible pair ``(a, b)``, if ``b > a`` then ``f(b) > f(a)``.
Since ``f`` is indeed monotonically increasing, the SMTChecker proves that our
property is correct. You are encouraged to play with the property and the function
definition to see what results come out!

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Monotonic {
        function f(uint x) internal pure returns (uint) {
            require(x < type(uint128).max);
            return x * 42;
        }

        function inv(uint a, uint b) public pure {
            require(b > a);
            assert(f(b) > f(a));
        }
    }

We can also add assertions inside loops to verify more complicated properties.
The following code searches for the maximum element of an unrestricted array of
numbers, and asserts the property that the found element must be greater or
equal every element in the array.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Max {
        function max(uint[] memory a) public pure returns (uint) {
            uint m = 0;
            for (uint i = 0; i < a.length; ++i)
                if (a[i] > m)
                    m = a[i];

            for (uint i = 0; i < a.length; ++i)
                assert(m >= a[i]);

            return m;
        }
    }

Note that in this example the SMTChecker will automatically try to prove three properties:

1. ``++i`` in the first loop does not overflow.
2. ``++i`` in the second loop does not overflow.
3. The assertion is always true.

.. note::

    The properties involve loops, which makes it *much much* harder than the previous
    examples, so beware of loops!

All the properties are correctly proven safe. Feel free to change the
properties and/or add restrictions on the array to see different results.
For example, changing the code to

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Max {
        function max(uint[] memory a) public pure returns (uint) {
            require(a.length >= 5);
            uint m = 0;
            for (uint i = 0; i < a.length; ++i)
                if (a[i] > m)
                    m = a[i];

            for (uint i = 0; i < a.length; ++i)
                assert(m > a[i]);

            return m;
        }
    }

gives us:

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:

    a = [0, 0, 0, 0, 0]
     = 0

    Transaction trace:
    Test.constructor()
    Test.max([0, 0, 0, 0, 0])
      --> max.sol:14:4:
       |
    14 |            assert(m > a[i]);


State Properties
================

So far the examples only demonstrated the use of the SMTChecker over pure code,
proving properties about specific operations or algorithms.
A common type of properties in smart contracts are properties that involve the
state of the contract. Multiple transactions might be needed to make an assertion
fail for such a property.

As an example, consider a 2D grid where both axis have coordinates in the range (-2^128, 2^128 - 1).
Let us place a robot at position (0, 0). The robot can only move diagonally, one step at a time,
and cannot move outside the grid. The robot's state machine can be represented by the smart contract
below.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Robot {
        int x = 0;
        int y = 0;

        modifier wall {
            require(x > type(int128).min && x < type(int128).max);
            require(y > type(int128).min && y < type(int128).max);
            _;
        }

        function moveLeftUp() wall public {
            --x;
            ++y;
        }

        function moveLeftDown() wall public {
            --x;
            --y;
        }

        function moveRightUp() wall public {
            ++x;
            ++y;
        }

        function moveRightDown() wall public {
            ++x;
            --y;
        }

        function inv() public view {
            assert((x + y) % 2 == 0);
        }
    }

Function ``inv`` represents an invariant of the state machine that ``x + y``
must be even.
The SMTChecker manages to prove that regardless how many commands we give the
robot, even if infinitely many, the invariant can *never* fail. The interested
reader may want to prove that fact manually as well.  Hint: this invariant is
inductive.

We can also trick the SMTChecker into giving us a path to a certain position we
think might be reachable.  We can add the property that (2, 4) is *not*
reachable, by adding the following function.

.. code-block:: Solidity

    function reach_2_4() public view {
        assert(!(x == 2 && y == 4));
    }

This property is false, and while proving that the property is false,
the SMTChecker tells us exactly *how* to reach (2, 4):

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:
    x = 2, y = 4

    Transaction trace:
    Robot.constructor()
    State: x = 0, y = 0
    Robot.moveLeftUp()
    State: x = (- 1), y = 1
    Robot.moveRightUp()
    State: x = 0, y = 2
    Robot.moveRightUp()
    State: x = 1, y = 3
    Robot.moveRightUp()
    State: x = 2, y = 4
    Robot.reach_2_4()
      --> r.sol:35:4:
       |
    35 |            assert(!(x == 2 && y == 4));
       |            ^^^^^^^^^^^^^^^^^^^^^^^^^^^

Note that the path above is not necessarily deterministic, as there are
other paths that could reach (2, 4). The choice of which path is shown
might change depending on the used solver, its version, or just randomly.

External Calls and Reentrancy
=============================

Every external call is treated as a call to unknown code by the SMTChecker.
The reasoning behind that is that even if the code of the called contract is
available at compile time, there is no guarantee that the deployed contract
will indeed be the same as the contract where the interface came from at
compile time.

In some cases, it is possible to automatically infer properties over state
variables that are still true even if the externally called code can do
anything, including reenter the caller contract.

.. code-block:: Solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    interface Unknown {
        function run() external;
    }

    contract Mutex {
        uint x;
        bool lock;

        Unknown immutable unknown;

        constructor(Unknown u) {
            require(address(u) != address(0));
            unknown = u;
        }

        modifier mutex {
            require(!lock);
            lock = true;
            _;
            lock = false;
        }

        function set(uint x_) mutex public {
            x = x_;
        }

        function run() mutex public {
            uint xPre = x;
            unknown.run();
            assert(xPre == x);
        }
    }

The example above shows a contract that uses a mutex flag to forbid reentrancy.
The solver is able to infer that when ``unknown.run()`` is called, the contract
is already "locked", so it would not be possible to change the value of ``x``,
regardless of what the unknown called code does.

If we "forget" to use the ``mutex`` modifier on function ``set``, the
SMTChecker is able to synthesize the behaviour of the externally called code so
that the assertion fails:

.. code-block:: text

    Warning: CHC: Assertion violation happens here.
    Counterexample:
    x = 1, lock = true, unknown = 1

    Transaction trace:
    Mutex.constructor(1)
    State: x = 0, lock = false, unknown = 1
    Mutex.run()
        unknown.run() -- untrusted external call, synthesized as:
            Mutex.set(1) -- reentrant call
      --> m.sol:32:3:
       |
    32 | 		assert(xPre == x);
       | 		^^^^^^^^^^^^^^^^^


.. _smtchecker_options:

*****************************
SMTChecker Options and Tuning
*****************************

Timeout
=======

The SMTChecker uses a hardcoded resource limit (``rlimit``) chosen per solver,
which is not precisely related to time. We chose the ``rlimit`` option as the default
because it gives more determinism guarantees than time inside the solver.

This options translates roughly to "a few seconds timeout" per query. Of course many properties
are very complex and need a lot of time to be solved, where determinism does not matter.
If the SMTChecker does not manage to solve the contract properties with the default ``rlimit``,
a timeout can be given in milliseconds via the CLI option ``--model-checker-timeout <time>`` or
the JSON option ``settings.modelChecker.timeout=<time>``, where 0 means no timeout.

.. _smtchecker_targets:

Verification Targets
====================

The types of verification targets created by the SMTChecker can also be
customized via the CLI option ``--model-checker-target <targets>`` or the JSON
option ``settings.modelChecker.targets=<targets>``.
In the CLI case, ``<targets>`` is a no-space-comma-separated list of one or
more verification targets, and an array of one or more targets as strings in
the JSON input.
The keywords that represent the targets are:

- Assertions: ``assert``.
- Arithmetic underflow: ``underflow``.
- Arithmetic overflow: ``overflow``.
- Division by zero: ``divByZero``.
- Trivial conditions and unreachable code: ``constantCondition``.
- Popping an empty array: ``popEmptyArray``.
- Out of bounds array/fixed bytes index access: ``outOfBounds``.
- Insufficient funds for a transfer: ``balance``.
- All of the above: ``default`` (CLI only).

A common subset of targets might be, for example:
``--model-checker-targets assert,overflow``.

All targets are checked by default, except underflow and overflow for Solidity >=0.8.7.

There is no precise heuristic on how and when to split verification targets,
but it can be useful especially when dealing with large contracts.

Unproved Targets
================

If there are any unproved targets, the SMTChecker issues one warning stating
how many unproved targets there are. If the user wishes to see all the specific
unproved targets, the CLI option ``--model-checker-show-unproved`` and
the JSON option ``settings.modelChecker.showUnproved = true`` can be used.

Verified Contracts
==================

By default all the deployable contracts in the given sources are analyzed separately as
the one that will be deployed. This means that if a contract has many direct
and indirect inheritance parents, all of them will be analyzed on their own,
even though only the most derived will be accessed directly on the blockchain.
This causes an unnecessary burden on the SMTChecker and the solver.  To aid
cases like this, users can specify which contracts should be analyzed as the
deployed one. The parent contracts are of course still analyzed, but only in
the context of the most derived contract, reducing the complexity of the
encoding and generated queries. Note that abstract contracts are by default
not analyzed as the most derived by the SMTChecker.

The chosen contracts can be given via a comma-separated list (whitespace is not
allowed) of <source>:<contract> pairs in the CLI:
``--model-checker-contracts "<source1.sol:contract1>,<source2.sol:contract2>,<source2.sol:contract3>"``,
and via the object ``settings.modelChecker.contracts`` in the :ref:`JSON input<compiler-api>`,
which has the following form:

.. code-block:: json

    "contracts": {
        "source1.sol": ["contract1"],
        "source2.sol": ["contract2", "contract3"]
    }

Reported Inferred Inductive Invariants
======================================

For properties that were proved safe with the CHC engine,
the SMTChecker can retrieve inductive invariants that were inferred by the Horn
solver as part of the proof.
Currently two types of invariants can be reported to the user:

- Contract Invariants: these are properties over the contract's state variables
  that are true before and after every possible transaction that the contract may ever run. For example, ``x >= y``, where ``x`` and ``y`` are a contract's state variables.
- Reentrancy Properties: they represent the behavior of the contract
  in the presence of external calls to unknown code. These properties can express a relation
  between the value of the state variables before and after the external call, where the external call is free to do anything, including making reentrant calls to the analyzed contract. Primed variables represent the state variables' values after said external call. Example: ``lock -> x = x'``.

The user can choose the type of invariants to be reported using the CLI option ``--model-checker-invariants "contract,reentrancy"`` or as an array in the field ``settings.modelChecker.invariants`` in the :ref:`JSON input<compiler-api>`.
By default the SMTChecker does not report invariants.

Division and Modulo With Slack Variables
========================================

Spacer, the default Horn solver used by the SMTChecker, often dislikes division
and modulo operations inside Horn rules. Because of that, by default the
Solidity division and modulo operations are encoded using the constraint
``a = b * d + m`` where ``d = a / b`` and ``m = a % b``.
However, other solvers, such as Eldarica, prefer the syntactically precise operations.
The command line flag ``--model-checker-div-mod-no-slacks`` and the JSON option
``settings.modelChecker.divModNoSlacks`` can be used to toggle the encoding
depending on the used solver preferences.

Natspec Function Abstraction
============================

Certain functions including common math methods such as ``pow``
and ``sqrt`` may be too complex to be analyzed in a fully automated way.
These functions can be annotated with Natspec tags that indicate to the
SMTChecker that these functions should be abstracted. This means that the
body of the function is not used, and when called, the function will:

- Return a nondeterministic value, and either keep the state variables unchanged if the abstracted function is view/pure, or also set the state variables to nondeterministic values otherwise. This can be used via the annotation ``/// @custom:smtchecker abstract-function-nondet``.
- Act as an uninterpreted function. This means that the semantics of the function (given by the body) are ignored, and the only property this function has is that given the same input it guarantees the same output. This is currently under development and will be available via the annotation ``/// @custom:smtchecker abstract-function-uf``.

.. _smtchecker_engines:

Model Checking Engines
======================

The SMTChecker module implements two different reasoning engines, a Bounded
Model Checker (BMC) and a system of Constrained Horn Clauses (CHC).  Both
engines are currently under development, and have different characteristics.
The engines are independent and every property warning states from which engine
it came. Note that all the examples above with counterexamples were
reported by CHC, the more powerful engine.

By default both engines are used, where CHC runs first, and every property that
was not proven is passed over to BMC. You can choose a specific engine via the CLI
option ``--model-checker-engine {all,bmc,chc,none}`` or the JSON option
``settings.modelChecker.engine={all,bmc,chc,none}``.

Bounded Model Checker (BMC)
---------------------------

The BMC engine analyzes functions in isolation, that is, it does not take the
overall behavior of the contract over multiple transactions into account when
analyzing each function.  Loops are also ignored in this engine at the moment.
Internal function calls are inlined as long as they are not recursive, directly
or indirectly. External function calls are inlined if possible. Knowledge
that is potentially affected by reentrancy is erased.

The characteristics above make BMC prone to reporting false positives,
but it is also lightweight and should be able to quickly find small local bugs.

Constrained Horn Clauses (CHC)
------------------------------

A contract's Control Flow Graph (CFG) is modelled as a system of
Horn clauses, where the life cycle of the contract is represented by a loop
that can visit every public/external function non-deterministically. This way,
the behavior of the entire contract over an unbounded number of transactions
is taken into account when analyzing any function. Loops are fully supported
by this engine. Internal function calls are supported, and external function
calls assume the called code is unknown and can do anything.

The CHC engine is much more powerful than BMC in terms of what it can prove,
and might require more computing resources.

SMT and Horn solvers
====================

The two engines detailed above use automated theorem provers as their logical
backends.  BMC uses an SMT solver, whereas CHC uses a Horn solver. Often the
same tool can act as both, as seen in `z3 <https://github.com/Z3Prover/z3>`_,
which is primarily an SMT solver and makes `Spacer
<https://spacer.bitbucket.io/>`_ available as a Horn solver, and `Eldarica
<https://github.com/uuverifiers/eldarica>`_ which does both.

The user can choose which solvers should be used, if available, via the CLI
option ``--model-checker-solvers {all,cvc4,smtlib2,z3}`` or the JSON option
``settings.modelChecker.solvers=[smtlib2,z3]``, where:

- ``cvc4`` is only available if the ``solc`` binary is compiled with it. Only BMC uses ``cvc4``.
- ``smtlib2`` outputs SMT/Horn queries in the `smtlib2 <http://smtlib.cs.uiowa.edu/>`_ format.
  These can be used together with the compiler's `callback mechanism <https://github.com/ethereum/solc-js>`_ so that
  any solver binary from the system can be employed to synchronously return the results of the queries to the compiler.
  This is currently the only way to use Eldarica, for example, since it does not have a C++ API.
  This can be used by both BMC and CHC depending on which solvers are called.
- ``z3`` is available

  - if ``solc`` is compiled with it;
  - if a dynamic ``z3`` library of version 4.8.x is installed in a Linux system (from Solidity 0.7.6);
  - statically in ``soljson.js`` (from Solidity 0.6.9), that is, the Javascript binary of the compiler.

Since both BMC and CHC use ``z3``, and ``z3`` is available in a greater variety
of environments, including in the browser, most users will almost never need to be
concerned about this option. More advanced users might apply this option to try
alternative solvers on more complex problems.

Please note that certain combinations of chosen engine and solver will lead to
the SMTChecker doing nothing, for example choosing CHC and ``cvc4``.

*******************************
Abstraction and False Positives
*******************************

The SMTChecker implements abstractions in an incomplete and sound way: If a bug
is reported, it might be a false positive introduced by abstractions (due to
erasing knowledge or using a non-precise type). If it determines that a
verification target is safe, it is indeed safe, that is, there are no false
negatives (unless there is a bug in the SMTChecker).

If a target cannot be proven you can try to help the solver by using the tuning
options in the previous section.
If you are sure of a false positive, adding ``require`` statements in the code
with more information may also give some more power to the solver.

SMT Encoding and Types
======================

The SMTChecker encoding tries to be as precise as possible, mapping Solidity types
and expressions to their closest `SMT-LIB <http://smtlib.cs.uiowa.edu/>`_
representation, as shown in the table below.

+-----------------------+--------------------------------+-----------------------------+
|Solidity type          |SMT sort                        |Theories                     |
+=======================+================================+=============================+
|Boolean                |Bool                            |Bool                         |
+-----------------------+--------------------------------+-----------------------------+
|intN, uintN, address,  |Integer                         |LIA, NIA                     |
|bytesN, enum, contract |                                |                             |
+-----------------------+--------------------------------+-----------------------------+
|array, mapping, bytes, |Tuple                           |Datatypes, Arrays, LIA       |
|string                 |(Array elements, Integer length)|                             |
+-----------------------+--------------------------------+-----------------------------+
|struct                 |Tuple                           |Datatypes                    |
+-----------------------+--------------------------------+-----------------------------+
|other types            |Integer                         |LIA                          |
+-----------------------+--------------------------------+-----------------------------+

Types that are not yet supported are abstracted by a single 256-bit unsigned
integer, where their unsupported operations are ignored.

For more details on how the SMT encoding works internally, see the paper
`SMT-based Verification of Solidity Smart Contracts <https://github.com/leonardoalt/text/blob/master/solidity_isola_2018/main.pdf>`_.

Function Calls
==============

In the BMC engine, function calls to the same contract (or base contracts) are
inlined when possible, that is, when their implementation is available.  Calls
to functions in other contracts are not inlined even if their code is
available, since we cannot guarantee that the actual deployed code is the same.

The CHC engine creates nonlinear Horn clauses that use summaries of the called
functions to support internal function calls. External function calls are treated
as calls to unknown code, including potential reentrant calls.

Complex pure functions are abstracted by an uninterpreted function (UF) over
the arguments.

+-----------------------------------+--------------------------------------+
|Functions                          |BMC/CHC behavior                      |
+===================================+======================================+
|``assert``                         |Verification target.                  |
+-----------------------------------+--------------------------------------+
|``require``                        |Assumption.                           |
+-----------------------------------+--------------------------------------+
|internal call                      |BMC: Inline function call.            |
|                                   |CHC: Function summaries.              |
+-----------------------------------+--------------------------------------+
|external call to known code        |BMC: Inline function call or          |
|                                   |erase knowledge about state variables |
|                                   |and local storage references.         |
|                                   |CHC: Assume called code is unknown.   |
|                                   |Try to infer invariants that hold     |
|                                   |after the call returns.               |
+-----------------------------------+--------------------------------------+
|Storage array push/pop             |Supported precisely.                  |
|                                   |Checks whether it is popping an       |
|                                   |empty array.                          |
+-----------------------------------+--------------------------------------+
|ABI functions                      |Abstracted with UF.                   |
+-----------------------------------+--------------------------------------+
|``addmod``, ``mulmod``             |Supported precisely.                  |
+-----------------------------------+--------------------------------------+
|``gasleft``, ``blockhash``,        |Abstracted with UF.                   |
|``keccak256``, ``ecrecover``       |                                      |
|``ripemd160``                      |                                      |
+-----------------------------------+--------------------------------------+
|pure functions without             |Abstracted with UF                    |
|implementation (external or        |                                      |
|complex)                           |                                      |
+-----------------------------------+--------------------------------------+
|external functions without         |BMC: Erase state knowledge and assume |
|implementation                     |result is nondeterminisc.             |
|                                   |CHC: Nondeterministic summary.        |
|                                   |Try to infer invariants that hold     |
|                                   |after the call returns.               |
+-----------------------------------+--------------------------------------+
|transfer                           |BMC: Checks whether the contract's    |
|                                   |balance is sufficient.                |
|                                   |CHC: does not yet perform the check.  |
+-----------------------------------+--------------------------------------+
|others                             |Currently unsupported                 |
+-----------------------------------+--------------------------------------+

Using abstraction means loss of precise knowledge, but in many cases it does
not mean loss of proving power.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Recover
    {
        function f(
            bytes32 hash,
            uint8 v1, uint8 v2,
            bytes32 r1, bytes32 r2,
            bytes32 s1, bytes32 s2
        ) public pure returns (address) {
            address a1 = ecrecover(hash, v1, r1, s1);
            require(v1 == v2);
            require(r1 == r2);
            require(s1 == s2);
            address a2 = ecrecover(hash, v2, r2, s2);
            assert(a1 == a2);
            return a1;
        }
    }

In the example above, the SMTChecker is not expressive enough to actually
compute ``ecrecover``, but by modelling the function calls as uninterpreted
functions we know that the return value is the same when called on equivalent
parameters. This is enough to prove that the assertion above is always true.

Abstracting a function call with an UF can be done for functions known to be
deterministic, and can be easily done for pure functions.  It is however
difficult to do this with general external functions, since they might depend
on state variables.

Reference Types and Aliasing
============================

Solidity implements aliasing for reference types with the same :ref:`data
location<data-location>`.
That means one variable may be modified through a reference to the same data
area.
The SMTChecker does not keep track of which references refer to the same data.
This implies that whenever a local reference or state variable of reference
type is assigned, all knowledge regarding variables of the same type and data
location is erased.
If the type is nested, the knowledge removal also includes all the prefix base
types.

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.8.0;

    contract Aliasing
    {
        uint[] array1;
        uint[][] array2;
        function f(
            uint[] memory a,
            uint[] memory b,
            uint[][] memory c,
            uint[] storage d
        ) internal {
            array1[0] = 42;
            a[0] = 2;
            c[0][0] = 2;
            b[0] = 1;
            // Erasing knowledge about memory references should not
            // erase knowledge about state variables.
            assert(array1[0] == 42);
            // However, an assignment to a storage reference will erase
            // storage knowledge accordingly.
            d[0] = 2;
            // Fails as false positive because of the assignment above.
            assert(array1[0] == 42);
            // Fails because `a == b` is possible.
            assert(a[0] == 2);
            // Fails because `c[i] == b` is possible.
            assert(c[0][0] == 2);
            assert(d[0] == 2);
            assert(b[0] == 1);
        }
        function g(
            uint[] memory a,
            uint[] memory b,
            uint[][] memory c,
            uint x
        ) public {
            f(a, b, c, array2[x]);
        }
    }

After the assignment to ``b[0]``, we need to clear knowledge about ``a`` since
it has the same type (``uint[]``) and data location (memory).  We also need to
clear knowledge about ``c``, since its base type is also a ``uint[]`` located
in memory. This implies that some ``c[i]`` could refer to the same data as
``b`` or ``a``.

Notice that we do not clear knowledge about ``array`` and ``d`` because they
are located in storage, even though they also have type ``uint[]``.  However,
if ``d`` was assigned, we would need to clear knowledge about ``array`` and
vice-versa.

Contract Balance
================

A contract may be deployed with funds sent to it, if ``msg.value`` > 0 in the
deployment transaction.
However, the contract's address may already have funds before deployment,
which are kept by the contract.
Therefore, the SMTChecker assumes that ``address(this).balance >= msg.value``
in the constructor in order to be consistent with the EVM rules.
The contract's balance may also increase without triggering any calls to the
contract, if

- ``selfdestruct`` is executed by another contract with the analyzed contract
  as the target of the remaining funds,
- the contract is the coinbase (i.e., ``block.coinbase``) of some block.

To model this properly, the SMTChecker assumes that at every new transaction
the contract's balance may grow by at least ``msg.value``.

**********************
Real World Assumptions
**********************

Some scenarios can be expressed in Solidity and the EVM, but are expected to
never occur in practice.
One of such cases is the length of a dynamic storage array overflowing during a
push: If the ``push`` operation is applied to an array of length 2^256 - 1, its
length silently overflows.
However, this is unlikely to happen in practice, since the operations required
to grow the array to that point would take billions of years to execute.
Another similar assumption taken by the SMTChecker is that an address' balance
can never overflow.

A similar idea was presented in `EIP-1985 <https://eips.ethereum.org/EIPS/eip-1985>`_.
>>>>>>> 43f29c00da02e19ff10d43f7eb6955d627c57728
