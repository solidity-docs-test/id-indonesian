<<<<<<< HEAD
.. index:: ! value type, ! type;value
.. _value-types:

Value Types (Nilai Types)
=========================

Tipe berikut ini juga disebut tipe nilai karena variabel dari tipe ini akan
selalu diteruskan dengan nilai, yaitu selalu disalin ketika digunakan sebagai
argumen fungsi atau dalam penugasan.

.. index:: ! bool, ! true, ! false

Booleans
--------

``bool``: Nilai yang mungkin adalah konstantan ``true`` dan ``false``.

Operators:

*  ``!`` (logical negation)
*  ``&&`` (logical conjunction, "and")
*  ``||`` (logical disjunction, "or")
*  ``==`` (equality)
*  ``!=`` (inequality)

Operator ``||`` dan ``&&`` menerapkan aturan *short-circuiting* yang umum. Ini berarti bahwa dalam ekspresi ``f(x) || g(y)``, jika ``f(x)`` bernilai ``true``, ``g(y)`` tidak akan dievaluasi meskipun mungkin memiliki efek samping.

.. index:: ! uint, ! int, ! integer
.. _integers:

Integers
--------

``int`` / ``uint``: Signed dan unsigned integers dalam berbagai ukuran. Keywords ``uint8`` ke ``uint256`` dalam langkah ``8`` (unsigned dari 8 sampai 256 bits) dan ``int8`` ke ``int256``. ``uint`` serta ``int`` adalah alias untuk ``uint256`` dan ``int256``, berturut-turut.

Operators:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evaluasi ke ``bool``)
* Bit operators: ``&``, ``|``, ``^`` (bitwise eksklusif atau), ``~`` (bitwise negation)
* Shift operators: ``<<`` (shift kiri), ``>>`` (shift kanan)
* Arithmetic operators: ``+``, ``-``, unary ``-`` (hanya untuk signed integers), ``*``, ``/``, ``%`` (modulo), ``**`` (exponentiation)

Untuk integer tipe ``X``, anda dapat menggunakan ``type(X).min`` dan ``type(X).max`` untuk
mengakses nilai minimum dan maksimum yang dapat diwakili oleh tipenya.

.. warning::

  Integers di Solidity terbatas pada kisaran tertentu. Sebagai contoh, dengan ``uint32``, ini adalah ``0`` hingga ``2**32 - 1``.
  Ada dua mode di mana aritmatika dilakukan pada tipe-tipe ini: Mode "wrapping" atau "unchecked" dan mode "checked".
  Secara default, aritmatika selalu "checked", yang berarti bahwa jika hasil operasi berada di luar rentang nilai
  dari jenisnya, panggilan dikembalikan melalui :ref:`pernyataan gagal<assert-and-require>`. Anda dapat beralih ke mode "unchecked"
  menggunakan ``unchecked { ... }``. Detail lebih lanjut dapat ditemukan di bagian :ref:`unchecked <unchecked>`.

Comparisons (Perbandingan)
^^^^^^^^^^^^^^^^^^^^^^^^^^

Nilai sebuah comparison adalah yang diperoleh dengan membandingkan nilai integer.

Bit operations
^^^^^^^^^^^^^^

Bit operations dilakukan pada representasi komplemen dua dari nomor tersebut.
Ini berarti, sebagai contoh ``~int256(0) == int256(-1)``.

Shifts
^^^^^^

Hasil dari operasi shift memiliki tipe operand kiri, memotong hasil agar sesuai dengan tipenya.
Operand kanan harus dari tipe yang tidak ditandatangani, mencoba untuk *shift* dengan tipe yang ditandatangani akan menghasilkan kesalahan kompilasi.

Shifts dapat di "simulasi" menggunakan perkalian dengan kekuatan dua dengan cara berikut. Perhatikan bahwa pemotongan
untuk jenis operand kiri selalu dilakukan di akhir, tetapi tidak disebutkan secara eksplisit.

- ``x << y`` setara dengan ekspresi matematika ``x * 2**y``.
- ``x >> y`` setara dengan ekspresi matematika ``x / 2**y``, dibulatkan ke arah negatif infinity.

.. warning::
    Sebelum versi ``0.5.0`` shift kanan ``x >> y`` untuk negatif ``x`` setara dengan
    ekspresi matematika ``x / 2**y`` dibulatkan menuju nol,
    yaitu, Shift kanan menggunakan pembulatan ke atas (menuju nol) daripada pembulatan ke bawah (menuju tak terhingga negatif).

.. note::
    Pemeriksaan overflow tidak pernah dilakukan untuk operasi shift seperti yang dilakukan untuk operasi aritmatika.
    Sebaliknya, hasilnya selalu terpotong.

Addition, Subtraction and Multiplication
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Addition, subtraction and multiplication memiliki semantics yang biasa, dengan dua mode
berbeda dalam hal over- dan underflow:

Secara default, semua aritmatika diperiksa untuk under- atau overflow, tetapi ini dapat dinonaktifkan
menggunakan :ref:`unchecked block<unchecked>`, menghasilkan *wrapping arithmetic*. Keterangan lebih lanjut
dapat ditemukan di bagian tersebut.

Ekspresi ``-x`` setara dengan ``(T(0) - x)`` dimana
``T`` adalah type dari ``x``. Ini hanya dapat diterapkan pada signed types.
Nilai ``-x`` dapat berupa
positif jika ``x`` adalah negatif. Ada peringatan lain yang juga dihasilkan
dari representasi dua komplemen:

Jika anda mempunyai ``int x = type(int).min;``, maka ``-x`` tidak sesuai dengan rentang positif.
Ini berarti bahwa ``unchecked { assert(-x == x); }`` bekerja, dan ekspresi ``-x``
ketika digunakan dalam mode checked akan menghasilkan pernyataan yang gagal

Division
^^^^^^^^

Karena tipe hasil operasi selalu tipe salah satu operan,
division pada integer selalu menghasilkan integer.
Di Solidity, putaran division menuju nol. Ini berarti bahwa ``int256(-5) / int256(2) == int256(-2)``.

Perhatikan bahwa sebaliknya, division pada :ref:`literals<rational_literals>` menghasilkan nilai
pecahan presisi arbitrary.

.. note::
  Division by zero mengakibatkan sebuah :ref:`Panic error<assert-and-require>`. Pemeriksaan ini **tidak** dapat dinonaktifkan melalui ``unchecked { ... }``.

.. note::
  Expression ``type(int).min / (-1)`` adalah satu-satunya kasus di mana division menyebabkan overflow.
  Di mode checked arithmetic, ini akan menyebabkan pernyataan yang gagal, sementara di
  mode wrapping, nilainya akan ``type(int).min``.

Modulo
^^^^^^

Operasi modulo ``a % n`` menghasilkan sisa ``r`` setelah pembagian operand ``a``
oleh operand ``n``, dimana ``q = int(a / n)`` dan ``r = a - (n * q)``. Ini berarti bahwa modulo
menghasilkan tanda yang sama dengan operand kiri (atau nol) dan ``a % n == -(-a % n)`` berlaku untuk ``a`` negatif:

* ``int256(5) % int256(2) == int256(1)``
* ``int256(5) % int256(-2) == int256(1)``
* ``int256(-5) % int256(2) == int256(-1)``
* ``int256(-5) % int256(-2) == int256(-1)``

.. note::
  Modulo dengan nol menyebabkan :ref:`Panic error<assert-and-require>`. Pemeriksaan ini **tidak** dapat dinonaktifkan melalui ``unchecked { ... }``.

Exponentiation
^^^^^^^^^^^^^^

Exponentiation hanya tersedia untuk tipe yang tidak ditandatangani dalam eksponen.
Jenis exponentiation yang dihasilkan selalu sama dengan tipe dasarnya.
Harap berhati-hati bahwa itu cukup besar untuk menampung hasil dan bersiap untuk
kemungkinan kegagalan pernyataan atau *wrapping behaviour*.

.. note::
  Di mode checked, exponentiation hanya menggunakan opcode ``exp`` yang relatif murah untuk basis kecil.
  Untuk kasus ``x**3``, expression ``x*x*x`` mungkin lebih murah.
  Bagaimanapun, tes biaya gas dan penggunaan pengoptimal disarankan.

.. note::
  Perhatikan bahwa ``0**0`` didefinisikan oleh EVM sebagai ``1``.

.. index:: ! ufixed, ! fixed, ! fixed point number

Fixed Point Numbers
-------------------

.. warning::
    Fixed point numbers belum sepenuhnya didukung oleh Solidity. Mereka dapat dideklarasikan,
    tetapi tidak dapat ditugaskan ke atau dari.

``fixed`` / ``ufixed``: Signed dan unsigned fixed point number dari berbagai ukuran. Keywords ``ufixedMxN`` dan ``fixedMxN``, dimana ``M`` mewakili jumlah bit yang diambil oleh
type dan ``N`` mewakili berapa banyak titik desimal yang tersedia. ``M`` harus habis dibagi 8 dan berubah dari 8 hingga 256 bit. ``N`` harus antara 0 dan 80, inklusif.
``ufixed`` dan ``fixed`` adalah alias untuk ``ufixed128x18`` dan ``fixed128x18``, secara respectif.

Operators:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evaluate to ``bool``)
* Arithmetic operators: ``+``, ``-``, unary ``-``, ``*``, ``/``, ``%`` (modulo)

.. note::
    Perbedaan utama antara floating point (``float`` dan ``double`` dibanyak bahasa, lebih tepatnya nomor IEEE 754) dan nomor fixed point adalah
    bahwa jumlah bit yang digunakan untuk integer dan bagian pecahan (bagian setelah titik desimal) fleksibel di bagian pertama, sementara itu
    didefinisikan secara ketat di bagian terakhir. Umumnya, di floating point hampir seluruh ruang digunakan untuk mewakili nomor, sementara hanya sejumlah kecil bit yang mendefinisikan
    dimana titik desimal berada.

.. index:: address, balance, send, call, delegatecall, staticcall, transfer

.. _address:

Address (alamat)
----------------

Tipe Address terdiri dari dua jenis, yang sebagian besar identik:

- ``address``: Memegang nilai 20 byte (ukuran alamat Ethereum).
- ``address payable``: Sama seperti ``address``, tetapi dengan anggota tambahan ``transfer`` dan ``send``.

Gagasan di balik perbedaan ini adalah bahwa ``address payable`` adalah alamat yang dapat Anda kirimi Ether,
sementara ``address`` biasa tidak dapat menerima Ether.

Tipe konversi:

Konversi implisit dari ``address payable`` ke ``address`` diperbolehkan, sedangkan konversi dari ``address`` ke ``address payable``
harus eksplisit melalui ``payable(<address>)``.

Konversi eksplisit ke dan dari ``address`` diperbolehkan untuk ``uint160``, literal integer,
``bytes20`` dan tipe kontrak.

Hanya tipe expressions ``address`` and contract-type yang dapat dikonversi ke tipe ``address
payable`` via konversi eksplisit ``payable(...)``. Untuk contract-type, konversi ini hanya
diperbolehkan jika kontrak dapat menerima Ether, yaitu kontrak yang memiliki :ref:`receive
<receive-ether-function>` atau fungsi fallback yang *payable*. Perhatikan bahwa ``payable(0)`` valid dan
pengecualian untuk aturan ini.

.. note::
    Jika Anda membutuhkan variabel bertipe ``address`` dan berencana mengirim Ether ke sana, maka
    mendeklarasikan jenisnya sebagai ``address payable`` untuk membuat persyaratan ini terlihat. Juga,
    cobalah untuk membuat perbedaan atau konversi ini sedini mungkin.

Operators:

* ``<=``, ``<``, ``==``, ``!=``, ``>=`` dan ``>``

.. warning::
    Jika Anda mengonversi tipe yang menggunakan ukuran byte yang lebih besar ke ``address``, misalnya ``bytes32``, maka ``address`` akan terpotong.
    Untuk mengurangi ambiguitas konversi versi 0.4.24 dan lebih tinggi dari kekuatan kompiler, Anda membuat pemotongan eksplisit dalam konversi.
    Ambil contoh nilai 32-byte ``0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC``.

    Anda dapat menggunakan ``address(uint160(bytes20(b)))``, yang akan menghasilkan ``0x111122223333444455556666777788889999aAaa``,
    atau anda dapat menggunakan ``address(uint160(uint256(b)))``, yang akan menghasilkan ``0x777788889999AaAAbBbbCcccddDdeeeEfFFfCcCc``.

.. note::
    Perbedaan antara ``address`` dan ``address payable`` diperkenalkan di versi 0.5.0.
    Juga mulai dari versi itu, kontrak tidak diturunkan dari tipe alamat, tetapi masih dapat secara eksplisit dikonversi ke
    ``address`` atau ke ``address payable``, jika mereka memiliki fungsi fallback terima atau *payable*.

.. _members-of-addresses:

Members of Addresses (Anggota Alamat)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Untuk referensi cepat dari semua anggota address, lihat :ref:`address_related`.

* ``balance`` dan ``transfer``

Dimungkinkan untuk menanyakan saldo alamat menggunakan properti ``balance``
dan untuk mengirim Ether (dalam satuan wei) ke alamat yang harus dibayar menggunakan fungsi ``transfer``:

.. code-block:: solidity
    :force:

    address payable x = payable(0x123);
    address myAddress = address(this);
    if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);

Fungsi ``transfer`` gagal jika saldo kontrak saat ini tidak cukup besar
atau jika transfer Ether ditolak oleh akun penerima. Fungsi ``transfer`` reverts saat kegagalan.

.. note::
    Jika ``x`` adalah alamat kontrak, kodenya (lebih spesifik: :ref:`receive-ether-function`, jika ada, atau sebaliknya :ref:`fallback-function`, jika ada) akan dieksekusi bersama dengan panggilan ``transfer`` (ini adalah fitur dari EVM dan tidak dapat dicegah). Jika saat eksekusi kehabisan gas atau gagal dengan cara apa pun, transfer Ether akan dikembalikan dan kontrak saat ini akan berhenti dengan pengecualian.

* ``send``

Send adalah mitra tingkat rendah dari ``transfer``. Jika eksekusi gagal, kontrak saat ini tidak akan berhenti dengan pengecualian, tetapi ``send`` akan kembali ``false``.

.. warning::
    Ada beberapa bahaya dalam menggunakan ``send``: Transfer gagal jika kedalaman tumpukan panggilan berada pada 1024
    (ini selalu dapat dipaksakan oleh penelepon) dan juga gagal jika penerima kehabisan bensin. Jadi untuk melakukan
    transfer Ether yang aman, selalu periksa nilai pengembalian ``send``, gunakan ``transfer`` atau lebih baik lagi:
    gunakan pola di mana penerima menarik uangnya.

* ``call``, ``delegatecall`` dan``staticcall``

Untuk berinteraksi dengan kontrak yang tidak mematuhi ABI,
atau untuk mendapatkan kontrol lebih langsung ke encoding,
fungsi ``call``, ``delegatecall`` dan ``staticcall`` disediakan.
Mereka semua mengambil satu parameter ``bytes memory`` dan
mengembalikan kondisi sukses (sebagai ``bool``) dan data yang
dikembalikan (``byte memory``).
Fungsi ``abi.encode``, ``abi.encodePacked``, ``abi.encodeWithSelector``
dan ``abi.encodeWithSignature`` dapat digunakan untuk mengkodekan data terstruktur.

Contoh:

.. code-block:: solidity

    bytes memory payload = abi.encodeWithSignature("register(string)", "MyName");
    (bool success, bytes memory returnData) = address(nameReg).call(payload);
    require(success);

.. warning::
    Semua fungsi ini adalah fungsi low-level dan harus digunakan dengan hati-hati.
    Secara khusus, kontrak yang tidak dikenal mungkin berbahaya dan jika Anda memanggilnya,
    Anda menyerahkan kendali ke kontrak itu yang pada gilirannya dapat memanggil kembali ke
    dalam kontrak Anda, jadi bersiaplah untuk perubahan pada variabel state Anda saat panggilan kembali.
    Cara biasa untuk berinteraksi dengan kontrak lain adalah dengan memanggil fungsi pada objek kontrak (``x.f()``).

.. note::
    Versi Solidity sebelumnya mengizinkan fungsi-fungsi ini untuk menerima argumen arbitrer
    dan juga akan menangani argumen pertama bertipe ``bytes4`` secara berbeda. Kasus edge
    ini telah dihapus di versi 0.5.0.

Dimungkinkan untuk menyesuaikan gas yang disediakan dengan pengubah ``gas``:

.. code-block:: solidity

    address(nameReg).call{gas: 1000000}(abi.encodeWithSignature("register(string)", "MyName"));

Demikian pula, nilai Ether yang disediakan juga dapat dikontrol:

.. code-block:: solidity

    address(nameReg).call{value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

Terakhir, modifier ini dapat digabungkan. Urutannya tidak masalah:

.. code-block:: solidity

    address(nameReg).call{gas: 1000000, value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

Dengan cara yang sama, fungsi ``delegatecall`` dapat digunakan: perbedaannya adalah hanya kode alamat yang diberikan yang digunakan, semua aspek lain (storage, balance, ...) diambil dari kontrak saat ini. Tujuan dari ``delegatecall`` adalah untuk menggunakan kode library yang disimpan dalam kontrak lain. Pengguna harus memastikan bahwa tata letak penyimpanan di kedua kontrak cocok untuk panggilan delegasi yang akan digunakan.

.. note::
    Sebelum homestead, hanya varian terbatas yang disebut ``callcode`` yang tersedia yang tidak menyediakan akses ke nilai ``msg.sender`` dan ``msg.value`` asli. Fungsi ini telah dihapus di versi 0.5.0.

Sejak byzantium ``staticcall`` dapat digunakan juga. Ini pada dasarnya sama dengan ``call``, tetapi akan dikembalikan jika fungsi yang dipanggil mengubah state dengan cara apa pun.

Ketiga fungsi ``call``, ``delegatecall`` dan ``staticcall`` adalah fungsi tingkat sangat rendah dan hanya boleh digunakan sebagai *pilihan terakhir* karena merusak keamanan tipe Solidity.

Opsi ``gas`` tersedia pada ketiga metode, sedangkan opsi ``nilai`` hanya tersedia
pada ``call``.

.. note::
    Yang terbaik adalah menghindari mengandalkan nilai gas yang dikodekan dalam kode smart kontrak Anda,
    terlepas dari apakah state dibaca atau ditulis, karena ini dapat memiliki banyak jebakan.
    Juga, akses ke gas mungkin berubah di masa depan.

.. note::
    Semua kontrak dapat dikonversi ke tipe ``address``, sehingga memungkinkan untuk menanyakan saldo kontrak
    saat ini menggunakan ``address(this).balance``.

.. index:: ! contract type, ! type; contract

.. _contract_types:

Tipe Kontrak
--------------

Setiap :ref:`kontrak<contracts>` mendefinisikan tipenya sendiri.
Anda dapat secara implisit mengonversi kontrak menjadi kontrak yang mereka warisi.
Kontrak dapat secara eksplisit dikonversi ke dan dari tipe ``address``.

Konversi eksplisit ke dan dari tipe ``address payable`` hanya dimungkinkan jika tipe
kontrak memiliki fungsi fallback payable atau terima. Konversi masih dilakukan
menggunakan ``address(x)``. Jika jenis kontrak tidak memiliki fungsi fallback payable
atau terima, konversi ke ``address payable`` dapat dilakukan menggunakan ``payable(address(x))``.
Anda dapat menemukan informasi lebih lanjut di bagian tentang :ref:`address type<address>`.

.. note::
    Sebelum versi 0.5.0, kontrak langsung diturunkan dari tipe alamat
    dan tidak ada perbedaan antara ``address`` dan ``address payable``.

Jika Anda mendeklarasikan variabel lokal tipe kontrak (``MyContract c``),
Anda dapat memanggil fungsi pada kontrak tersebut. Berhati-hatilah untuk
menetapkannya dari suatu tempat dengan jenis kontrak yang sama.

Anda juga dapat membuat instance kontrak (yang berarti kontrak tersebut baru dibuat).
Anda dapat menemukan detail selengkapnya di bagian :ref:`'Contracts via new'<creating-contracts>`.

Representasi data sebuah kontrak identik dengan representasi ``address``
type dan jenis ini juga digunakan dalam :ref:`ABI<ABI>`.

Kontrak tidak mendukung operator mana pun.

Anggota dari jenis kontrak adalah fungsi eksternal kontrak
termasuk variabel state apa pun yang ditandai sebagai ``public``.

Untuk kontrak ``C`` Anda dapat menggunakan ``type(C)`` untuk mengakses
:ref:`type information<meta-type>` tentang kontrak.

.. index:: byte array, bytes32

Fixed-size byte arrays
----------------------

Jenis nilai ``bytes1``, ``bytes2``, ``bytes3``, ..., ``byte32``
menyimpan urutan byte dari satu hingga 32.

Operator:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (mengevaluasi ke ``bool``)
* Bit operators: ``&``, ``|``, ``^`` (bitwise exclusive atau), ``~`` (negasi bitwise)
* Shift operators: ``<<`` (shift kiri), ``>>`` (shift kanan)
* Index access: Jika ``x`` adalah tipe dari ``bytesI``, maka ``x[k]`` untuk ``0 <= k < I`` mengembalikan byte ke ``k`` (read-only).

Operator shifting bekerja dengan tipe integer unsigned sebagai as operand kanan
(tetapi menghasilkan jenis operand kiri), yang menunjukkan jumlah bit yang akan digeser.
Shifting menggunakan tipe signed akan menghasilkan *compilation error*.

Members:

* ``.length`` menghasilkan panjang tetap dari array byte (read-only).

.. note::
    Tipe ``bytes1[]`` adalah array byte, tapi karena aturan padding, itu membuang
    31 byte ruang untuk setiap elemen (kecuali dalam storage). Lebih baik menggunakan tipe
    ``byte`` sebagai gantinya.

.. note::
    Sebelum versi 0.8.0, ``byte`` digunakan sebagai alias untuk ``bytes1``.

Dynamically-sized byte array
----------------------------

``bytes``:
    Dynamically-sized byte array, lihat :ref:`arrays`. Bukan sebuah value-type!
``string``:
    Dynamically-sized UTF-8-encoded string, lihat :ref:`arrays`. Bukan sebuah value-type!

.. index:: address, literal;address

.. _address_literals:

Address Literal
---------------

Literal heksadesimal yang lulus tes alamat checksum, misalnya
``0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF`` bertipe ``address``.
Literal heksadesimal dengan panjang antara
39 dan 41 digit dan tidak lulus tes checksum menghasilkan
sebuah kesalahan. Anda dapat menambahkan (untuk tipe integer) atau menambahkan (untuk tipe byteNN) nol untuk memperbaiki kesalahan.

.. note::
    Format mixed-case address checksum didefinisikan dalam `EIP-55 <https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md>`_.

.. index:: literal, literal;rational

.. _rational_literals:

Rational dan Integer Literal
----------------------------

Literal integer dibentuk dari urutan angka dalam rentang 0-9.
Mereka ditafsirkan sebagai desimal. Misalnya, ``69`` berarti enam puluh sembilan.
Literal oktal tidak ada dalam Solidity dan angka nol di depan tidak valid.

Literal pecahan desimal dibentuk oleh ``.`` dengan setidaknya satu angka di satu sisi.
Contohnya termasuk ``1.``, ``.1`` dan ``1.3``.

Notasi ilmiah juga didukung, di mana basis dapat memiliki pecahan dan eksponen tidak bisa.
Contohnya termasuk ``2e10``, ``-2e10``, ``2e-10``, ``2.5e1``.

Garis bawah dapat digunakan untuk memisahkan digit literal numerik agar mudah dibaca.
Misalnya, desimal ``123_000``, heksadesimal ``0x2eff_abde``, notasi desimal ilmiah ``1_2e345_678`` semuanya valid.
Garis bawah hanya diperbolehkan antara dua digit dan hanya satu garis bawah berurutan yang diperbolehkan.
Tidak ada makna semantik tambahan yang ditambahkan ke angka literal yang mengandung garis bawah,
garis bawah diabaikan.

mempertahankan presisi arbitrer hingga dikonversi ke tipe non-literal (yaitu dengan
menggunakannya bersama dengan ekspresi non-literal atau dengan konversi eksplisit).
Ini berarti bahwa komputasi tidak overflow dan pembagian tidak terpotong
dalam number literal expressions.

Sebagai contoh, ``(2**800 + 1) - 2**800`` menghasilkan konstanta ``1`` (dari tipe ``uint8``)
ameskipun hasilnya antara atau bahkan tidak sesuai dengan ukuran *machine word*. Selanjutnya, hasil ``.5 * 8``
dalam integer ``4`` (walaupun non-integer bulat digunakan di antaranya).

Operator apa pun yang dapat diterapkan ke integer juga dapat diterapkan ke number literal expressions
selama operand adalah integer. Jika salah satu dari keduanya adalah pecahan, operasi bit tidak diizinkan
dan eksponensial tidak diizinkan jika eksponennya pecahan (karena itu mungkin menghasilkan
bilangan non-rasional)

Shifts dan exponentiation dengan angka literal sebagai operand kiri (atau basis) dan tipe integer
sebagai operand kanan (eksponen) selalu dilakukan
dalam tipe ``uint256`` (untuk literal non-negatif) atau ``int256`` (untuk literal negatif),
terlepas dari jenis operand kanan (eksponen).

.. warning::
    Division pada literal integer yang digunakan untuk memotong di Solidity sebelum versi 0.4.0, tetapi sekarang diubah menjadi bilangan rasional, yaitu ``5 / 2`` tidak sama dengan ``2``, tetapi menjadi ``2,5`` .

.. note::
    Solidity memiliki tipe literal bilangan untuk setiap bilangan rasional.
    Literal integer dan literal bilangan rasional termasuk dalam tipe literal bilangan.
    Selain itu, semua ekspresi literal angka (yaitu ekspresi yang hanya berisi literal
    angka dan operator) termasuk dalam tipe literal angka. Jadi number literal expressions
    ``1 + 2`` dan ``2 + 1`` keduanya termasuk dalam tipe literal bilangan yang sama untuk
    bilangan rasional tiga.


.. note::
    Number literal expressions diubah menjadi tipe non-literal segera setelah digunakan dengan ekspresi
    non-literal. Mengabaikan jenis, nilai ekspresi yang ditetapkan ke ``b`` di bawah
    ini dievaluasi menjadi integer. Karena ``a`` bertipe ``uint128``, maka
    ekspresi ``2.5 + a`` harus memiliki tipe yang tepat. Karena tidak ada tipe yang umum
    untuk tipe ``2.5`` dan ``uint128``, compiler Solidity tidak menerima
    kode ini.

.. code-block:: solidity

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string
.. _string_literals:

String Literals dan Tipe
-------------------------

String literals ditulis dengan double atau single-quotes (``"foo"`` atau ``'bar'``), dan mereka juga dapat dipecah menjadi beberapa bagian berurutan (``"foo" "bar"`` setara dengan ``"foobar"``) yang dapat membantu saat menangani string panjang. Mereka tidak menyiratkan nol trailing seperti di C; ``"foo"`` mewakili tiga byte, bukan empat. Seperti literal integer, tipenya dapat bervariasi, tetapi secara implisit dapat dikonversi ke ``bytes1``, ..., ``bytes32``, jika cocok, ke ``byte`` dan ke ``string``.

Misalnya, dengan ``bytes32 samevar = "stringliteral"``, literal string diinterpretasikan dalam bentuk byte mentahnya saat ditetapkan ke tipe ``bytes32``.

Literal string hanya dapat berisi karakter ASCII yang dapat dicetak, yang berarti karakter antara dan termasuk 0x1F .. 0x7E.

Selain itu, literal string juga mendukung karakter escape berikut:

- ``\<newline>`` (escapes an actual newline)
- ``\\`` (backslash)
- ``\'`` (single quote)
- ``\"`` (double quote)
- ``\n`` (newline)
- ``\r`` (carriage return)
- ``\t`` (tab)
- ``\xNN`` (hex escape, see below)
- ``\uNNNN`` (unicode escape, see below)

``\xNN`` mengambil nilai hex dan menyisipkan byte yang sesuai, sementara ``\uNNNN`` mengambil titik kode Unicode dan menyisipkan urutan UTF-8.

.. note::

    Hingga versi 0.8.0 ada tiga urutan escape tambahan: ``\b``, ``\f`` dan ``\v``.
    Mereka umumnya tersedia dalam bahasa lain tetapi jarang dibutuhkan dalam praktiknya.
    Jika Anda memang membutuhkannya, mereka masih dapat dimasukkan melalui escape heksadesimal, yaitu ``\x08``, ``\x0c``
    dan ``\x0b``, masing-masing, sama seperti karakter ASCII lainnya.

String dalam contoh berikut memiliki panjang sepuluh byte.
Ini dimulai dengan newline byte, diikuti dengan tanda kutip ganda,
tanda kutip tunggal karakter garis miring terbalik dan kemudian (tanpa pemisah)
urutan karakter ``abcdef``.

.. code-block:: solidity
    :force:

    "\n\"\'\\abc\
    def"

Setiap terminator baris Unicode yang bukan merupakan baris baru (yaitu LF, VF, FF, CR, NEL, LS, PS) dianggap
mengakhiri string literal. Baris baru hanya mengakhiri literal string jika tidak didahului oleh ``\``.

Literal Unicode
---------------

Sementara literal string biasa hanya dapat berisi ASCII, literal Unicode – diawali dengan kata kunci ``unicode`` – dapat berisi urutan UTF-8 yang valid.
Mereka juga mendukung urutan escape yang sama seperti literal string biasa.

.. code-block:: solidity

    string memory a = unicode"Hello 😃";

.. index:: literal, bytes

Literal Hexadecimal
-------------------

Literal heksadesimal diawali dengan kata kunci ``hex`` dan diapit dua kali
atau tanda kutip tunggal (``hex"001122FF"``, ``hex'0011_22_FF'``). Konten mereka harus
digit heksadesimal yang secara opsional dapat menggunakan garis bawah tunggal sebagai pemisah antara
batas byte. Nilai literal akan menjadi representasi biner
dari barisan heksadesimal.

Beberapa literal heksadesimal yang dipisahkan oleh spasi digabung menjadi satu literal:
``hex"00112233" hex"44556677"`` setara dengan ``hex"0011223344556677"``

Literal heksadesimal berperilaku seperti :ref:`string literal <string_literals>` dan memiliki batasan konvertibilitas yang sama.

.. index:: enum

.. _enums:

Enums
-----

Enum adalah salah satu cara untuk membuat tipe *user-defined* di Solidity.
Mereka secara eksplisit dapat dikonversi ke dan dari semua tipe integer tetapi konversi implisit tidak diperbolehkan.
Konversi eksplisit dari integer memeriksa saat runtime bahwa nilainya berada di dalam rentang enum dan menyebabkan :ref:`Panic error<assert-and-require>` sebaliknya.
Enum membutuhkan setidaknya satu anggota, dan nilai defaultnya saat dideklarasikan adalah anggota pertama.
Enum tidak boleh memiliki lebih dari 256 anggota.

Representasi data sama dengan enum di C: Opsi diwakili oleh nilai integer tak bertanda berikutnya yang dimulai dari ``0``.

Menggunakan ``type(NameOfEnum).min`` dan ``type(NameOfEnum).max`` Anda bisa mendapatkan nilai
terkecil dan terbesar dari enum yang diberikan.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.8;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() public {
            choice = ActionChoices.GoStraight;
        }

        // Since enum types are not part of the ABI, the signature of "getChoice"
        // will automatically be changed to "getChoice() returns (uint8)"
        // for all matters external to Solidity.
        function getChoice() public view returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() public pure returns (uint) {
            return uint(defaultChoice);
        }

        function getLargestValue() public pure returns (ActionChoices) {
            return type(ActionChoices).max;
        }

        function getSmallestValue() public pure returns (ActionChoices) {
            return type(ActionChoices).min;
        }
    }

.. note::
    Enum juga dapat dideklarasikan pada level file, di luar definisi kontrak atau library.

.. index:: ! user defined value type, custom type

.. _user-defined-value-types:

Tipe Nilai *user defined* (Ditentukan oleh pengguna)
-----------------------------------------------------

Tipe nilai yang ditentukan pengguna memungkinkan pembuatan abstraksi tanpa biaya di atas tipe nilai elementary.
Ini mirip dengan alias, tetapi dengan persyaratan tipe yang lebih ketat.

Tipe nilai yang ditentukan pengguna didefinisikan menggunakan ``type C is V``,
di mana ``C`` adalah nama tipe yang baru diperkenalkan dan ``V`` harus berupa tipe nilai bawaan ("underlying type").
Fungsi ``C.wrap`` digunakan untuk mengonversi dari tipe underlying ke tipe kustom.
Demikian pula, fungsi ``C.unwrap`` digunakan untuk mengonversi dari tipe kustom ke tipe underlying.

Tipe ``C`` tidak memiliki operator atau fungsi anggota terikat.
Secara khusus, bahkan operator ``==`` tidak didefinisikan.
Konversi eksplisit dan implisit ke dan dari jenis lain tidak diizinkan.

Representasi data dari nilai tipe tersebut diwarisi dari tipe underlying
dan tipe underlying juga digunakan dalam ABI.

Contoh berikut mengilustrasikan tipe kustom ``UFixed256x18`` yang mewakili tipe *decimal fixed point* dengan 18 desimal
dan sebuah library minimal untuk melakukan operasi aritmatika pada tipe tersebut.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.8;

    // Represent a 18 decimal, 256 bit wide fixed point type using a user defined value type.
    type UFixed256x18 is uint256;

    /// A minimal library to do fixed point operations on UFixed256x18.
    library FixedMath {
        uint constant multiplier = 10**18;

        /// Adds two UFixed256x18 numbers. Reverts on overflow, relying on checked
        /// arithmetic on uint256.
        function add(UFixed256x18 a, UFixed256x18 b) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(UFixed256x18.unwrap(a) + UFixed256x18.unwrap(b));
        }
        /// Multiplies UFixed256x18 and uint256. Reverts on overflow, relying on checked
        /// arithmetic on uint256.
        function mul(UFixed256x18 a, uint256 b) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(UFixed256x18.unwrap(a) * b);
        }
        /// Take the floor of a UFixed256x18 number.
        /// @return the largest integer that does not exceed `a`.
        function floor(UFixed256x18 a) internal pure returns (uint256) {
            return UFixed256x18.unwrap(a) / multiplier;
        }
        /// Turns a uint256 into a UFixed256x18 of the same value.
        /// Reverts if the integer is too large.
        function toUFixed256x18(uint256 a) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(a * multiplier);
        }
    }

Perhatikan bagaimana ``UFixed256x18.wrap`` dan ``FixedMath.toUFixed256x18`` memiliki tanda tangan
yang sama tetapi melakukan dua operasi yang sangat berbeda: Fungsi ``UFixed256x18.wrap`` menghasilkan
``UFixed256x18`` yang memiliki representasi data yang sama sebagai input, sedangkan ``toUFixed256x18``
menghasilkan ``UFixed256x18`` yang memiliki nilai numerik yang sama.

.. index:: ! function type, ! type; function

.. _function_types:

Function Types (Tipe Fungsi)
----------------------------

Function types adalah tipe-tipe fungsi. Variabel Function types dapat
ditetapkan oleh fungsi dan parameter fungsi dari Function types dapat
digunakan untuk meneruskan *fungsi to dan fungsi return* dari fungsi calls.
Function types datang dalam dua jenis - fungsi *internal* dan *eksternal*:

Fungsi internal hanya dapat dipanggil di dalam kontrak saat ini (lebih khusus,
di dalam unit kode saat ini, yang juga mencakup fungsi library internal dan fungsi inherited)
karena mereka tidak dapat dieksekusi di luar konteks kontrak saat ini.
Memanggil fungsi internal diwujudkan dengan melompat ke label entrinya, sama seperti
saat memanggil fungsi kontrak saat ini secara internal.

Fungsi eksternal terdiri dari alamat dan fungsi tanda tangan serta dapat diteruskan
dan dikembalikan dari panggilan fungsi eksternal.

Function types dinotasikan sebagai berikut:

.. code-block:: solidity
    :force:

    function (<parameter types>) {internal|external} [pure|view|payable] [returns (<return types>)]

Berbeda dengan tipe parameter, tipe return tidak boleh kosong - jika function
type tidak boleh mengembalikan apa pun, seluruh bagian ``returns (<return types>)``
harus dihilangkan.

Secara default, function types bersifat internal, sehingga kata kunci ``internal``
dapat dihilangkan. Perhatikan bahwa ini hanya berlaku untuk function types.
Visibilitas harus ditentukan secara eksplisit untuk fungsi yang didefinisikan dalam kontrak,
mereka tidak memiliki default.

Konversi:

Function type ``A`` secara implisit dapat dikonversi menjadi function type ``B`` jika
dan hanya jika tipe parameternya identik, tipe return identik, properti internal/eksternalnya identik, dan
status mutabilitas `` A`` lebih restriktif daripada mutabilitas status ``B``. Secara khusus:

- fungsi  ``pure`` dapat dikonversi menjadi ``view`` dann fungsi ``non-payable``
- fungsi ``view`` dapat dikonversi menjadi fungsi ``non-payable``
- fungsi ``payable`` dapat dikonversi menjadi fungsi ``non-payable``

Tidak ada konversi lain antara function types yang mungkinkan.

Aturan tentang ``payable`` dan ``non-payable`` mungkin sedikit
membingungkan, tetapi pada intinya, jika suatu fungsi adalah ``payable``, ini berarti bahwa itu
juga dapat menerima pembayaran nol Ether, jadi ini juga ``non-payable``.
Di sisi lain, fungsi ``non-payable`` akan menolak Ether yang dikirim ke sana,
jadi fungsi ``non-payable`` tidak dapat dikonversi ke fungsi ``payable``.

Jika variabel sebuah function type tidak diinialisasi, memanggilnya akan mengasilkan
:ref:`Panic error<assert-and-require>`. Hal yang sama terjadi jika Anda memanggil fungsi setelah menggunakan ``delete``
di dalamnya.

Jika function types eksternal digunakan di luar konteks Solidity,
mereka diperlakukan sebagai tipe ``function``, yang mengkodekan alamat yang diikuti oleh
pengidentifikasi fungsi bersama-sama dalam satu tipe ``bytes24``.

Perhatikan bahwa fungsi publik dari kontrak saat ini dapat digunakan baik sebagai fungsi
internal maupun sebagai fungsi eksternal. Untuk menggunakan ``f`` sebagai fungsi internal,
cukup gunakan ``f``, jika Anda ingin menggunakan bentuk eksternal, gunakan ``this.f``.

Fungsi dari tipe internal dapat ditetapkan ke variabel function type internal terlepas dari mana itu didefinisikan.
Ini termasuk fungsi pribadi, internal dan publik dari kontrak dan  liblary serta fungsi bebas.
function types eksternal, di sisi lain, hanya kompatibel dengan fungsi kontrak publik dan eksternal.
Library dikecualikan karena memerlukan ``delegatecall`` dan menggunakan :ref:`konvensi ABI yang berbeda
untuk pemilihnya <library-selectors>`.
Fungsi yang dideklarasikan dalam antarmuka tidak memiliki definisi sehingga menunjuknya juga tidak masuk akal.

Members (Anggota):

Fungsi External (atau public) memiliki anggota sebagai berikut:

* ``.address`` menghasilkan alamat kontrak dari fungsi tersebut.
* ``.selector`` menghasilkan :ref:`Pemilih fungsi ABI <abi_function_selector>`

.. note::
  Fungsi External (atau public) yang digunakan untuk memiliki anggota tambahan
  ``.gas(uint)`` dan ``.value(uint)``. Ini tidak digunakan lagi di Solidity 0.6.2
  dan dihapus di Solidity 0.7.0. Sebagai gantinya gunakan ``{gas: ...}`` dan ``{value: ...}``
  untuk menentukan jumlah gas atau jumlah wei yang dikirim ke suatu fungsi,
  secara masing-masing. Lihat :ref:`External Function Calls <external-function-calls>` untuk
  informasi lebih lanjut.

Contoh yang menunjukkan cara menggunakan member:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.4 <0.9.0;

    contract Example {
        function f() public payable returns (bytes4) {
            assert(this.f.address == address(this));
            return this.f.selector;
        }

        function g() public {
            this.f{gas: 10, value: 800}();
        }
    }

Contoh yang menunjukkan cara menggunakan function types internal:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    library ArrayUtils {
        // internal functions can be used in internal library functions because
        // they will be part of the same code context
        function map(uint[] memory self, function (uint) pure returns (uint) f)
            internal
            pure
            returns (uint[] memory r)
        {
            r = new uint[](self.length);
            for (uint i = 0; i < self.length; i++) {
                r[i] = f(self[i]);
            }
        }

        function reduce(
            uint[] memory self,
            function (uint, uint) pure returns (uint) f
        )
            internal
            pure
            returns (uint r)
        {
            r = self[0];
            for (uint i = 1; i < self.length; i++) {
                r = f(r, self[i]);
            }
        }

        function range(uint length) internal pure returns (uint[] memory r) {
            r = new uint[](length);
            for (uint i = 0; i < r.length; i++) {
                r[i] = i;
            }
        }
    }


    contract Pyramid {
        using ArrayUtils for *;

        function pyramid(uint l) public pure returns (uint) {
            return ArrayUtils.range(l).map(square).reduce(sum);
        }

        function square(uint x) internal pure returns (uint) {
            return x * x;
        }

        function sum(uint x, uint y) internal pure returns (uint) {
            return x + y;
        }
    }

Contoh lain yang menggunakan function types eksternal:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.22 <0.9.0;


    contract Oracle {
        struct Request {
            bytes data;
            function(uint) external callback;
        }

        Request[] private requests;
        event NewRequest(uint);

        function query(bytes memory data, function(uint) external callback) public {
            requests.push(Request(data, callback));
            emit NewRequest(requests.length - 1);
        }

        function reply(uint requestID, uint response) public {
            // Here goes the check that the reply comes from a trusted source
            requests[requestID].callback(response);
        }
    }


    contract OracleUser {
        Oracle constant private ORACLE_CONST = Oracle(address(0x00000000219ab540356cBB839Cbe05303d7705Fa)); // known contract
        uint private exchangeRate;

        function buySomething() public {
            ORACLE_CONST.query("USD", this.oracleResponse);
        }

        function oracleResponse(uint response) public {
            require(
                msg.sender == address(ORACLE_CONST),
                "Only oracle can call this."
            );
            exchangeRate = response;
        }
    }

.. note::
    Lambda atau fungsi inline direncanakan tetapi belum didukung.
=======
.. index:: ! value type, ! type;value
.. _value-types:

Value Types
===========

The following types are also called value types because variables of these
types will always be passed by value, i.e. they are always copied when they
are used as function arguments or in assignments.

.. index:: ! bool, ! true, ! false

Booleans
--------

``bool``: The possible values are constants ``true`` and ``false``.

Operators:

*  ``!`` (logical negation)
*  ``&&`` (logical conjunction, "and")
*  ``||`` (logical disjunction, "or")
*  ``==`` (equality)
*  ``!=`` (inequality)

The operators ``||`` and ``&&`` apply the common short-circuiting rules. This means that in the expression ``f(x) || g(y)``, if ``f(x)`` evaluates to ``true``, ``g(y)`` will not be evaluated even if it may have side-effects.

.. index:: ! uint, ! int, ! integer
.. _integers:

Integers
--------

``int`` / ``uint``: Signed and unsigned integers of various sizes. Keywords ``uint8`` to ``uint256`` in steps of ``8`` (unsigned of 8 up to 256 bits) and ``int8`` to ``int256``. ``uint`` and ``int`` are aliases for ``uint256`` and ``int256``, respectively.

Operators:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evaluate to ``bool``)
* Bit operators: ``&``, ``|``, ``^`` (bitwise exclusive or), ``~`` (bitwise negation)
* Shift operators: ``<<`` (left shift), ``>>`` (right shift)
* Arithmetic operators: ``+``, ``-``, unary ``-`` (only for signed integers), ``*``, ``/``, ``%`` (modulo), ``**`` (exponentiation)

For an integer type ``X``, you can use ``type(X).min`` and ``type(X).max`` to
access the minimum and maximum value representable by the type.

.. warning::

  Integers in Solidity are restricted to a certain range. For example, with ``uint32``, this is ``0`` up to ``2**32 - 1``.
  There are two modes in which arithmetic is performed on these types: The "wrapping" or "unchecked" mode and the "checked" mode.
  By default, arithmetic is always "checked", which mean that if the result of an operation falls outside the value range
  of the type, the call is reverted through a :ref:`failing assertion<assert-and-require>`. You can switch to "unchecked" mode
  using ``unchecked { ... }``. More details can be found in the section about :ref:`unchecked <unchecked>`.

Comparisons
^^^^^^^^^^^

The value of a comparison is the one obtained by comparing the integer value.

Bit operations
^^^^^^^^^^^^^^

Bit operations are performed on the two's complement representation of the number.
This means that, for example ``~int256(0) == int256(-1)``.

Shifts
^^^^^^

The result of a shift operation has the type of the left operand, truncating the result to match the type.
The right operand must be of unsigned type, trying to shift by a signed type will produce a compilation error.

Shifts can be "simulated" using multiplication by powers of two in the following way. Note that the truncation
to the type of the left operand is always performed at the end, but not mentioned explicitly.

- ``x << y`` is equivalent to the mathematical expression ``x * 2**y``.
- ``x >> y`` is equivalent to the mathematical expression ``x / 2**y``, rounded towards negative infinity.

.. warning::
    Before version ``0.5.0`` a right shift ``x >> y`` for negative ``x`` was equivalent to
    the mathematical expression ``x / 2**y`` rounded towards zero,
    i.e., right shifts used rounding up (towards zero) instead of rounding down (towards negative infinity).

.. note::
    Overflow checks are never performed for shift operations as they are done for arithmetic operations.
    Instead, the result is always truncated.

Addition, Subtraction and Multiplication
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Addition, subtraction and multiplication have the usual semantics, with two different
modes in regard to over- and underflow:

By default, all arithmetic is checked for under- or overflow, but this can be disabled
using the :ref:`unchecked block<unchecked>`, resulting in wrapping arithmetic. More details
can be found in that section.

The expression ``-x`` is equivalent to ``(T(0) - x)`` where
``T`` is the type of ``x``. It can only be applied to signed types.
The value of ``-x`` can be
positive if ``x`` is negative. There is another caveat also resulting
from two's complement representation:

If you have ``int x = type(int).min;``, then ``-x`` does not fit the positive range.
This means that ``unchecked { assert(-x == x); }`` works, and the expression ``-x``
when used in checked mode will result in a failing assertion.

Division
^^^^^^^^

Since the type of the result of an operation is always the type of one of
the operands, division on integers always results in an integer.
In Solidity, division rounds towards zero. This means that ``int256(-5) / int256(2) == int256(-2)``.

Note that in contrast, division on :ref:`literals<rational_literals>` results in fractional values
of arbitrary precision.

.. note::
  Division by zero causes a :ref:`Panic error<assert-and-require>`. This check can **not** be disabled through ``unchecked { ... }``.

.. note::
  The expression ``type(int).min / (-1)`` is the only case where division causes an overflow.
  In checked arithmetic mode, this will cause a failing assertion, while in wrapping
  mode, the value will be ``type(int).min``.

Modulo
^^^^^^

The modulo operation ``a % n`` yields the remainder ``r`` after the division of the operand ``a``
by the operand ``n``, where ``q = int(a / n)`` and ``r = a - (n * q)``. This means that modulo
results in the same sign as its left operand (or zero) and ``a % n == -(-a % n)`` holds for negative ``a``:

* ``int256(5) % int256(2) == int256(1)``
* ``int256(5) % int256(-2) == int256(1)``
* ``int256(-5) % int256(2) == int256(-1)``
* ``int256(-5) % int256(-2) == int256(-1)``

.. note::
  Modulo with zero causes a :ref:`Panic error<assert-and-require>`. This check can **not** be disabled through ``unchecked { ... }``.

Exponentiation
^^^^^^^^^^^^^^

Exponentiation is only available for unsigned types in the exponent. The resulting type
of an exponentiation is always equal to the type of the base. Please take care that it is
large enough to hold the result and prepare for potential assertion failures or wrapping behaviour.

.. note::
  In checked mode, exponentiation only uses the comparatively cheap ``exp`` opcode for small bases.
  For the cases of ``x**3``, the expression ``x*x*x`` might be cheaper.
  In any case, gas cost tests and the use of the optimizer are advisable.

.. note::
  Note that ``0**0`` is defined by the EVM as ``1``.

.. index:: ! ufixed, ! fixed, ! fixed point number

Fixed Point Numbers
-------------------

.. warning::
    Fixed point numbers are not fully supported by Solidity yet. They can be declared, but
    cannot be assigned to or from.

``fixed`` / ``ufixed``: Signed and unsigned fixed point number of various sizes. Keywords ``ufixedMxN`` and ``fixedMxN``, where ``M`` represents the number of bits taken by
the type and ``N`` represents how many decimal points are available. ``M`` must be divisible by 8 and goes from 8 to 256 bits. ``N`` must be between 0 and 80, inclusive.
``ufixed`` and ``fixed`` are aliases for ``ufixed128x18`` and ``fixed128x18``, respectively.

Operators:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evaluate to ``bool``)
* Arithmetic operators: ``+``, ``-``, unary ``-``, ``*``, ``/``, ``%`` (modulo)

.. note::
    The main difference between floating point (``float`` and ``double`` in many languages, more precisely IEEE 754 numbers) and fixed point numbers is
    that the number of bits used for the integer and the fractional part (the part after the decimal dot) is flexible in the former, while it is strictly
    defined in the latter. Generally, in floating point almost the entire space is used to represent the number, while only a small number of bits define
    where the decimal point is.

.. index:: address, balance, send, call, delegatecall, staticcall, transfer

.. _address:

Address
-------

The address type comes in two flavours, which are largely identical:

- ``address``: Holds a 20 byte value (size of an Ethereum address).
- ``address payable``: Same as ``address``, but with the additional members ``transfer`` and ``send``.

The idea behind this distinction is that ``address payable`` is an address you can send Ether to,
while you are not supposed to send Ether to a plain ``address``, for example because it might be a smart contract
that was not built to accept Ether.

Type conversions:

Implicit conversions from ``address payable`` to ``address`` are allowed, whereas conversions from ``address`` to ``address payable``
must be explicit via ``payable(<address>)``.

Explicit conversions to and from ``address`` are allowed for ``uint160``, integer literals,
``bytes20`` and contract types.

Only expressions of type ``address`` and contract-type can be converted to the type ``address
payable`` via the explicit conversion ``payable(...)``. For contract-type, this conversion is only
allowed if the contract can receive Ether, i.e., the contract either has a :ref:`receive
<receive-ether-function>` or a payable fallback function. Note that ``payable(0)`` is valid and is
an exception to this rule.

.. note::
    If you need a variable of type ``address`` and plan to send Ether to it, then
    declare its type as ``address payable`` to make this requirement visible. Also,
    try to make this distinction or conversion as early as possible.

Operators:

* ``<=``, ``<``, ``==``, ``!=``, ``>=`` and ``>``

.. warning::
    If you convert a type that uses a larger byte size to an ``address``, for example ``bytes32``, then the ``address`` is truncated.
    To reduce conversion ambiguity version 0.4.24 and higher of the compiler force you make the truncation explicit in the conversion.
    Take for example the 32-byte value ``0x111122223333444455556666777788889999AAAABBBBCCCCDDDDEEEEFFFFCCCC``.

    You can use ``address(uint160(bytes20(b)))``, which results in ``0x111122223333444455556666777788889999aAaa``,
    or you can use ``address(uint160(uint256(b)))``, which results in ``0x777788889999AaAAbBbbCcccddDdeeeEfFFfCcCc``.

.. note::
    The distinction between ``address`` and ``address payable`` was introduced with version 0.5.0.
    Also starting from that version, contracts do not derive from the address type, but can still be explicitly converted to
    ``address`` or to ``address payable``, if they have a receive or payable fallback function.

.. _members-of-addresses:

Members of Addresses
^^^^^^^^^^^^^^^^^^^^

For a quick reference of all members of address, see :ref:`address_related`.

* ``balance`` and ``transfer``

It is possible to query the balance of an address using the property ``balance``
and to send Ether (in units of wei) to a payable address using the ``transfer`` function:

.. code-block:: solidity
    :force:

    address payable x = payable(0x123);
    address myAddress = address(this);
    if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);

The ``transfer`` function fails if the balance of the current contract is not large enough
or if the Ether transfer is rejected by the receiving account. The ``transfer`` function
reverts on failure.

.. note::
    If ``x`` is a contract address, its code (more specifically: its :ref:`receive-ether-function`, if present, or otherwise its :ref:`fallback-function`, if present) will be executed together with the ``transfer`` call (this is a feature of the EVM and cannot be prevented). If that execution runs out of gas or fails in any way, the Ether transfer will be reverted and the current contract will stop with an exception.

* ``send``

Send is the low-level counterpart of ``transfer``. If the execution fails, the current contract will not stop with an exception, but ``send`` will return ``false``.

.. warning::
    There are some dangers in using ``send``: The transfer fails if the call stack depth is at 1024
    (this can always be forced by the caller) and it also fails if the recipient runs out of gas. So in order
    to make safe Ether transfers, always check the return value of ``send``, use ``transfer`` or even better:
    use a pattern where the recipient withdraws the money.

* ``call``, ``delegatecall`` and ``staticcall``

In order to interface with contracts that do not adhere to the ABI,
or to get more direct control over the encoding,
the functions ``call``, ``delegatecall`` and ``staticcall`` are provided.
They all take a single ``bytes memory`` parameter and
return the success condition (as a ``bool``) and the returned data
(``bytes memory``).
The functions ``abi.encode``, ``abi.encodePacked``, ``abi.encodeWithSelector``
and ``abi.encodeWithSignature`` can be used to encode structured data.

Example:

.. code-block:: solidity

    bytes memory payload = abi.encodeWithSignature("register(string)", "MyName");
    (bool success, bytes memory returnData) = address(nameReg).call(payload);
    require(success);

.. warning::
    All these functions are low-level functions and should be used with care.
    Specifically, any unknown contract might be malicious and if you call it, you
    hand over control to that contract which could in turn call back into
    your contract, so be prepared for changes to your state variables
    when the call returns. The regular way to interact with other contracts
    is to call a function on a contract object (``x.f()``).

.. note::
    Previous versions of Solidity allowed these functions to receive
    arbitrary arguments and would also handle a first argument of type
    ``bytes4`` differently. These edge cases were removed in version 0.5.0.

It is possible to adjust the supplied gas with the ``gas`` modifier:

.. code-block:: solidity

    address(nameReg).call{gas: 1000000}(abi.encodeWithSignature("register(string)", "MyName"));

Similarly, the supplied Ether value can be controlled too:

.. code-block:: solidity

    address(nameReg).call{value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

Lastly, these modifiers can be combined. Their order does not matter:

.. code-block:: solidity

    address(nameReg).call{gas: 1000000, value: 1 ether}(abi.encodeWithSignature("register(string)", "MyName"));

In a similar way, the function ``delegatecall`` can be used: the difference is that only the code of the given address is used, all other aspects (storage, balance, ...) are taken from the current contract. The purpose of ``delegatecall`` is to use library code which is stored in another contract. The user has to ensure that the layout of storage in both contracts is suitable for delegatecall to be used.

.. note::
    Prior to homestead, only a limited variant called ``callcode`` was available that did not provide access to the original ``msg.sender`` and ``msg.value`` values. This function was removed in version 0.5.0.

Since byzantium ``staticcall`` can be used as well. This is basically the same as ``call``, but will revert if the called function modifies the state in any way.

All three functions ``call``, ``delegatecall`` and ``staticcall`` are very low-level functions and should only be used as a *last resort* as they break the type-safety of Solidity.

The ``gas`` option is available on all three methods, while the ``value`` option is only available
on ``call``.

.. note::
    It is best to avoid relying on hardcoded gas values in your smart contract code,
    regardless of whether state is read from or written to, as this can have many pitfalls.
    Also, access to gas might change in the future.

* ``code`` and ``codehash``

You can query the deployed code for any smart contract. Use ``.code`` to get the EVM bytecode as a
``bytes memory``, which might be empty. Use ``.codehash`` get the Keccak-256 hash of that code
(as a ``bytes32``). Note that ``addr.codehash`` is cheaper than using ``keccak256(addr.code)``.

.. note::
    All contracts can be converted to ``address`` type, so it is possible to query the balance of the
    current contract using ``address(this).balance``.

.. index:: ! contract type, ! type; contract

.. _contract_types:

Contract Types
--------------

Every :ref:`contract<contracts>` defines its own type.
You can implicitly convert contracts to contracts they inherit from.
Contracts can be explicitly converted to and from the ``address`` type.

Explicit conversion to and from the ``address payable`` type is only possible
if the contract type has a receive or payable fallback function.  The conversion is still
performed using ``address(x)``. If the contract type does not have a receive or payable
fallback function, the conversion to ``address payable`` can be done using
``payable(address(x))``.
You can find more information in the section about
the :ref:`address type<address>`.

.. note::
    Before version 0.5.0, contracts directly derived from the address type
    and there was no distinction between ``address`` and ``address payable``.

If you declare a local variable of contract type (``MyContract c``), you can call
functions on that contract. Take care to assign it from somewhere that is the
same contract type.

You can also instantiate contracts (which means they are newly created). You
can find more details in the :ref:`'Contracts via new'<creating-contracts>`
section.

The data representation of a contract is identical to that of the ``address``
type and this type is also used in the :ref:`ABI<ABI>`.

Contracts do not support any operators.

The members of contract types are the external functions of the contract
including any state variables marked as ``public``.

For a contract ``C`` you can use ``type(C)`` to access
:ref:`type information<meta-type>` about the contract.

.. index:: byte array, bytes32

Fixed-size byte arrays
----------------------

The value types ``bytes1``, ``bytes2``, ``bytes3``, ..., ``bytes32``
hold a sequence of bytes from one to up to 32.

Operators:

* Comparisons: ``<=``, ``<``, ``==``, ``!=``, ``>=``, ``>`` (evaluate to ``bool``)
* Bit operators: ``&``, ``|``, ``^`` (bitwise exclusive or), ``~`` (bitwise negation)
* Shift operators: ``<<`` (left shift), ``>>`` (right shift)
* Index access: If ``x`` is of type ``bytesI``, then ``x[k]`` for ``0 <= k < I`` returns the ``k`` th byte (read-only).

The shifting operator works with unsigned integer type as right operand (but
returns the type of the left operand), which denotes the number of bits to shift by.
Shifting by a signed type will produce a compilation error.

Members:

* ``.length`` yields the fixed length of the byte array (read-only).

.. note::
    The type ``bytes1[]`` is an array of bytes, but due to padding rules, it wastes
    31 bytes of space for each element (except in storage). It is better to use the ``bytes``
    type instead.

.. note::
    Prior to version 0.8.0, ``byte`` used to be an alias for ``bytes1``.

Dynamically-sized byte array
----------------------------

``bytes``:
    Dynamically-sized byte array, see :ref:`arrays`. Not a value-type!
``string``:
    Dynamically-sized UTF-8-encoded string, see :ref:`arrays`. Not a value-type!

.. index:: address, literal;address

.. _address_literals:

Address Literals
----------------

Hexadecimal literals that pass the address checksum test, for example
``0xdCad3a6d3569DF655070DEd06cb7A1b2Ccd1D3AF`` are of ``address`` type.
Hexadecimal literals that are between 39 and 41 digits
long and do not pass the checksum test produce
an error. You can prepend (for integer types) or append (for bytesNN types) zeros to remove the error.

.. note::
    The mixed-case address checksum format is defined in `EIP-55 <https://github.com/ethereum/EIPs/blob/master/EIPS/eip-55.md>`_.

.. index:: literal, literal;rational

.. _rational_literals:

Rational and Integer Literals
-----------------------------

Integer literals are formed from a sequence of digits in the range 0-9.
They are interpreted as decimals. For example, ``69`` means sixty nine.
Octal literals do not exist in Solidity and leading zeros are invalid.

Decimal fractional literals are formed by a ``.`` with at least one number on
one side.  Examples include ``1.``, ``.1`` and ``1.3``.

Scientific notation in the form of ``2e10`` is also supported, where the
mantissa can be fractional but the exponent has to be an integer.
The literal ``MeE`` is equivalent to ``M * 10**E``.
Examples include ``2e10``, ``-2e10``, ``2e-10``, ``2.5e1``.

Underscores can be used to separate the digits of a numeric literal to aid readability.
For example, decimal ``123_000``, hexadecimal ``0x2eff_abde``, scientific decimal notation ``1_2e345_678`` are all valid.
Underscores are only allowed between two digits and only one consecutive underscore is allowed.
There is no additional semantic meaning added to a number literal containing underscores,
the underscores are ignored.

Number literal expressions retain arbitrary precision until they are converted to a non-literal type (i.e. by
using them together with anything other than a number literal expression (like boolean literals) or by explicit conversion).
This means that computations do not overflow and divisions do not truncate
in number literal expressions.

For example, ``(2**800 + 1) - 2**800`` results in the constant ``1`` (of type ``uint8``)
although intermediate results would not even fit the machine word size. Furthermore, ``.5 * 8`` results
in the integer ``4`` (although non-integers were used in between).

.. warning::
    While most operators produce a literal expression when applied to literals, there are certain operators that do not follow this pattern:

    - Ternary operator (``... ? ... : ...``),
    - Array subscript (``<array>[<index>]``).

    You might expect expressions like ``255 + (true ? 1 : 0)`` or ``255 + [1, 2, 3][0]`` to be equivalent to using the literal 256
    directly, but in fact they are computed within the type ``uint8`` and can overflow.

Any operator that can be applied to integers can also be applied to number literal expressions as
long as the operands are integers. If any of the two is fractional, bit operations are disallowed
and exponentiation is disallowed if the exponent is fractional (because that might result in
a non-rational number).

Shifts and exponentiation with literal numbers as left (or base) operand and integer types
as the right (exponent) operand are always performed
in the ``uint256`` (for non-negative literals) or ``int256`` (for a negative literals) type,
regardless of the type of the right (exponent) operand.

.. warning::
    Division on integer literals used to truncate in Solidity prior to version 0.4.0, but it now converts into a rational number, i.e. ``5 / 2`` is not equal to ``2``, but to ``2.5``.

.. note::
    Solidity has a number literal type for each rational number.
    Integer literals and rational number literals belong to number literal types.
    Moreover, all number literal expressions (i.e. the expressions that
    contain only number literals and operators) belong to number literal
    types.  So the number literal expressions ``1 + 2`` and ``2 + 1`` both
    belong to the same number literal type for the rational number three.


.. note::
    Number literal expressions are converted into a non-literal type as soon as they are used with non-literal
    expressions. Disregarding types, the value of the expression assigned to ``b``
    below evaluates to an integer. Because ``a`` is of type ``uint128``, the
    expression ``2.5 + a`` has to have a proper type, though. Since there is no common type
    for the type of ``2.5`` and ``uint128``, the Solidity compiler does not accept
    this code.

.. code-block:: solidity

    uint128 a = 1;
    uint128 b = 2.5 + a + 0.5;

.. index:: literal, literal;string, string
.. _string_literals:

String Literals and Types
-------------------------

String literals are written with either double or single-quotes (``"foo"`` or ``'bar'``), and they can also be split into multiple consecutive parts (``"foo" "bar"`` is equivalent to ``"foobar"``) which can be helpful when dealing with long strings.  They do not imply trailing zeroes as in C; ``"foo"`` represents three bytes, not four.  As with integer literals, their type can vary, but they are implicitly convertible to ``bytes1``, ..., ``bytes32``, if they fit, to ``bytes`` and to ``string``.

For example, with ``bytes32 samevar = "stringliteral"`` the string literal is interpreted in its raw byte form when assigned to a ``bytes32`` type.

String literals can only contain printable ASCII characters, which means the characters between and including 0x20 .. 0x7E.

Additionally, string literals also support the following escape characters:

- ``\<newline>`` (escapes an actual newline)
- ``\\`` (backslash)
- ``\'`` (single quote)
- ``\"`` (double quote)
- ``\n`` (newline)
- ``\r`` (carriage return)
- ``\t`` (tab)
- ``\xNN`` (hex escape, see below)
- ``\uNNNN`` (unicode escape, see below)

``\xNN`` takes a hex value and inserts the appropriate byte, while ``\uNNNN`` takes a Unicode codepoint and inserts an UTF-8 sequence.

.. note::

    Until version 0.8.0 there were three additional escape sequences: ``\b``, ``\f`` and ``\v``.
    They are commonly available in other languages but rarely needed in practice.
    If you do need them, they can still be inserted via hexadecimal escapes, i.e. ``\x08``, ``\x0c``
    and ``\x0b``, respectively, just as any other ASCII character.

The string in the following example has a length of ten bytes.
It starts with a newline byte, followed by a double quote, a single
quote a backslash character and then (without separator) the
character sequence ``abcdef``.

.. code-block:: solidity
    :force:

    "\n\"\'\\abc\
    def"

Any Unicode line terminator which is not a newline (i.e. LF, VF, FF, CR, NEL, LS, PS) is considered to
terminate the string literal. Newline only terminates the string literal if it is not preceded by a ``\``.

Unicode Literals
----------------

While regular string literals can only contain ASCII, Unicode literals – prefixed with the keyword ``unicode`` – can contain any valid UTF-8 sequence.
They also support the very same escape sequences as regular string literals.

.. code-block:: solidity

    string memory a = unicode"Hello 😃";

.. index:: literal, bytes

Hexadecimal Literals
--------------------

Hexadecimal literals are prefixed with the keyword ``hex`` and are enclosed in double
or single-quotes (``hex"001122FF"``, ``hex'0011_22_FF'``). Their content must be
hexadecimal digits which can optionally use a single underscore as separator between
byte boundaries. The value of the literal will be the binary representation
of the hexadecimal sequence.

Multiple hexadecimal literals separated by whitespace are concatenated into a single literal:
``hex"00112233" hex"44556677"`` is equivalent to ``hex"0011223344556677"``

Hexadecimal literals behave like :ref:`string literals <string_literals>` and have the same convertibility restrictions.

.. index:: enum

.. _enums:

Enums
-----

Enums are one way to create a user-defined type in Solidity. They are explicitly convertible
to and from all integer types but implicit conversion is not allowed.  The explicit conversion
from integer checks at runtime that the value lies inside the range of the enum and causes a
:ref:`Panic error<assert-and-require>` otherwise.
Enums require at least one member, and its default value when declared is the first member.
Enums cannot have more than 256 members.

The data representation is the same as for enums in C: The options are represented by
subsequent unsigned integer values starting from ``0``.

Using ``type(NameOfEnum).min`` and ``type(NameOfEnum).max`` you can get the
smallest and respectively largest value of the given enum.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.8;

    contract test {
        enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
        ActionChoices choice;
        ActionChoices constant defaultChoice = ActionChoices.GoStraight;

        function setGoStraight() public {
            choice = ActionChoices.GoStraight;
        }

        // Since enum types are not part of the ABI, the signature of "getChoice"
        // will automatically be changed to "getChoice() returns (uint8)"
        // for all matters external to Solidity.
        function getChoice() public view returns (ActionChoices) {
            return choice;
        }

        function getDefaultChoice() public pure returns (uint) {
            return uint(defaultChoice);
        }

        function getLargestValue() public pure returns (ActionChoices) {
            return type(ActionChoices).max;
        }

        function getSmallestValue() public pure returns (ActionChoices) {
            return type(ActionChoices).min;
        }
    }

.. note::
    Enums can also be declared on the file level, outside of contract or library definitions.

.. index:: ! user defined value type, custom type

.. _user-defined-value-types:

User Defined Value Types
------------------------

A user defined value type allows creating a zero cost abstraction over an elementary value type.
This is similar to an alias, but with stricter type requirements.

A user defined value type is defined using ``type C is V``, where ``C`` is the name of the newly
introduced type and ``V`` has to be a built-in value type (the "underlying type"). The function
``C.wrap`` is used to convert from the underlying type to the custom type. Similarly, the
function ``C.unwrap`` is used to convert from the custom type to the underlying type.

The type ``C`` does not have any operators or bound member functions. In particular, even the
operator ``==`` is not defined. Explicit and implicit conversions to and from other types are
disallowed.

The data-representation of values of such types are inherited from the underlying type
and the underlying type is also used in the ABI.

The following example illustrates a custom type ``UFixed256x18`` representing a decimal fixed point
type with 18 decimals and a minimal library to do arithmetic operations on the type.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity ^0.8.8;

    // Represent a 18 decimal, 256 bit wide fixed point type using a user defined value type.
    type UFixed256x18 is uint256;

    /// A minimal library to do fixed point operations on UFixed256x18.
    library FixedMath {
        uint constant multiplier = 10**18;

        /// Adds two UFixed256x18 numbers. Reverts on overflow, relying on checked
        /// arithmetic on uint256.
        function add(UFixed256x18 a, UFixed256x18 b) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(UFixed256x18.unwrap(a) + UFixed256x18.unwrap(b));
        }
        /// Multiplies UFixed256x18 and uint256. Reverts on overflow, relying on checked
        /// arithmetic on uint256.
        function mul(UFixed256x18 a, uint256 b) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(UFixed256x18.unwrap(a) * b);
        }
        /// Take the floor of a UFixed256x18 number.
        /// @return the largest integer that does not exceed `a`.
        function floor(UFixed256x18 a) internal pure returns (uint256) {
            return UFixed256x18.unwrap(a) / multiplier;
        }
        /// Turns a uint256 into a UFixed256x18 of the same value.
        /// Reverts if the integer is too large.
        function toUFixed256x18(uint256 a) internal pure returns (UFixed256x18) {
            return UFixed256x18.wrap(a * multiplier);
        }
    }

Notice how ``UFixed256x18.wrap`` and ``FixedMath.toUFixed256x18`` have the same signature but
perform two very different operations: The ``UFixed256x18.wrap`` function returns a ``UFixed256x18``
that has the same data representation as the input, whereas ``toUFixed256x18`` returns a
``UFixed256x18`` that has the same numerical value.

.. index:: ! function type, ! type; function

.. _function_types:

Function Types
--------------

Function types are the types of functions. Variables of function type
can be assigned from functions and function parameters of function type
can be used to pass functions to and return functions from function calls.
Function types come in two flavours - *internal* and *external* functions:

Internal functions can only be called inside the current contract (more specifically,
inside the current code unit, which also includes internal library functions
and inherited functions) because they cannot be executed outside of the
context of the current contract. Calling an internal function is realized
by jumping to its entry label, just like when calling a function of the current
contract internally.

External functions consist of an address and a function signature and they can
be passed via and returned from external function calls.

Function types are notated as follows:

.. code-block:: solidity
    :force:

    function (<parameter types>) {internal|external} [pure|view|payable] [returns (<return types>)]

In contrast to the parameter types, the return types cannot be empty - if the
function type should not return anything, the whole ``returns (<return types>)``
part has to be omitted.

By default, function types are internal, so the ``internal`` keyword can be
omitted. Note that this only applies to function types. Visibility has
to be specified explicitly for functions defined in contracts, they
do not have a default.

Conversions:

A function type ``A`` is implicitly convertible to a function type ``B`` if and only if
their parameter types are identical, their return types are identical,
their internal/external property is identical and the state mutability of ``A``
is more restrictive than the state mutability of ``B``. In particular:

- ``pure`` functions can be converted to ``view`` and ``non-payable`` functions
- ``view`` functions can be converted to ``non-payable`` functions
- ``payable`` functions can be converted to ``non-payable`` functions

No other conversions between function types are possible.

The rule about ``payable`` and ``non-payable`` might be a little
confusing, but in essence, if a function is ``payable``, this means that it
also accepts a payment of zero Ether, so it also is ``non-payable``.
On the other hand, a ``non-payable`` function will reject Ether sent to it,
so ``non-payable`` functions cannot be converted to ``payable`` functions.

If a function type variable is not initialised, calling it results
in a :ref:`Panic error<assert-and-require>`. The same happens if you call a function after using ``delete``
on it.

If external function types are used outside of the context of Solidity,
they are treated as the ``function`` type, which encodes the address
followed by the function identifier together in a single ``bytes24`` type.

Note that public functions of the current contract can be used both as an
internal and as an external function. To use ``f`` as an internal function,
just use ``f``, if you want to use its external form, use ``this.f``.

A function of an internal type can be assigned to a variable of an internal function type regardless
of where it is defined.
This includes private, internal and public functions of both contracts and libraries as well as free
functions.
External function types, on the other hand, are only compatible with public and external contract
functions.
Libraries are excluded because they require a ``delegatecall`` and use :ref:`a different ABI
convention for their selectors <library-selectors>`.
Functions declared in interfaces do not have definitions so pointing at them does not make sense either.

Members:

External (or public) functions have the following members:

* ``.address`` returns the address of the contract of the function.
* ``.selector`` returns the :ref:`ABI function selector <abi_function_selector>`

.. note::
  External (or public) functions used to have the additional members
  ``.gas(uint)`` and ``.value(uint)``. These were deprecated in Solidity 0.6.2
  and removed in Solidity 0.7.0. Instead use ``{gas: ...}`` and ``{value: ...}``
  to specify the amount of gas or the amount of wei sent to a function,
  respectively. See :ref:`External Function Calls <external-function-calls>` for
  more information.

Example that shows how to use the members:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.6.4 <0.9.0;

    contract Example {
        function f() public payable returns (bytes4) {
            assert(this.f.address == address(this));
            return this.f.selector;
        }

        function g() public {
            this.f{gas: 10, value: 800}();
        }
    }

Example that shows how to use internal function types:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.16 <0.9.0;

    library ArrayUtils {
        // internal functions can be used in internal library functions because
        // they will be part of the same code context
        function map(uint[] memory self, function (uint) pure returns (uint) f)
            internal
            pure
            returns (uint[] memory r)
        {
            r = new uint[](self.length);
            for (uint i = 0; i < self.length; i++) {
                r[i] = f(self[i]);
            }
        }

        function reduce(
            uint[] memory self,
            function (uint, uint) pure returns (uint) f
        )
            internal
            pure
            returns (uint r)
        {
            r = self[0];
            for (uint i = 1; i < self.length; i++) {
                r = f(r, self[i]);
            }
        }

        function range(uint length) internal pure returns (uint[] memory r) {
            r = new uint[](length);
            for (uint i = 0; i < r.length; i++) {
                r[i] = i;
            }
        }
    }


    contract Pyramid {
        using ArrayUtils for *;

        function pyramid(uint l) public pure returns (uint) {
            return ArrayUtils.range(l).map(square).reduce(sum);
        }

        function square(uint x) internal pure returns (uint) {
            return x * x;
        }

        function sum(uint x, uint y) internal pure returns (uint) {
            return x + y;
        }
    }

Another example that uses external function types:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.22 <0.9.0;


    contract Oracle {
        struct Request {
            bytes data;
            function(uint) external callback;
        }

        Request[] private requests;
        event NewRequest(uint);

        function query(bytes memory data, function(uint) external callback) public {
            requests.push(Request(data, callback));
            emit NewRequest(requests.length - 1);
        }

        function reply(uint requestID, uint response) public {
            // Here goes the check that the reply comes from a trusted source
            requests[requestID].callback(response);
        }
    }


    contract OracleUser {
        Oracle constant private ORACLE_CONST = Oracle(address(0x00000000219ab540356cBB839Cbe05303d7705Fa)); // known contract
        uint private exchangeRate;

        function buySomething() public {
            ORACLE_CONST.query("USD", this.oracleResponse);
        }

        function oracleResponse(uint response) public {
            require(
                msg.sender == address(ORACLE_CONST),
                "Only oracle can call this."
            );
            exchangeRate = response;
        }
    }

.. note::
    Lambda or inline functions are planned but not yet supported.
>>>>>>> 8d51eb6ea9505ea3dc45ffcf17a3b35c278da2cb
