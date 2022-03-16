<<<<<<< HEAD
.. _yul:

###
Yul
###

.. index:: ! assembly, ! asm, ! evmasm, ! yul, julia, iulia

Yul (sebelumnaya juga dipanggil JULIA atau IULIA) adalah bahasa perantara yang dapat
dikompilasi ke bytecode untuk backend yang berbeda.

Dukungan untuk EVM 1.0, EVM 1.5 dan Ewasm direncanakan, dan dirancang untuk menjadi
penyebut umum yang dapat digunakan dari ketiga platform. Itu sudah dapat digunakan
dalam mode mandiri dan untuk "inline assembly" di dalam Solidity dan ada implementasi
eksperimental dari kompiler Solidity yang menggunakan Yul sebagai bahasa perantara.
Yul adalah target yang baik untuk tahap pengoptimalan tingkat tinggi yang dapat menguntungkan
semua platform target secara merata.

Motivation dan Deskiripsi High-level
=====================================

Desain Yul mencoba mencapai beberapa tujuan:

1. Program yang ditulis dalam Yul harus dapat dibaca, meskipun kode dibuat oleh kompiler dari Solidity atau bahasa tingkat tinggi lainnya.
2. Aliran kontrol harus mudah dipahami untuk membantu inspeksi manual, verifikasi formal, dan pengoptimalan.
3. Terjemahan dari Yul ke bytecode harus sesederhana mungkin.
4. Yul harus cocok untuk pengoptimalan seluruh program.

Untuk mencapai tujuan pertama dan kedua, Yul menyediakan konstruksi tingkat tinggi
seperti ``for`` loop, ``if`` dan ``switch`` pernyataan dan panggilan fungsi. Ini harus
cukup untuk mewakili secara memadai aliran kontrol untuk program perakitan.
Oleh karena itu, tidak ada pernyataan eksplisit untuk ``SWAP``, ``DUP``, ``JUMPDEST``, ``JUMP`` dan ``JUMPI``
disediakan, karena dua yang pertama mengaburkan aliran data
dan dua yang terakhir mengaburkan aliran kontrol. Selanjutnya, pernyataan fungsional dari
bentuk ``mul(add(x, y), 7)`` lebih disukai daripada pernyataan opcode murni seperti
``7 y x add mul`` karena dalam bentuk pertama, lebih mudah untuk melihat operand
mana digunakan untuk opcode yang mana.

Meskipun dirancang untuk mesin stack, Yul tidak mengekspos kompleksitas dari stack itu sendiri.
Pemrogram atau auditor tidak perlu khawatir tentang stack.

Tujuan ketiga dicapai dengan menyusun
konstruksi tingkat yang lebih tinggi ke bytecode dengan cara yang sangat teratur.
Satu-satunya operasi non-lokal yang dilakukan
oleh assembler adalah pencarian nama pengidentifikasi yang ditentukan pengguna (fungsi, variabel, ...)
dan pembersihan variabel lokal dari stack.

Untuk menghindari kebingungan antara konsep seperti nilai dan referensi,
Yul diketik secara statis. Pada saat yang sama, ada tipe default
(biasanya kata integer dari mesin target) yang selalu bisa
dihilangkan untuk membantu keterbacaan.

Untuk menjaga bahasa tetap sederhana dan fleksibel, Yul tidak memiliki
operasi, fungsi, atau tipe bawaan apapun dalam bentuk murninya.
Ini ditambahkan bersama dengan semantiknya saat menentukan dialek Yul,
yang memungkinkan mengkhususkan Yul dengan persyaratan target platfiorm
yang berbeda dan set fitur.

Saat ini, hanya ada satu dialek khusus Yul. Dialek ini menggunakan
opcode EVM sebagai fungsi bawaan
(lihat di bawah) dan hanya mendefinisikan tipe ``u256``, yang merupakan tipe
EVM 256-bit asli. Karena itu, kami tidak akan memberikan tipe pada contoh di bawah ini.


Contoh Sederhana
================

Contoh program berikut ditulis dalam dialek EVM dan menghitung eksponensial.
Itu dapat dikompilasi menggunakan ``solc --strict-assembly``. Fungsi bawaan
``mul`` dan ``div`` menghitung produk dan pembagian, secara masing-masing.

.. code-block:: yul

    {
        function power(base, exponent) -> result
        {
            switch exponent
            case 0 { result := 1 }
            case 1 { result := base }
            default
            {
                result := power(mul(base, base), div(exponent, 2))
                switch mod(exponent, 2)
                    case 1 { result := mul(base, result) }
            }
        }
    }

Dimungkinkan juga untuk mengimplementasikan fungsi yang sama menggunakan for-loop
bukannya dengan rekursi. Di sini, ``lt(a, b)`` menghitung apakah ``a`` lebih kecil dari ``b``.
perbandingan kurang-dari.

.. code-block:: yul

    {
        function power(base, exponent) -> result
        {
            result := 1
            for { let i := 0 } lt(i, exponent) { i := add(i, 1) }
            {
                result := mul(result, base)
            }
        }
    }

Di :ref:`akhir bagian <erc20yul>`, implementasi lengkap dari
standar ERC-20 dapat ditemukan.



Penggunaan Stand-Alone
======================

Anda dapat menggunakan Yul dalam bentuk stand-alone dalam dialek EVM menggunakan compiler Solidity.
Ini akan menggunakan :ref:`objek notasi Yul <yul-object>` sehingga memungkinkan untuk merujuk
kode sebagai data untuk menyebarkan kontrak. Mode Yul ini tersedia untuk kompiler baris perintah
(gunakan ``--strict-assembly``) dan untuk :ref:`standard-json interface <compiler-api>`:

.. code-block:: json

    {
        "language": "Yul",
        "sources": { "input.yul": { "content": "{ sstore(0, 1) }" } },
        "settings": {
            "outputSelection": { "*": { "*": ["*"], "": [ "*" ] } },
            "optimizer": { "enabled": true, "details": { "yul": true } }
        }
    }

.. warning::

    Yul sedang dalam pengembangan aktif dan pembuatan bytecode hanya diimplementasikan sepenuhnya untuk dialek EVM Yul
    dengan EVM 1.0 sebagai target.


Deskripsi Informal Yul
===========================

Berikut ini, kita akan berbicara tentang setiap aspek individu
dari bahasa Yul. Dalam contoh, kami akan menggunakan dialek EVM default.

Syntax
------

Yul mem-parsing komentar, literal, dan pengidentifikasi dengan cara yang sama seperti Solidity,
jadi Anda bisa misalnya menggunakan ``//`` dan ``/* */`` untuk menunjukkan komentar.
Ada satu pengecualian: Pengidentifikasi di Yul dapat berisi titik: ``.``.

Yul dapat menentukan "objek" yang terdiri dari kode, data, dan sub-objek.
Silakan lihat :ref:`Yul Objects <yul-object>` di bawah untuk detailnya.
Di bagian ini, kita hanya membahas bagian kode dari objek semacam itu.
Bagian kode ini selalu terdiri dari curly-braces
blok yang dibatasi. Sebagian besar alat mendukung menentukan hanya blok kode
dimana suatu objek diharapkan.

Di dalam blok kode, elemen berikut dapat digunakan:
(lihat bagian selanjutnya untuk lebih jelasnya):

- literals, misalnya ``0x123``, ``42`` atau ``"abc"`` (string hingga 32 karakter)
- panggilan ke fungsi bawaan, mis. ``add(1, mload(0))``
- deklarasi variabel, mis. ``let x := 7``, ``let x := add(y, 3)`` atau ``let x`` (nilai awal 0 ditetapkan)
- identifiers (variables), mis. ``add(3, x)``
- assignments, mis. ``x := add(y, 3)``
- blok di mana variabel lokal dicakup di dalamnya, mis. ``{ let x := 3 { let y := add(x, 1) } }``
- if statements, mis. ``if lt(a, b) { sstore(0, 1) }``
- switch statements, mis. ``switch mload(0) case 0 { revert() } default { mstore(0, 1) }``
- for loops, mis. ``for { let i := 0} lt(i, 10) { i := add(i, 1) } { mstore(i, 7) }``
- function definitions, mis. ``function f(a, b) -> c { c := add(a, b) }```

Beberapa elemen sintaksis dapat mengikuti satu sama lain hanya dipisahkan oleh
whitespace, yaitu tidak diperlukan penghentian ``;`` atau baris baru.

Literals
--------

Sebagai literal, Anda dapat menggunakan:

- Konstanta integer dalam notasi desimal atau heksadesimal.

- String ASCII (mis. ``"abc"``), yang mungkin berisi escape hex ``\xNN`` dan escape Unicode ``\uNNNN`` di mana ``N`` adalah digit heksadesimal.

- String hex (mis. ``hex"616263"``).

Dalam dialek EVM Yul, literal mewakili 256-bit kata sebagai berikut:

- Konstanta desimal atau heksadesimal harus kurang dari ``2**256``.
  Mereka mewakili kata 256-bit dengan nilai itu sebagai unsigned integer dalam pengkodean big endian.

- String ASCII pertama kali dilihat sebagai urutan byte, dengan melihat
  karakter ASCII non-escape sebagai byte tunggal yang nilainya adalah kode ASCII,
  escape ``\xNN`` sebagai byte tunggal dengan nilai tersebut, dan
  escape ``\uNNNN`` sebagai urutan byte UTF-8 untuk titik kode tersebut.
  Urutan byte tidak boleh melebihi 32 byte.
  Urutan byte diisi dengan nol di sebelah kanan untuk mencapai panjang 32 byte;
  dengan kata lain, string disimpan rata kiri.
  Urutan byte yang diisi mewakili kata 256-bit yang 8 bit paling signifikannya adalah yang berasal dari byte pertama,
  yaitu byte ditafsirkan dalam bentuk big endian.

- String hex pertama kali dilihat sebagai urutan byte, dengan melihat
  setiap pasangan digit hex yang berdekatan sebagai byte.
  Urutan byte tidak boleh melebihi 32 byte (yaitu 64 digit hex), dan diperlakukan seperti di atas.

Saat mengkompilasi untuk EVM, ini akan diterjemahkan ke dalam
instruksi ``PUSHI`` yang sesuai. Dalam contoh berikut,
``3`` dan ``2`` ditambahkan sehingga menghasilkan 5 dan kemudian
bitwise ``and`` dengan string "abc" dihitung.
Nilai akhir ditetapkan ke variabel lokal yang disebut ``x``.

Batas 32 byte di atas tidak berlaku untuk literal string yang diteruskan ke fungsi bawaan yang memerlukan
argumen literal (mis. ``setimmutable`` atau ``loadimmutable``). String tersebut tidak pernah berakhir di
bytecode yang dihasilkan.

.. code-block:: yul

    let x := and("abc", add(3, 2))

Kecuali itu adalah tipe default, tipe literal
harus ditentukan setelah titik dua:

.. code-block:: yul

    // This will not compile (u32 and u256 type not implemented yet)
    let x := and("abc":u32, add(3:u256, 2:u256))


Function Calls
--------------

Fungsi bawaan dan fungsi yang ditentukan pengguna (lihat di bawah) dapat dipanggil
dengan cara yang sama seperti yang ditunjukkan pada contoh sebelumnya.
Jika fungsi menghasilkan nilai tunggal, itu dapat langsung digunakan
dalam ekspresi lagi. Jika menghasilkan beberapa nilai,
mereka harus ditugaskan ke variabel lokal.

.. code-block:: yul

    function f(x, y) -> a, b { /* ... */ }
    mstore(0x80, add(mload(0x80), 3))
    // Here, the user-defined function `f` returns two values.
    let x, y := f(1, mload(0))

Untuk fungsi bawaan EVM, ekspresi fungsional
dapat langsung diterjemahkan ke aliran opcode:
Anda hanya membaca ekspresi dari kanan ke kiri untuk mendapatkan
opcode. Dalam kasus baris pertama dalam contoh, ini
adalah ``PUSH1 3 PUSH1 0x80 MLOAD ADD PUSH1 0x80 MSTORE``.

Untuk panggilan ke fungsi yang ditentukan pengguna, argumennya juga
ditaruh di stack dari kanan ke kiri dan ini urutannya
di mana daftar argumen dievaluasi. Nilai return,
meskipun, diharapkan pada stack dari kiri ke kanan,
yaitu dalam contoh ini, ``y`` berada di atas tumpukan dan ``x``
berada di bawahnya.

Deklarasi Variabel
---------------------

Anda dapat menggunakan kata kunci ``let`` untuk mendeklarasikan variabel.
Sebuah variabel hanya terlihat di dalam
``{...}``-blok itu didefinisikan. Saat mengkompilasi ke EVM,
slot stack baru dibuat yang dicadangkan
untuk variabel dan secara otomatis dihapus lagi ketika akhir blok
tercapai. Anda dapat memberikan nilai awal untuk variabel.
Jika Anda tidak memberikan nilai, variabel akan diinisialisasi ke nol.

Karena variabel disimpan di stack, mereka tidak secara langsung
mempengaruhi memori atau penyimpanan, tetapi mereka dapat digunakan sebagai petunjuk
ke memori atau lokasi penyimpanan dalam fungsi bawaan
``mstore``, ``mload``, ``sstore`` dan ``sload``.
Dialek masa depan mungkin memperkenalkan tipe khusus untuk petunjuk tersebut.

Ketika sebuah variabel direferensikan, nilainya saat ini disalin.
Untuk EVM, ini diterjemahkan menjadi instruksi ``DUP``.

.. code-block:: yul

    {
        let zero := 0
        let v := calldataload(zero)
        {
            let y := add(sload(v), 1)
            v := y
        } // y is "deallocated" here
        sstore(v, zero)
    } // v and zero are "deallocated" here


Jika variabel yang dideklarasikan harus memiliki tipe yang berbeda dari tipe default,
Anda menunjukkan setelah tanda titik dua. Anda juga dapat mendeklarasikan beberapa
variabel dalam satu pernyataan saat Anda menetapkan dari panggilan fungsi
yang menghasilkan banyak nilai.

.. code-block:: yul

    // This will not compile (u32 and u256 type not implemented yet)
    {
        let zero:u32 := 0:u32
        let v:u256, t:u32 := f()
        let x, y := g()
    }

Bergantung pada pengaturan pengoptimal, kompiler dapat mengosongkan
slot stack setelah variabel digunakan untuk
terakhir kali, meskipun masih dalam ruang lingkup.


Assignments
-----------

Variabel dapat ditetapkan setelah definisinya menggunakan
``: =`` operator. Dimungkinkan untuk menetapkan beberapa
variabel sekaligus. Untuk itu, jumlah dan jenis
nilai harus cocok.
Jika Anda ingin menetapkan nilai yang dikembalikan dari fungsi yang memiliki
beberapa parameter pengembalian, Anda harus menyediakan banyak variabel.
Variabel yang sama tidak boleh muncul beberapa kali di sisi kiri
tugas, mis. ``x, x := f()`` tidak valid.

.. code-block:: yul

    let v := 0
    // re-assign v
    v := 2
    let t := add(v, 2)
    function f() -> a, b { }
    // assign multiple values
    v, t := f()


If
--

Pernyataan if dapat digunakan untuk mengeksekusi kode secara kondisional.
Tidak ada blok "lain" yang dapat ditentukan. Pertimbangkan untuk menggunakan "switch" sebagai gantinya (lihat di bawah) jika
Anda membutuhkan beberapa alternatif.

.. code-block:: yul

    if lt(calldatasize(), 4) { revert(0, 0) }

Kurung kurawal untuk tubuh diperlukan.

Switch
------

Anda dapat menggunakan pernyataan switch sebagai versi lanjutan dari pernyataan if.
Dibutuhkan nilai ekspresi dan membandingkannya dengan beberapa konstanta literal.
Cabang yang sesuai dengan konstanta pencocokan diambil.
Bertentangan dengan bahasa pemrograman lain, untuk alasan keamanan, aliran kontrol tidak
melanjutkan dari satu kasus ke kasus berikutnya. Mungkin ada fallback atau default
case yang disebut ``default`` yang diambil jika tidak ada konstanta literal yang cocok.

.. code-block:: yul

    {
        let x := 0
        switch calldataload(4)
        case 0 {
            x := calldataload(0x24)
        }
        default {
            x := calldataload(0x44)
        }
        sstore(0, div(x, 2))
    }

Daftar kasus tidak diapit oleh kurung kurawal, tetapi badan
kasing membutuhkannya.

Loops
-----

Yul mendukung for-loop yang terdiri dari
header yang berisi bagian inisialisasi, kondisi, pasca-iterasi
bagian dan tubuh. Kondisinya harus berupa ekspresi, sementara
tiga lainnya adalah balok. Jika bagian inisialisasi
mendeklarasikan variabel apa pun di tingkat atas, ruang lingkup variabel ini meluas ke semua variabel lainnya
bagian dari lingkaran.

Pernyataan ``break`` dan ``continue`` dapat digunakan dalam body untuk keluar dari loop
atau lompat ke bagian post, secara masing-masing.

Contoh berikut menghitung jumlah area dalam memori.

.. code-block:: yul

    {
        let x := 0
        for { let i := 0 } lt(i, 0x100) { i := add(i, 0x20) } {
            x := add(x, mload(i))
        }
    }

For loop juga dapat digunakan sebagai pengganti while loop:
Cukup biarkan bagian inisialisasi dan pasca-iterasi kosong.

.. code-block:: yul

    {
        let x := 0
        let i := 0
        for { } lt(i, 0x100) { } {     // while(i < 0x100)
            x := add(x, mload(i))
            i := add(i, 0x20)
        }
    }

Function Declarations
---------------------

Yul memungkinkan definisi fungsi. Ini tidak boleh dikacaukan dengan fungsi
dalam Solidity karena mereka tidak pernah menjadi bagian dari antarmuka eksternal kontrak dan
adalah bagian dari namespace yang terpisah dari yang untuk fungsi Solidity.

Untuk EVM, fungsi Yul mengambil
argumen (dan PC kembali) dari stack dan juga menempatkan hasilnya ke
stack. Fungsi yang ditentukan pengguna dan fungsi bawaan dipanggil dengan cara yang sama persis.

Fungsi dapat didefinisikan di mana saja dan terlihat di bloknya
dideklarasikan. Di dalam suatu fungsi, Anda tidak dapat mengakses variabel lokal
didefinisikan di luar fungsi itu.

Fungsi mendeklarasikan parameter dan mengembalikan variabel, mirip dengan Solidity.
Untuk mengembalikan nilai, Anda menetapkannya ke variabel return.

Jika Anda memanggil fungsi yang mengembalikan banyak nilai, Anda harus menetapkan
mereka ke beberapa variabel menggunakan ``a, b := f(x)`` atau ``biarkan a, b := f(x)``.

Pernyataan ``leave`` dapat digunakan untuk keluar dari fungsi saat ini. Ini
berfungsi seperti pernyataan ``return`` dalam bahasa lain hanya saja
tidak mengambil nilai untuk dikembalikan, itu hanya keluar dari fungsi dan fungsi
akan mengembalikan nilai apa pun yang saat ini ditetapkan ke variabel return.

Perhatikan bahwa dialek EVM memiliki fungsi bawaan yang disebut ``return`` yang
keluar dari konteks eksekusi penuh (panggilan pesan internal) dan bukan hanya
fungsi yul saat ini.

Contoh berikut mengimplementasikan fungsi daya dengan kuadrat-dan-kalikan.

.. code-block:: yul

    {
        function power(base, exponent) -> result {
            switch exponent
            case 0 { result := 1 }
            case 1 { result := base }
            default {
                result := power(mul(base, base), div(exponent, 2))
                switch mod(exponent, 2)
                    case 1 { result := mul(base, result) }
            }
        }
    }

Spesifikasi Yul
====================

Bab ini menjelaskan kode Yul secara formal. Kode Yul biasanya ditempatkan di dalam objek Yul,
yang dijelaskan dalam bab mereka sendiri.

.. code-block:: none

    Block = '{' Statement* '}'
    Statement =
        Block |
        FunctionDefinition |
        VariableDeclaration |
        Assignment |
        If |
        Expression |
        Switch |
        ForLoop |
        BreakContinue |
        Leave
    FunctionDefinition =
        'function' Identifier '(' TypedIdentifierList? ')'
        ( '->' TypedIdentifierList )? Block
    VariableDeclaration =
        'let' TypedIdentifierList ( ':=' Expression )?
    Assignment =
        IdentifierList ':=' Expression
    Expression =
        FunctionCall | Identifier | Literal
    If =
        'if' Expression Block
    Switch =
        'switch' Expression ( Case+ Default? | Default )
    Case =
        'case' Literal Block
    Default =
        'default' Block
    ForLoop =
        'for' Block Expression Block Block
    BreakContinue =
        'break' | 'continue'
    Leave = 'leave'
    FunctionCall =
        Identifier '(' ( Expression ( ',' Expression )* )? ')'
    Identifier = [a-zA-Z_$] [a-zA-Z_$0-9.]*
    IdentifierList = Identifier ( ',' Identifier)*
    TypeName = Identifier
    TypedIdentifierList = Identifier ( ':' TypeName )? ( ',' Identifier ( ':' TypeName )? )*
    Literal =
        (NumberLiteral | StringLiteral | TrueLiteral | FalseLiteral) ( ':' TypeName )?
    NumberLiteral = HexNumber | DecimalNumber
    StringLiteral = '"' ([^"\r\n\\] | '\\' .)* '"'
    TrueLiteral = 'true'
    FalseLiteral = 'false'
    HexNumber = '0x' [0-9a-fA-F]+
    DecimalNumber = [0-9]+


Batasan pada Tata Bahasa
---------------------------

Terlepas dari yang secara langsung dipaksakan oleh tata bahasa, pembatasa
berikut berlaku :

Switch harus memiliki setidaknya satu kasing (termasuk kasing default).
Semua nilai kasing harus memiliki tipe yang sama dan nilai yang berbeda.
Jika semua nilai yang mungkin dari tipe ekspresi tercakup, kasing default
tidak diizinkan (yaitu switch dengan ekspresi ``bool`` yang keduanya memiliki kasing
benar dan  palsu tidak memungkinkan kasing default).

Setiap ekspresi mengevaluasi ke nilai nol atau lebih. Identifier dan Literal
mengevaluasi dengan tepat
satu nilai dan panggilan fungsi mengevaluasi ke sejumlah nilai yang sama dengan
jumlah variabel return dari fungsi yang dipanggil.

Dalam deklarasi dan penugasan variabel, ekspresi sisi kanan
(jika ada) harus mengevaluasi sejumlah nilai yang sama dengan jumlah
variabel di sisi kiri.
Ini adalah satu-satunya situasi di mana ekspresi mengevaluasi
untuk lebih dari satu nilai diperbolehkan.
Nama variabel yang sama tidak boleh muncul lebih dari satu kali di sisi kiri
deklarasi tugas atau variabel.

Ekspresi yang juga merupakan pernyataan (yaitu pada level blok) harus
dievaluasi ke nilai nol.

Dalam semua situasi lain, ekspresi harus mengevaluasi tepat satu nilai.

Pernyataan ``continue`` dan ``break`` hanya dapat digunakan di dalam badan loop
dan harus dalam fungsi yang sama dengan loop (atau keduanya harus berada di
level tertinggi). Pernyataan ``continue`` dan ``break`` tidak dapat digunakan
di bagian lain dari loop, bahkan ketika itu dicakup di dalam tubuh loop kedua.

Bagian kondisi dari for-loop harus dievaluasi ke tepat satu nilai.

Pernyataan ``leave`` hanya dapat digunakan di dalam suatu fungsi.

Fungsi tidak dapat didefinisikan di mana pun di dalam blok init loop.

Literal tidak boleh lebih besar dari tipenya. Jenis terbesar yang ditentukan adalah lebar 256-bit.

Selama assignment dan panggilan fungsi, jenis nilai masing-masing harus cocok.
Tidak ada konversi tipe implisit. Konversi tipe secara umum hanya dapat dicapai
jika dialek menyediakan fungsi bawaan yang sesuai yang mengambil nilai satu
jenis dan mengembalikan nilai dari jenis yang berbeda.

Aturan Scoping
--------------

Scope di Yul terikat ke Blok (pengecualian adalah fungsi dan for loop
seperti yang dijelaskan di bawah) dan semua deklarasi
(``FunctionDefinition``, ``VariableDeclaration``)
memperkenalkan pengidentifikasi baru ke dalam cakupan ini.

Pengidentifikasi terlihat di
blok tempat mereka didefinisikan (termasuk semua sub-node dan sub-blok):
Fungsi terlihat di seluruh blok (bahkan sebelum definisinya) sementara
variabel hanya terlihat mulai dari pernyataan setelah ``VariableDeclaration``.

Khususnya,
variabel tidak dapat direferensikan di sisi kanan variabel mereka sendiri
pernyataan.
Fungsi dapat direferensikan sebelum deklarasinya (jika terlihat).

Sebagai pengecualian untuk aturan pelingkupan umum, ruang lingkup bagian "init" dari for-loop
(blok pertama) meluas di semua bagian lain dari loop.
Ini berarti bahwa variabel (dan fungsi) dideklarasikan di bagian init (tetapi tidak di dalam sebuah
blok di dalam bagian init) terlihat di semua bagian lain dari for-loop.

Pengidentifikasi yang dideklarasikan di bagian lain dari loop for menghormati aturan
pelingkupan sintaksis reguler.

Ini berarti for-loop dari form ``for { I... } C { P... } { B... }`` setara
dengan ``{ I... for {} C { P... } { B... } }``.

Parameter dan return parameter fungsi terlihat di
fungsi tubuh dan nama mereka harus berbeda.

Di dalam fungsi, tidak mungkin untuk mereferensikan variabel yang dideklarasikan
di luar fungsi tersebut.

Membayangi tidak diizinkan, yaitu Anda tidak dapat mendeklarasikan pengidentifikasi pada suatu titik
di mana pengidentifikasi lain dengan nama yang sama juga terlihat, meskipun itu
tidak mungkin untuk merujuknya karena dideklarasikan di luar fungsi saat ini.

Spesifikasi Resmi
--------------------

Kami secara resmi menentukan Yul dengan memberikan fungsi evaluasi E overload
pada berbagai node AST. Karena fungsi bawaan dapat memiliki efek samping,
E mengambil dua objek state dan simpul AST dan mengembalikan dua state objek
yang baru dan sejumlah variabel nilai lainnya.
Dua objek state adalah objek status global
(yang dalam konteks EVM adalah memori, penyimpanan, dan status
blockchain) dan objek state lokal (keadaan variabel lokal, mis
segmen stack di EVM).

Jika node AST adalah pernyataan, E mengembalikan dua objek state dan "mode",
yang digunakan untuk pernyataan ``break``, ``continue`` dan ``leave``.
Jika node AST adalah ekspresi, E mengembalikan dua objek state dan
sebanyak nilai yang dievaluasi oleh ekspresi.


Sifat pasti dari state global tidak ditentukan untuk deskripsi
tingkat tinggi ini. State lokal ``L`` adalah pemetaan pengidentifikasi ``i`` ke nilai ``v``,
dilambangkan sebagai ``L[i] = v``.

Untuk pengenal ``v``, biarkan ``$v`` menjadi nama pengenal.

Kami akan menggunakan notasi destructuring untuk node AST.

.. code-block:: none

    E(G, L, <{St1, ..., Stn}>: Block) =
        let G1, L1, mode = E(G, L, St1, ..., Stn)
        let L2 be a restriction of L1 to the identifiers of L
        G1, L2, mode
    E(G, L, St1, ..., Stn: Statement) =
        if n is zero:
            G, L, regular
        else:
            let G1, L1, mode = E(G, L, St1)
            if mode is regular then
                E(G1, L1, St2, ..., Stn)
            otherwise
                G1, L1, mode
    E(G, L, FunctionDefinition) =
        G, L, regular
    E(G, L, <let var_1, ..., var_n := rhs>: VariableDeclaration) =
        E(G, L, <var_1, ..., var_n := rhs>: Assignment)
    E(G, L, <let var_1, ..., var_n>: VariableDeclaration) =
        let L1 be a copy of L where L1[$var_i] = 0 for i = 1, ..., n
        G, L1, regular
    E(G, L, <var_1, ..., var_n := rhs>: Assignment) =
        let G1, L1, v1, ..., vn = E(G, L, rhs)
        let L2 be a copy of L1 where L2[$var_i] = vi for i = 1, ..., n
        G, L2, regular
    E(G, L, <for { i1, ..., in } condition post body>: ForLoop) =
        if n >= 1:
            let G1, L, mode = E(G, L, i1, ..., in)
            // mode has to be regular or leave due to the syntactic restrictions
            if mode is leave then
                G1, L1 restricted to variables of L, leave
            otherwise
                let G2, L2, mode = E(G1, L1, for {} condition post body)
                G2, L2 restricted to variables of L, mode
        else:
            let G1, L1, v = E(G, L, condition)
            if v is false:
                G1, L1, regular
            else:
                let G2, L2, mode = E(G1, L, body)
                if mode is break:
                    G2, L2, regular
                otherwise if mode is leave:
                    G2, L2, leave
                else:
                    G3, L3, mode = E(G2, L2, post)
                    if mode is leave:
                        G2, L3, leave
                    otherwise
                        E(G3, L3, for {} condition post body)
    E(G, L, break: BreakContinue) =
        G, L, break
    E(G, L, continue: BreakContinue) =
        G, L, continue
    E(G, L, leave: Leave) =
        G, L, leave
    E(G, L, <if condition body>: If) =
        let G0, L0, v = E(G, L, condition)
        if v is true:
            E(G0, L0, body)
        else:
            G0, L0, regular
    E(G, L, <switch condition case l1:t1 st1 ... case ln:tn stn>: Switch) =
        E(G, L, switch condition case l1:t1 st1 ... case ln:tn stn default {})
    E(G, L, <switch condition case l1:t1 st1 ... case ln:tn stn default st'>: Switch) =
        let G0, L0, v = E(G, L, condition)
        // i = 1 .. n
        // Evaluate literals, context doesn't matter
        let _, _, v1 = E(G0, L0, l1)
        ...
        let _, _, vn = E(G0, L0, ln)
        if there exists smallest i such that vi = v:
            E(G0, L0, sti)
        else:
            E(G0, L0, st')

    E(G, L, <name>: Identifier) =
        G, L, L[$name]
    E(G, L, <fname(arg1, ..., argn)>: FunctionCall) =
        G1, L1, vn = E(G, L, argn)
        ...
        G(n-1), L(n-1), v2 = E(G(n-2), L(n-2), arg2)
        Gn, Ln, v1 = E(G(n-1), L(n-1), arg1)
        Let <function fname (param1, ..., paramn) -> ret1, ..., retm block>
        be the function of name $fname visible at the point of the call.
        Let L' be a new local state such that
        L'[$parami] = vi and L'[$reti] = 0 for all i.
        Let G'', L'', mode = E(Gn, L', block)
        G'', Ln, L''[$ret1], ..., L''[$retm]
    E(G, L, l: StringLiteral) = G, L, utf8EncodeLeftAligned(l),
        where utf8EncodeLeftAligned performs a UTF-8 encoding of l
        and aligns it left into 32 bytes
    E(G, L, n: HexNumber) = G, L, hex(n)
        where hex is the hexadecimal decoding function
    E(G, L, n: DecimalNumber) = G, L, dec(n),
        where dec is the decimal decoding function

.. _opcodes:

Dialek EVM
-----------

Dialek default Yul saat ini adalah dialek EVM untuk versi EVM yang saat ini dipilih.
dengan versi EVM. Satu-satunya jenis yang tersedia dalam dialek ini
adalah ``u256``, tipe asli 256-bit dari Mesin Virtual Ethereum.
Karena ini adalah tipe default dari dialek, ini dapat dihilangkan.

Tabel berikut mencantumkan semua fungsi bawaan
(tergantung pada versi EVM) dan memberikan deskripsi singkat tentang
semantik dari fungsi/opcode.
Dokumen ini tidak ingin menjadi gambaran lengkap dari mesin virtual Ethereum.
Silakan merujuk ke dokumen lain jika Anda tertarik dengan semantik yang tepat.

Opcode yang ditandai dengan ``-`` tidak mengembalikan hasil dan yang lainnya mengembalikan tepat satu nilai.
Opcode yang ditandai dengan ``F``, ``H``, ``B``, ``C``, ``I`` dan ``L`` hadir sejak Frontier, Homestead,
Byzantium, Konstantinopel, Istanbul atau London berturutan.

Berikut ini, ``mem[a...b)`` menandakan byte memori mulai dari posisi ``a`` hingga
tetapi tidak termasuk posisi ``b`` dan ``storage[p]`` menandakan isi penyimpanan pada slot ``p``.

Karena Yul mengelola variabel lokal dan control-flow,
opcode yang mengganggu fitur ini tidak tersedia. Ini termasuk
instruksi ``dup`` dan ``swap`` serta instruksi ``jump``, label dan instruksi ``push``.

+-------------------------+-----+---+-----------------------------------------------------------------+
| Instruksi               |     |   | Penjelasan                                                      |
+=========================+=====+===+=================================================================+
| stop()                  + `-` | F | hentikan eksekusi, identik dengan return(0, 0)                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| add(x, y)               |     | F | x + y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sub(x, y)               |     | F | x - y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mul(x, y)               |     | F | x * y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| div(x, y)               |     | F | x / y atau 0 jika y == 0                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sdiv(x, y)              |     | F | x / y, untuk signed numbers di dua komplement, 0 jika y == 0    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mod(x, y)               |     | F | x % y, 0 jika y == 0                                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| smod(x, y)              |     | F | x % y, untuk signed numbers di komplement dua, 0 jika y == 0    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| exp(x, y)               |     | F | x dengan kekuatan y                                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| not(x)                  |     | F | bitwise "bukan" dari x (setiap bit x dinegasikan)               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| lt(x, y)                |     | F | 1 jika x < y, 0 jika tidak                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gt(x, y)                |     | F | 1 jika x > y, 0 jika tidak                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| slt(x, y)               |     | F | 1 jika x < y, 0 sebaliknya, untuk bilangan bertanda             |
|                         |     |   | dalam komplemen dua                                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sgt(x, y)               |     | F | 1 jika x > y, 0 sebaliknya, untuk bilangan bertanda             |
|                         |     |   | dalam komplemen dua                                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| eq(x, y)                |     | F | 1 jika x == y, 0 jika tidak                                     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| iszero(x)               |     | F | 1 jika x == 0, 0 jika tidak                                     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| and(x, y)               |     | F | bitwise "and" dari x dan y                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| or(x, y)                |     | F | bitwise "or" dari x dan y                                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| xor(x, y)               |     | F | bitwise "xor" dari x dan y                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| byte(n, x)              |     | F | byte ke-n dari x, dimana byte yang paling signifikan adalah     |
|                         |     |   | byte ke-0                                                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| shl(x, y)               |     | C | pergeseran logis ke kiri y dengan x bit                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| shr(x, y)               |     | C | pergeseran logis ke kanan y dengan x bit                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sar(x, y)               |     | C | signed aritmatika bergeser ke kanan y dengan x bit              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| addmod(x, y, m)         |     | F | (x + y) % m dengan arbitrary precision arithmetic, 0 jika m == 0|
+-------------------------+-----+---+-----------------------------------------------------------------+
| mulmod(x, y, m)         |     | F | (x * y) % m dengan arbitrary precision arithmetic, 0 jika m == 0|
+-------------------------+-----+---+-----------------------------------------------------------------+
| signextend(i, x)        |     | F | sign diperpanjang dari (i*8+7) bit dihitung dari yang paling    |
|                         |     |   | tidak signifikan                                                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| keccak256(p, n)         |     | F | keccak(mem[p...(p+n)))                                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| pc()                    |     | F | posisi saat ini dalam kode                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| pop(x)                  | `-` | F | buang nilai x                                                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mload(p)                |     | F | mem[p...(p+32))                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mstore(p, v)            | `-` | F | mem[p...(p+32)) := v                                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mstore8(p, v)           | `-` | F | mem[p] := v & 0xff (hanya memodifikasi satu byte)               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sload(p)                |     | F | storage[p]                                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sstore(p, v)            | `-` | F | storage[p] := v                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| msize()                 |     | F | ukuran memori, yaitu indeks memori yang diakses terbesar        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gas()                   |     | F | gas masih tersedia untuk dieksekusi                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| address()               |     | F | alamat konteks kontrak/eksekusi saat ini                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| balance(a)              |     | F | saldo wei di alamat a                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| selfbalance()           |     | I | setara dengan balance(address()), tetapi lebih murah            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| caller()                |     | F | pengirim panggilan (tidak termasuk ``delegatecall``)            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| callvalue()             |     | F | wei dikirim bersama dengan panggilan saat ini                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldataload(p)         |     | F | memanggil data mulai dari posisi p (32 byte)                    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldatasize()          |     | F | ukuran data panggilan dalam byte                                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldatacopy(t, f, s)   | `-` | F | salin s byte dari data panggilan di posisi f ke mem di posisi t |
+-------------------------+-----+---+-----------------------------------------------------------------+
| codesize()              |     | F | ukuran kode konteks kontrak/eksekusi saat ini                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| codecopy(t, f, s)       | `-` | F | salin s byte dari kode di posisi f ke mem di posisi t           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodesize(a)          |     | F | ukuran kode di alamat a                                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodecopy(a, t, f, s) | `-` | F | seperti codecopy(t, f, s) tapi ambil kode di alamat a           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| returndatasize()        |     | B | ukuran returndata terakhir                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| returndatacopy(t, f, s) | `-` | B | salin s byte dari returndata di posisi f ke mem di posisi t     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodehash(a)          |     | C | kode hash alamat a                                              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| create(v, p, n)         |     | F | buat kontrak baru dengan kode mem[p...(p+n)) dan kirim v wei    |
|                         |     |   | dan kembalikan alamat baru; mengembalikan 0 pada kesalahan      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| create2(v, p, n, s)     |     | C | buat kontrak baru dengan kode mem[p...(p+n)) di alamat          |
|                         |     |   | keccak256(0xff . this . s . keccak256(mem[p...(p+n)))           |
|                         |     |   | dan kirim v wei dan kembalikan alamat baru, di mana ``0xff``    |
|                         |     |   | adalah nilai 1 byte, ``this`` adalah alamat kontrak saat ini    |
|                         |     |   | sebagai nilai 20 byte dan ``s`` adalah nilai 256-bit big-endian;|
|                         |     |   | mengembalikan 0 pada kesalahan                                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| call(g, a, v, in,       |     | F | kontrak panggilan di alamat a dengan input mem[in...(in+insize))|
| insize, out, outsize)   |     |   | menyediakan gas g dan v wei dan area output                     |
|                         |     |   | mem[out...(out+outsize)) menghasilkan 0 saat error              |
|                         |     |   | (mis. out of gas) dan 1 jika sukses                             |
|                         |     |   | :ref:`Lihat lebih banyak <yul-call-return-area>`                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| callcode(g, a, v, in,   |     | F | identik dengan ``call`` tetapi hanya menggunakan kode dari a    |
| insize, out, outsize)   |     |   | dan tetap dalam konteks kontrak saat ini jika tidak             |
|                         |     |   | :ref:`Lihat lebih banyak <yul-call-return-area>`                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| delegatecall(g, a, in,  |     | H | identik dengan ``callcode`` tetapi juga tetap menggunakan       |
| insize, out, outsize)   |     |   | ``caller`` dan ``callvalue``                                    |
|                         |     |   | :ref:`Lihat lebih banyak <yul-call-return-area>`                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| staticcall(g, a, in,    |     | B | identik dengan ``call(g, a, 0, in, insize, out, outsize)``      |
| insize, out, outsize)   |     |   | tetapi tidak mengizinkan modifikasi state                       |
|                         |     |   | :ref:`Lihat lebih banyak <yul-call-return-area>`                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| return(p, s)            | `-` | F | akhiri eksekusi, kembalikan data mem[p...(p+s))                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| revert(p, s)            | `-` | B | akhiri eksekusi, kembalikan perubahan state, kembalikan data    |
|                         |     |   | mem[p...(p+s))                                                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| selfdestruct(a)         | `-` | F | akhiri eksekusi, hancurkan kontrak saat ini dan kirim dana ke a |
+-------------------------+-----+---+-----------------------------------------------------------------+
| invalid()               | `-` | F | akhiri eksekusi dengan instruksi yang tidak valid               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log0(p, s)              | `-` | F | log tanpa topik dan data mem[p...(p+s))                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log1(p, s, t1)          | `-` | F | log dengan topik t1 dan data mem[p...(p+s))                     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log2(p, s, t1, t2)      | `-` | F | log dengan topik t1, t2 dan data mem[p...(p+s))                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log3(p, s, t1, t2, t3)  | `-` | F | log dengan topik t1, t2, t3 dan data mem[p...(p+s))             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log4(p, s, t1, t2, t3,  | `-` | F | log dengan topik t1, t2, t3, t4 dan data mem[p...(p+s))         |
| t4)                     |     |   |                                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| chainid()               |     | I | ID chain pelaksana (EIP-1344)                                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| basefee()               |     | L | biaya dasar blok saat ini (EIP-3198 and EIP-1559)               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| origin()                |     | F | pengirim transaksi                                              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gasprice()              |     | F | harga gas dari transaksi                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| blockhash(b)            |     | F | hash blok nr b - hanya untuk 256 blok terakhir tidak            |
|                         |     |   | termasuk saat ini                                               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| coinbase()              |     | F | penerima manfaat pertambangan saat ini                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| timestamp()             |     | F | stempel waktu blok saat ini dalam hitungan detik sejak zaman    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| number()                |     | F | nomor blok saat ini                                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| difficulty()            |     | F | kesulitan blok saat ini                                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gaslimit()              |     | F | batas gas blok dari blok saat ini                               |
+-------------------------+-----+---+-----------------------------------------------------------------+

.. _yul-call-return-area:

.. note::
  Instruksi ``call*`` menggunakan parameter ``out`` dan ``outsize`` untuk menentukan area di memori tempat
  pengembalian atau kegagalan data ditempatkan. Area ini ditulis tergantung pada berapa banyak byte yang disebut pengembalian kontrak.
  Jika mengembalikan lebih banyak data, hanya byte ``outsize`` pertama yang akan ditulis. Anda dapat mengakses sisa data
  menggunakan opcode ``returndatacopy``. Jika mengembalikan lebih sedikit data, maka byte yang tersisa tidak disentuh sama sekali.
  Anda perlu menggunakan opcode ``returndatasize`` untuk memeriksa bagian mana dari area memori ini yang berisi data yang dikembalikan.
  Byte yang tersisa akan mempertahankan nilainya seperti sebelum panggilan.


Dalam beberapa dialek internal, ada fungsi tambahan:

datasize, dataoffset, datacopy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Fungsi ``datasize(x)``, ``dataoffset(x)`` dan ``datacopy(t, f, l)``
digunakan untuk mengakses bagian lain dari objek Yul.

``datasize`` dan ``dataoffset`` hanya dapat mengambil literal string (nama objek lain)
sebagai argumen dan masing-masing mengembalikan ukuran dan offset di area data.
Untuk EVM, fungsi ``datacopy`` sama dengan ``codecopy``.


setimmutable, loadimmutable
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Fungsi ``setimmutable(offset, "name", value)`` dan ``loadimmutable("name")`` adalah
digunakan untuk mekanisme yang tidak dapat diubah di Solidity dan tidak dipetakan dengan baik ke Yul murni.
Panggilan ke ``setimmutable(offset, "name", value)`` mengasumsikan bahwa kode runtime kontrak
berisi nama yang tidak dapat diubah yang diberikan disalin ke memori pada offset ``offset`` dan akan menulis ``value`` ke semua
posisi dalam memori (relatif terhadap ``offset``) yang berisi placeholder yang dihasilkan untuk panggilan
ke ``loadimmutable("name")`` dalam kode runtime.


linkersymbol
^^^^^^^^^^^^
Fungsi ``linkerssymbol("library_id")`` adalah placeholder untuk literal alamat yang akan diganti
oleh penghubung.
Argumen pertama dan satu-satunya harus berupa string literal dan secara unik mewakili alamat yang akan disisipkan.
Pengidentifikasi dapat sewenang-wenang tetapi ketika kompiler menghasilkan kode Yul dari sumber Solidity,
itu menggunakan nama perpustakaan yang memenuhi syarat dengan nama unit sumber yang mendefinisikan library itu.
Untuk menautkan kode dengan alamat perpustakaan tertentu, pengenal yang sama harus diberikan ke
Opsi ``--libraries`` pada baris perintah.

Misalnya kode ini

.. code-block:: yul

    let a := linkersymbol("file.sol:Math")

setara dengan

.. code-block:: yul

    let a := 0x1234567890123456789012345678901234567890

ketika penaut dipanggil dengan pilihan ``--libraries "file.sol:Math=0x1234567890123456789012345678901234567890``.

Lihat :ref:`Menggunakan Commandline Compiler <commandline-compiler>` untuk detail tentang linker Solidity.

memoryguard
^^^^^^^^^^^

Fungsi ini tersedia dalam dialek EVM dengan objek. Pemanggil
``let ptr := memoryguard(size)`` (di mana ``size`` harus berupa angka literal)
menjanjikan bahwa mereka hanya menggunakan memori dalam rentang ``[0, size)`` atau
rentang tak terbatas mulai dari ``ptr``.

Karena adanya panggilan ``memoryguard`` menunjukkan bahwa semua akses memori
mematuhi batasan ini, ini memungkinkan pengoptimal untuk melakukan tambahan
langkah-langkah pengoptimalan, misalnya penghindar batas tumpukan, yang mencoba memindahkan
variabel stack yang tidak dapat dijangkau ke memori.

Pengoptimal Yul berjanji untuk hanya menggunakan rentang memori ``[size, ptr)`` untuk tujuannya.
Jika pengoptimal tidak perlu mencadangkan memori apa pun, ia akan menyimpan ``ptr == size`` itu.

``memoryguard`` dapat dipanggil beberapa kali, tetapi harus memiliki literal yang sama dengan argumen
dalam satu subjek Yul. Jika setidaknya satu panggilan ``memoryguard`` ditemukan dalam subobjek,
langkah-langkah pengoptimal tambahan akan dijalankan di atasnya.


.. _yul-verbatim:

verbatim
^^^^^^^^

Kumpulan fungsi bawaan ``verbatim...`` memungkinkan Anda membuat bytecode untuk opcodes
yang tidak diketahui oleh kompiler Yul. Ini juga memungkinkan Anda untuk membuat
urutan bytecode yang tidak akan diubah oleh pengoptimal.

Fungsinya adalah ``verbatim_<n>i_<m>o("<data>", ...)``, di mana

- ``n`` adalah desimal antara 0 dan 99 yang menentukan jumlah slot/variabel tumpukan input
- ``m`` adalah desimal antara 0 dan 99 yang menentukan jumlah slot/variabel tumpukan keluaran
- ``data`` adalah string literal yang berisi urutan byte

Jika Anda misalnya ingin mendefinisikan fungsi yang mengalikan input
oleh dua, tanpa pengoptimal menyentuh dua konstan, Anda dapat menggunakan

.. code-block:: yul

    let x := calldataload(0)
    let double := verbatim_1i_1o(hex"600202", x)

Kode ini akan menghasilkan opcode ``dup1`` untuk mengambil ``x``
(pengoptimal mungkin secara langsung menggunakan kembali hasil
opcode ``calldataload``)
langsung diikuti oleh ``600202``. Kode diasumsikan menggunakan
nilai ``x`` yang disalin dan menghasilkan hasil di bagian atas stack.
Kompiler kemudian menghasilkan kode untuk mengalokasikan slot tumpukan untuk
``double`` dan menyimpan hasilnya di sana.

Seperti semua opcode, argumen disusun di stack
dengan argumen paling kiri di atas, sedangkan nilai yang dikembalikan
diasumsikan ditata sedemikian rupa sehingga variabel paling kanan adalah
di bagian atas stack.

Karena ``verbatim`` dapat digunakan untuk menghasilkan opcode arbitrer
atau bahkan opcode yang tidak diketahui oleh kompiler Solidity, harus berhati-hati
saat menggunakan ``verbatim`` bersama dengan pengoptimal. Bahkan ketika
pengoptimal dimatikan, pembuat kode harus menentukan
tata letak stack, yang berarti bahwa mis. menggunakan ``verbatim`` untuk memodifikasi
tinggi stack dapat menyebabkan perilaku tidak terdefinisi.

Berikut ini adalah daftar pembatasan yang tidak lengkap pada
bytecode verbatim yang tidak diperiksa oleh
kompiler. Pelanggaran terhadap pembatasan ini dapat mengakibatkan
perilaku yang tidak terdefinisi.

- Control-flow tidak boleh melompat ke dalam atau keluar dari blok verbatim,
  tetapi dapat melompat dalam blok verbatim yang sama.
- Konten Stack selain dari parameter input dan output
  tidak boleh diakses.
- Perbedaan tinggi stack harus tepat ``m - n``
  (slot keluaran dikurangi slot masukan).
- Bytecode verbatim tidak dapat membuat asumsi apa pun tentang
  bytecode sekitarnya. Semua parameter yang diperlukan harus
  diteruskan sebagai variabel stack.

Pengoptimal tidak menganalisis bytecode verbatim dan selalu
mengasumsikan bahwa itu mengubah semua aspek state dan dengan demikian hanya bisa
melakukan sedikit pengoptimalan di seluruh pemanggilan fungsi ``verbatim``.

Pengoptimal memperlakukan bytecode verbatim sebagai blok kode buram.
Itu tidak akan membaginya tetapi mungkin bergerak, duplikat
atau gabungkan dengan blok bytecode verbatim yang identik.
Jika blok bytecode verbatim tidak dapat dijangkau oleh aliran kontrol,
itu bisa dihapus.


.. warning::

    Selama diskusi tentang apakah peningkatan EVM
    dapat merusak kontrak pintar yang ada, fitur di dalam ``verbatim``
    tidak dapat menerima pertimbangan yang sama seperti yang
    digunakan oleh kompiler Solidity itu sendiri.

.. note::

    Untuk menghindari kebingungan, semua pengidentifikasi yang dimulai dengan string ``verbatim`` dicadangkan
    dan tidak dapat digunakan untuk pengidentifikasi yang ditentukan pengguna.

.. _yul-object:

Spesifikasi Objek Yul
=====================

Objek Yul digunakan untuk mengelompokkan bagian kode dan data bernama.
Fungsi ``datasize``, ``dataoffset`` dan ``datacopy``
dapat digunakan untuk mengakses bagian ini dari dalam kode.
String hex dapat digunakan untuk menentukan data dalam pengkodean hex,
string biasa dalam penyandian asli. Untuk kode,
``datacopy`` akan mengakses representasi biner rakitannya.

.. code-block:: none

    Object = 'object' StringLiteral '{' Code ( Object | Data )* '}'
    Code = 'code' Block
    Data = 'data' StringLiteral ( HexLiteral | StringLiteral )
    HexLiteral = 'hex' ('"' ([0-9a-fA-F]{2})* '"' | '\'' ([0-9a-fA-F]{2})* '\'')
    StringLiteral = '"' ([^"\r\n\\] | '\\' .)* '"'

Di atas, ``Block`` mengacu pada ``Block`` dalam tata bahasa kode Yul yang dijelaskan pada bab sebelumnya.

.. note::

    Objek data atau sub-objek yang namanya mengandung ``.`` dapat didefinisikan
    tetapi tidak mungkin untuk mengaksesnya melalui ``datasize``,
    ``dataoffset`` atau ``datacopy`` karena ``.`` digunakan sebagai pemisah
    untuk mengakses objek di dalam objek lain.

.. note::

    Objek data yang disebut ``".metadata"`` memiliki arti khusus:
    Itu tidak dapat diakses dari kode dan selalu ditambahkan ke bagian paling akhir
    bytecode, terlepas dari posisinya di objek.

    Objek data lain dengan signifikansi khusus dapat ditambahkan di
    masa depan, tetapi nama mereka akan selalu dimulai dengan ``.``.


Contoh Yul Object ditunjukkan di bawah ini:

.. code-block:: yul

    // A contract consists of a single object with sub-objects representing
    // the code to be deployed or other contracts it can create.
    // The single "code" node is the executable code of the object.
    // Every (other) named object or data section is serialized and
    // made accessible to the special built-in functions datacopy / dataoffset / datasize
    // The current object, sub-objects and data items inside the current object
    // are in scope.
    object "Contract1" {
        // This is the constructor code of the contract.
        code {
            function allocate(size) -> ptr {
                ptr := mload(0x40)
                if iszero(ptr) { ptr := 0x60 }
                mstore(0x40, add(ptr, size))
            }

            // first create "Contract2"
            let size := datasize("Contract2")
            let offset := allocate(size)
            // This will turn into codecopy for EVM
            datacopy(offset, dataoffset("Contract2"), size)
            // constructor parameter is a single number 0x1234
            mstore(add(offset, size), 0x1234)
            pop(create(offset, add(size, 32), 0))

            // now return the runtime object (the currently
            // executing code is the constructor code)
            size := datasize("runtime")
            offset := allocate(size)
            // This will turn into a memory->memory copy for Ewasm and
            // a codecopy for EVM
            datacopy(offset, dataoffset("runtime"), size)
            return(offset, size)
        }

        data "Table2" hex"4123"

        object "runtime" {
            code {
                function allocate(size) -> ptr {
                    ptr := mload(0x40)
                    if iszero(ptr) { ptr := 0x60 }
                    mstore(0x40, add(ptr, size))
                }

                // runtime code

                mstore(0, "Hello, World!")
                return(0, 0x20)
            }
        }

        // Embedded object. Use case is that the outside is a factory contract,
        // and Contract2 is the code to be created by the factory
        object "Contract2" {
            code {
                // code here ...
            }

            object "runtime" {
                code {
                    // code here ...
                }
            }

            data "Table1" hex"4123"
        }
    }

Yul Optimizer
=============

Pengoptimal Yul beroperasi pada kode Yul dan menggunakan bahasa yang sama untuk input, output, dan
intermediate state. Ini memungkinkan debugging dan verifikasi pengoptimal dengan mudah.

Silakan lihat dokumentasi umum :ref:`optimizer <optimizer>`
untuk detail selengkapnya tentang berbagai tahapan pengoptimalan dan cara menggunakan pengoptimal.

Jika Anda ingin menggunakan Solidity dalam mode Yul yang berdiri sendiri, Anda mengaktifkan pengoptimal menggunakan ``--optimize``
dan secara opsional tentukan :ref:`jumlah eksekusi kontrak yang diharapkan <optimizer-parameter-runs>` dengan
``--optimize-runs``:

.. code-block:: sh

    solc --strict-assembly --optimize --optimize-runs 200

Dalam mode Solidity, pengoptimal Yul diaktifkan bersama dengan pengoptimal biasa.

Urutan Langkah Pengoptimalan
----------------------------

Secara default, pengoptimal Yul menerapkan urutan langkah pengoptimalan yang telah ditentukan sebelumnya ke rakitan yang dihasilkan.
Anda dapat mengganti urutan ini dan menyediakan urutan Anda sendiri menggunakan opsi ``--yul-optimizations``:

.. code-block:: sh

    solc --optimize --ir-optimized --yul-optimizations 'dhfoD[xarrscLMcCTU]uljmul'

Urutan langkah sangat penting dan mempengaruhi kualitas output.
Selain itu, menerapkan suatu langkah dapat mengungkap peluang pengoptimalan baru bagi orang lain yang sudah ada
diterapkan sehingga langkah berulang seringkali bermanfaat.
Dengan melampirkan bagian dari urutan dalam tanda kurung siku (``[]``), Anda memberi tahu pengoptimal untuk berulang kali
menerapkan bagian itu sampai tidak lagi meningkatkan ukuran rakitan yang dihasilkan.
Anda dapat menggunakan tanda kurung beberapa kali dalam satu urutan tetapi tanda kurung tidak dapat nested.

Langkah-langkah pengoptimalan berikut tersedia:

============ ===============================
Abbreviation Full name
============ ===============================
``f``        ``BlockFlattener``
``l``        ``CircularReferencesPruner``
``c``        ``CommonSubexpressionEliminator``
``C``        ``ConditionalSimplifier``
``U``        ``ConditionalUnsimplifier``
``n``        ``ControlFlowSimplifier``
``D``        ``DeadCodeEliminator``
``v``        ``EquivalentFunctionCombiner``
``e``        ``ExpressionInliner``
``j``        ``ExpressionJoiner``
``s``        ``ExpressionSimplifier``
``x``        ``ExpressionSplitter``
``I``        ``ForLoopConditionIntoBody``
``O``        ``ForLoopConditionOutOfBody``
``o``        ``ForLoopInitRewriter``
``i``        ``FullInliner``
``g``        ``FunctionGrouper``
``h``        ``FunctionHoister``
``F``        ``FunctionSpecializer``
``T``        ``LiteralRematerialiser``
``L``        ``LoadResolver``
``M``        ``LoopInvariantCodeMotion``
``r``        ``RedundantAssignEliminator``
``R``        ``ReasoningBasedSimplifier`` - highly experimental
``m``        ``Rematerialiser``
``V``        ``SSAReverser``
``a``        ``SSATransform``
``t``        ``StructuralSimplifier``
``u``        ``UnusedPruner``
``p``        ``UnusedFunctionParameterPruner``
``d``        ``VarDeclInitializer``
============ ===============================

Beberapa langkah bergantung pada properti yang dipastikan oleh ``BlockFlattener``, ``FunctionGrouper``, ``ForLoopInitRewriter``.
Untuk alasan ini, pengoptimal Yul selalu menerapkannya sebelum menerapkan langkah apa pun yang disediakan oleh pengguna.

ReasoningBasedSimplifier adalah langkah pengoptimal yang saat ini tidak diaktifkan
dalam rangkaian langkah default. Ini menggunakan pemecah SMT untuk menyederhanakan ekspresi aritmatika
dan kondisi boolean. Itu belum menerima pengujian atau validasi menyeluruh dan dapat menghasilkan
hasil yang tidak dapat direproduksi, jadi harap gunakan dengan hati-hati!

.. _erc20yul:

Contoh ERC20 Lengkap
======================

.. code-block:: yul

    object "Token" {
        code {
            // Store the creator in slot zero.
            sstore(0, caller())

            // Deploy the contract
            datacopy(0, dataoffset("runtime"), datasize("runtime"))
            return(0, datasize("runtime"))
        }
        object "runtime" {
            code {
                // Protection against sending Ether
                require(iszero(callvalue()))

                // Dispatcher
                switch selector()
                case 0x70a08231 /* "balanceOf(address)" */ {
                    returnUint(balanceOf(decodeAsAddress(0)))
                }
                case 0x18160ddd /* "totalSupply()" */ {
                    returnUint(totalSupply())
                }
                case 0xa9059cbb /* "transfer(address,uint256)" */ {
                    transfer(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                case 0x23b872dd /* "transferFrom(address,address,uint256)" */ {
                    transferFrom(decodeAsAddress(0), decodeAsAddress(1), decodeAsUint(2))
                    returnTrue()
                }
                case 0x095ea7b3 /* "approve(address,uint256)" */ {
                    approve(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                case 0xdd62ed3e /* "allowance(address,address)" */ {
                    returnUint(allowance(decodeAsAddress(0), decodeAsAddress(1)))
                }
                case 0x40c10f19 /* "mint(address,uint256)" */ {
                    mint(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                default {
                    revert(0, 0)
                }

                function mint(account, amount) {
                    require(calledByOwner())

                    mintTokens(amount)
                    addToBalance(account, amount)
                    emitTransfer(0, account, amount)
                }
                function transfer(to, amount) {
                    executeTransfer(caller(), to, amount)
                }
                function approve(spender, amount) {
                    revertIfZeroAddress(spender)
                    setAllowance(caller(), spender, amount)
                    emitApproval(caller(), spender, amount)
                }
                function transferFrom(from, to, amount) {
                    decreaseAllowanceBy(from, caller(), amount)
                    executeTransfer(from, to, amount)
                }

                function executeTransfer(from, to, amount) {
                    revertIfZeroAddress(to)
                    deductFromBalance(from, amount)
                    addToBalance(to, amount)
                    emitTransfer(from, to, amount)
                }


                /* ---------- calldata decoding functions ----------- */
                function selector() -> s {
                    s := div(calldataload(0), 0x100000000000000000000000000000000000000000000000000000000)
                }

                function decodeAsAddress(offset) -> v {
                    v := decodeAsUint(offset)
                    if iszero(iszero(and(v, not(0xffffffffffffffffffffffffffffffffffffffff)))) {
                        revert(0, 0)
                    }
                }
                function decodeAsUint(offset) -> v {
                    let pos := add(4, mul(offset, 0x20))
                    if lt(calldatasize(), add(pos, 0x20)) {
                        revert(0, 0)
                    }
                    v := calldataload(pos)
                }
                /* ---------- calldata encoding functions ---------- */
                function returnUint(v) {
                    mstore(0, v)
                    return(0, 0x20)
                }
                function returnTrue() {
                    returnUint(1)
                }

                /* -------- events ---------- */
                function emitTransfer(from, to, amount) {
                    let signatureHash := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
                    emitEvent(signatureHash, from, to, amount)
                }
                function emitApproval(from, spender, amount) {
                    let signatureHash := 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
                    emitEvent(signatureHash, from, spender, amount)
                }
                function emitEvent(signatureHash, indexed1, indexed2, nonIndexed) {
                    mstore(0, nonIndexed)
                    log3(0, 0x20, signatureHash, indexed1, indexed2)
                }

                /* -------- storage layout ---------- */
                function ownerPos() -> p { p := 0 }
                function totalSupplyPos() -> p { p := 1 }
                function accountToStorageOffset(account) -> offset {
                    offset := add(0x1000, account)
                }
                function allowanceStorageOffset(account, spender) -> offset {
                    offset := accountToStorageOffset(account)
                    mstore(0, offset)
                    mstore(0x20, spender)
                    offset := keccak256(0, 0x40)
                }

                /* -------- storage access ---------- */
                function owner() -> o {
                    o := sload(ownerPos())
                }
                function totalSupply() -> supply {
                    supply := sload(totalSupplyPos())
                }
                function mintTokens(amount) {
                    sstore(totalSupplyPos(), safeAdd(totalSupply(), amount))
                }
                function balanceOf(account) -> bal {
                    bal := sload(accountToStorageOffset(account))
                }
                function addToBalance(account, amount) {
                    let offset := accountToStorageOffset(account)
                    sstore(offset, safeAdd(sload(offset), amount))
                }
                function deductFromBalance(account, amount) {
                    let offset := accountToStorageOffset(account)
                    let bal := sload(offset)
                    require(lte(amount, bal))
                    sstore(offset, sub(bal, amount))
                }
                function allowance(account, spender) -> amount {
                    amount := sload(allowanceStorageOffset(account, spender))
                }
                function setAllowance(account, spender, amount) {
                    sstore(allowanceStorageOffset(account, spender), amount)
                }
                function decreaseAllowanceBy(account, spender, amount) {
                    let offset := allowanceStorageOffset(account, spender)
                    let currentAllowance := sload(offset)
                    require(lte(amount, currentAllowance))
                    sstore(offset, sub(currentAllowance, amount))
                }

                /* ---------- utility functions ---------- */
                function lte(a, b) -> r {
                    r := iszero(gt(a, b))
                }
                function gte(a, b) -> r {
                    r := iszero(lt(a, b))
                }
                function safeAdd(a, b) -> r {
                    r := add(a, b)
                    if or(lt(r, a), lt(r, b)) { revert(0, 0) }
                }
                function calledByOwner() -> cbo {
                    cbo := eq(owner(), caller())
                }
                function revertIfZeroAddress(addr) {
                    require(addr)
                }
                function require(condition) {
                    if iszero(condition) { revert(0, 0) }
                }
            }
        }
    }
=======
.. _yul:

###
Yul
###

.. index:: ! assembly, ! asm, ! evmasm, ! yul, julia, iulia

Yul (previously also called JULIA or IULIA) is an intermediate language that can be
compiled to bytecode for different backends.

Support for EVM 1.0, EVM 1.5 and Ewasm is planned, and it is designed to
be a usable common denominator of all three
platforms. It can already be used in stand-alone mode and
for "inline assembly" inside Solidity
and there is an experimental implementation of the Solidity compiler
that uses Yul as an intermediate language. Yul is a good target for
high-level optimisation stages that can benefit all target platforms equally.

Motivation and High-level Description
=====================================

The design of Yul tries to achieve several goals:

1. Programs written in Yul should be readable, even if the code is generated by a compiler from Solidity or another high-level language.
2. Control flow should be easy to understand to help in manual inspection, formal verification and optimization.
3. The translation from Yul to bytecode should be as straightforward as possible.
4. Yul should be suitable for whole-program optimization.

In order to achieve the first and second goal, Yul provides high-level constructs
like ``for`` loops, ``if`` and ``switch`` statements and function calls. These should
be sufficient for adequately representing the control flow for assembly programs.
Therefore, no explicit statements for ``SWAP``, ``DUP``, ``JUMPDEST``, ``JUMP`` and ``JUMPI``
are provided, because the first two obfuscate the data flow
and the last two obfuscate control flow. Furthermore, functional statements of
the form ``mul(add(x, y), 7)`` are preferred over pure opcode statements like
``7 y x add mul`` because in the first form, it is much easier to see which
operand is used for which opcode.

Even though it was designed for stack machines, Yul does not expose the complexity of the stack itself.
The programmer or auditor should not have to worry about the stack.

The third goal is achieved by compiling the
higher level constructs to bytecode in a very regular way.
The only non-local operation performed
by the assembler is name lookup of user-defined identifiers (functions, variables, ...)
and cleanup of local variables from the stack.

To avoid confusions between concepts like values and references,
Yul is statically typed. At the same time, there is a default type
(usually the integer word of the target machine) that can always
be omitted to help readability.

To keep the language simple and flexible, Yul does not have
any built-in operations, functions or types in its pure form.
These are added together with their semantics when specifying a dialect of Yul,
which allows specializing Yul to the requirements of different
target platforms and feature sets.

Currently, there is only one specified dialect of Yul. This dialect uses
the EVM opcodes as builtin functions
(see below) and defines only the type ``u256``, which is the native 256-bit
type of the EVM. Because of that, we will not provide types in the examples below.


Simple Example
==============

The following example program is written in the EVM dialect and computes exponentiation.
It can be compiled using ``solc --strict-assembly``. The builtin functions
``mul`` and ``div`` compute product and division, respectively.

.. code-block:: yul

    {
        function power(base, exponent) -> result
        {
            switch exponent
            case 0 { result := 1 }
            case 1 { result := base }
            default
            {
                result := power(mul(base, base), div(exponent, 2))
                switch mod(exponent, 2)
                    case 1 { result := mul(base, result) }
            }
        }
    }

It is also possible to implement the same function using a for-loop
instead of with recursion. Here, ``lt(a, b)`` computes whether ``a`` is less than ``b``.
less-than comparison.

.. code-block:: yul

    {
        function power(base, exponent) -> result
        {
            result := 1
            for { let i := 0 } lt(i, exponent) { i := add(i, 1) }
            {
                result := mul(result, base)
            }
        }
    }

At the :ref:`end of the section <erc20yul>`, a complete implementation of
the ERC-20 standard can be found.



Stand-Alone Usage
=================

You can use Yul in its stand-alone form in the EVM dialect using the Solidity compiler.
This will use the :ref:`Yul object notation <yul-object>` so that it is possible to refer
to code as data to deploy contracts. This Yul mode is available for the commandline compiler
(use ``--strict-assembly``) and for the :ref:`standard-json interface <compiler-api>`:

.. code-block:: json

    {
        "language": "Yul",
        "sources": { "input.yul": { "content": "{ sstore(0, 1) }" } },
        "settings": {
            "outputSelection": { "*": { "*": ["*"], "": [ "*" ] } },
            "optimizer": { "enabled": true, "details": { "yul": true } }
        }
    }

.. warning::

    Yul is in active development and bytecode generation is only fully implemented for the EVM dialect of Yul
    with EVM 1.0 as target.


Informal Description of Yul
===========================

In the following, we will talk about each individual aspect
of the Yul language. In examples, we will use the default EVM dialect.

Syntax
------

Yul parses comments, literals and identifiers in the same way as Solidity,
so you can e.g. use ``//`` and ``/* */`` to denote comments.
There is one exception: Identifiers in Yul can contain dots: ``.``.

Yul can specify "objects" that consist of code, data and sub-objects.
Please see :ref:`Yul Objects <yul-object>` below for details on that.
In this section, we are only concerned with the code part of such an object.
This code part always consists of a curly-braces
delimited block. Most tools support specifying just a code block
where an object is expected.

Inside a code block, the following elements can be used
(see the later sections for more details):

- literals, i.e. ``0x123``, ``42`` or ``"abc"`` (strings up to 32 characters)
- calls to builtin functions, e.g. ``add(1, mload(0))``
- variable declarations, e.g. ``let x := 7``, ``let x := add(y, 3)`` or ``let x`` (initial value of 0 is assigned)
- identifiers (variables), e.g. ``add(3, x)``
- assignments, e.g. ``x := add(y, 3)``
- blocks where local variables are scoped inside, e.g. ``{ let x := 3 { let y := add(x, 1) } }``
- if statements, e.g. ``if lt(a, b) { sstore(0, 1) }``
- switch statements, e.g. ``switch mload(0) case 0 { revert() } default { mstore(0, 1) }``
- for loops, e.g. ``for { let i := 0} lt(i, 10) { i := add(i, 1) } { mstore(i, 7) }``
- function definitions, e.g. ``function f(a, b) -> c { c := add(a, b) }``

Multiple syntactical elements can follow each other simply separated by
whitespace, i.e. there is no terminating ``;`` or newline required.

Literals
--------

As literals, you can use:

- Integer constants in decimal or hexadecimal notation.

- ASCII strings (e.g. ``"abc"``), which may contain hex escapes ``\xNN`` and Unicode escapes ``\uNNNN`` where ``N`` are hexadecimal digits.

- Hex strings (e.g. ``hex"616263"``).

In the EVM dialect of Yul, literals represent 256-bit words as follows:

- Decimal or hexadecimal constants must be less than ``2**256``.
  They represent the 256-bit word with that value as an unsigned integer in big endian encoding.

- An ASCII string is first viewed as a byte sequence, by viewing
  a non-escape ASCII character as a single byte whose value is the ASCII code,
  an escape ``\xNN`` as single byte with that value, and
  an escape ``\uNNNN`` as the UTF-8 sequence of bytes for that code point.
  The byte sequence must not exceed 32 bytes.
  The byte sequence is padded with zeros on the right to reach 32 bytes in length;
  in other words, the string is stored left-aligned.
  The padded byte sequence represents a 256-bit word whose most significant 8 bits are the ones from the first byte,
  i.e. the bytes are interpreted in big endian form.

- A hex string is first viewed as a byte sequence, by viewing
  each pair of contiguous hex digits as a byte.
  The byte sequence must not exceed 32 bytes (i.e. 64 hex digits), and is treated as above.

When compiling for the EVM, this will be translated into an
appropriate ``PUSHi`` instruction. In the following example,
``3`` and ``2`` are added resulting in 5 and then the
bitwise ``and`` with the string "abc" is computed.
The final value is assigned to a local variable called ``x``.

The 32-byte limit above does not apply to string literals passed to builtin functions that require
literal arguments (e.g. ``setimmutable`` or ``loadimmutable``). Those strings never end up in the
generated bytecode.

.. code-block:: yul

    let x := and("abc", add(3, 2))

Unless it is the default type, the type of a literal
has to be specified after a colon:

.. code-block:: yul

    // This will not compile (u32 and u256 type not implemented yet)
    let x := and("abc":u32, add(3:u256, 2:u256))


Function Calls
--------------

Both built-in and user-defined functions (see below) can be called
in the same way as shown in the previous example.
If the function returns a single value, it can be directly used
inside an expression again. If it returns multiple values,
they have to be assigned to local variables.

.. code-block:: yul

    function f(x, y) -> a, b { /* ... */ }
    mstore(0x80, add(mload(0x80), 3))
    // Here, the user-defined function `f` returns two values.
    let x, y := f(1, mload(0))

For built-in functions of the EVM, functional expressions
can be directly translated to a stream of opcodes:
You just read the expression from right to left to obtain the
opcodes. In the case of the first line in the example, this
is ``PUSH1 3 PUSH1 0x80 MLOAD ADD PUSH1 0x80 MSTORE``.

For calls to user-defined functions, the arguments are also
put on the stack from right to left and this is the order
in which argument lists are evaluated. The return values,
though, are expected on the stack from left to right,
i.e. in this example, ``y`` is on top of the stack and ``x``
is below it.

Variable Declarations
---------------------

You can use the ``let`` keyword to declare variables.
A variable is only visible inside the
``{...}``-block it was defined in. When compiling to the EVM,
a new stack slot is created that is reserved
for the variable and automatically removed again when the end of the block
is reached. You can provide an initial value for the variable.
If you do not provide a value, the variable will be initialized to zero.

Since variables are stored on the stack, they do not directly
influence memory or storage, but they can be used as pointers
to memory or storage locations in the built-in functions
``mstore``, ``mload``, ``sstore`` and ``sload``.
Future dialects might introduce specific types for such pointers.

When a variable is referenced, its current value is copied.
For the EVM, this translates to a ``DUP`` instruction.

.. code-block:: yul

    {
        let zero := 0
        let v := calldataload(zero)
        {
            let y := add(sload(v), 1)
            v := y
        } // y is "deallocated" here
        sstore(v, zero)
    } // v and zero are "deallocated" here


If the declared variable should have a type different from the default type,
you denote that following a colon. You can also declare multiple
variables in one statement when you assign from a function call
that returns multiple values.

.. code-block:: yul

    // This will not compile (u32 and u256 type not implemented yet)
    {
        let zero:u32 := 0:u32
        let v:u256, t:u32 := f()
        let x, y := g()
    }

Depending on the optimiser settings, the compiler can free the stack slots
already after the variable has been used for
the last time, even though it is still in scope.


Assignments
-----------

Variables can be assigned to after their definition using the
``:=`` operator. It is possible to assign multiple
variables at the same time. For this, the number and types of the
values have to match.
If you want to assign the values returned from a function that has
multiple return parameters, you have to provide multiple variables.
The same variable may not occur multiple times on the left-hand side of
an assignment, e.g. ``x, x := f()`` is invalid.

.. code-block:: yul

    let v := 0
    // re-assign v
    v := 2
    let t := add(v, 2)
    function f() -> a, b { }
    // assign multiple values
    v, t := f()


If
--

The if statement can be used for conditionally executing code.
No "else" block can be defined. Consider using "switch" instead (see below) if
you need multiple alternatives.

.. code-block:: yul

    if lt(calldatasize(), 4) { revert(0, 0) }

The curly braces for the body are required.

Switch
------

You can use a switch statement as an extended version of the if statement.
It takes the value of an expression and compares it to several literal constants.
The branch corresponding to the matching constant is taken.
Contrary to other programming languages, for safety reasons, control flow does
not continue from one case to the next. There can be a fallback or default
case called ``default`` which is taken if none of the literal constants matches.

.. code-block:: yul

    {
        let x := 0
        switch calldataload(4)
        case 0 {
            x := calldataload(0x24)
        }
        default {
            x := calldataload(0x44)
        }
        sstore(0, div(x, 2))
    }

The list of cases is not enclosed by curly braces, but the body of a
case does require them.

Loops
-----

Yul supports for-loops which consist of
a header containing an initializing part, a condition, a post-iteration
part and a body. The condition has to be an expression, while
the other three are blocks. If the initializing part
declares any variables at the top level, the scope of these variables extends to all other
parts of the loop.

The ``break`` and ``continue`` statements can be used in the body to exit the loop
or skip to the post-part, respectively.

The following example computes the sum of an area in memory.

.. code-block:: yul

    {
        let x := 0
        for { let i := 0 } lt(i, 0x100) { i := add(i, 0x20) } {
            x := add(x, mload(i))
        }
    }

For loops can also be used as a replacement for while loops:
Simply leave the initialization and post-iteration parts empty.

.. code-block:: yul

    {
        let x := 0
        let i := 0
        for { } lt(i, 0x100) { } {     // while(i < 0x100)
            x := add(x, mload(i))
            i := add(i, 0x20)
        }
    }

Function Declarations
---------------------

Yul allows the definition of functions. These should not be confused with functions
in Solidity since they are never part of an external interface of a contract and
are part of a namespace separate from the one for Solidity functions.

For the EVM, Yul functions take their
arguments (and a return PC) from the stack and also put the results onto the
stack. User-defined functions and built-in functions are called in exactly the same way.

Functions can be defined anywhere and are visible in the block they are
declared in. Inside a function, you cannot access local variables
defined outside of that function.

Functions declare parameters and return variables, similar to Solidity.
To return a value, you assign it to the return variable(s).

If you call a function that returns multiple values, you have to assign
them to multiple variables using ``a, b := f(x)`` or ``let a, b := f(x)``.

The ``leave`` statement can be used to exit the current function. It
works like the ``return`` statement in other languages just that it does
not take a value to return, it just exits the functions and the function
will return whatever values are currently assigned to the return variable(s).

Note that the EVM dialect has a built-in function called ``return`` that
quits the full execution context (internal message call) and not just
the current yul function.

The following example implements the power function by square-and-multiply.

.. code-block:: yul

    {
        function power(base, exponent) -> result {
            switch exponent
            case 0 { result := 1 }
            case 1 { result := base }
            default {
                result := power(mul(base, base), div(exponent, 2))
                switch mod(exponent, 2)
                    case 1 { result := mul(base, result) }
            }
        }
    }

Specification of Yul
====================

This chapter describes Yul code formally. Yul code is usually placed inside Yul objects,
which are explained in their own chapter.

.. code-block:: none

    Block = '{' Statement* '}'
    Statement =
        Block |
        FunctionDefinition |
        VariableDeclaration |
        Assignment |
        If |
        Expression |
        Switch |
        ForLoop |
        BreakContinue |
        Leave
    FunctionDefinition =
        'function' Identifier '(' TypedIdentifierList? ')'
        ( '->' TypedIdentifierList )? Block
    VariableDeclaration =
        'let' TypedIdentifierList ( ':=' Expression )?
    Assignment =
        IdentifierList ':=' Expression
    Expression =
        FunctionCall | Identifier | Literal
    If =
        'if' Expression Block
    Switch =
        'switch' Expression ( Case+ Default? | Default )
    Case =
        'case' Literal Block
    Default =
        'default' Block
    ForLoop =
        'for' Block Expression Block Block
    BreakContinue =
        'break' | 'continue'
    Leave = 'leave'
    FunctionCall =
        Identifier '(' ( Expression ( ',' Expression )* )? ')'
    Identifier = [a-zA-Z_$] [a-zA-Z_$0-9.]*
    IdentifierList = Identifier ( ',' Identifier)*
    TypeName = Identifier
    TypedIdentifierList = Identifier ( ':' TypeName )? ( ',' Identifier ( ':' TypeName )? )*
    Literal =
        (NumberLiteral | StringLiteral | TrueLiteral | FalseLiteral) ( ':' TypeName )?
    NumberLiteral = HexNumber | DecimalNumber
    StringLiteral = '"' ([^"\r\n\\] | '\\' .)* '"'
    TrueLiteral = 'true'
    FalseLiteral = 'false'
    HexNumber = '0x' [0-9a-fA-F]+
    DecimalNumber = [0-9]+


Restrictions on the Grammar
---------------------------

Apart from those directly imposed by the grammar, the following
restrictions apply:

Switches must have at least one case (including the default case).
All case values need to have the same type and distinct values.
If all possible values of the expression type are covered, a default case is
not allowed (i.e. a switch with a ``bool`` expression that has both a
true and a false case do not allow a default case).

Every expression evaluates to zero or more values. Identifiers and Literals
evaluate to exactly
one value and function calls evaluate to a number of values equal to the
number of return variables of the function called.

In variable declarations and assignments, the right-hand-side expression
(if present) has to evaluate to a number of values equal to the number of
variables on the left-hand-side.
This is the only situation where an expression evaluating
to more than one value is allowed.
The same variable name cannot occur more than once in the left-hand-side of
an assignment or variable declaration.

Expressions that are also statements (i.e. at the block level) have to
evaluate to zero values.

In all other situations, expressions have to evaluate to exactly one value.

A ``continue`` or ``break`` statement can only be used inside the body of a for-loop, as follows.
Consider the innermost loop that contains the statement.
The loop and the statement must be in the same function, or both must be at the top level.
The statement must be in the loop's body block;
it cannot be in the loop's initialization block or update block.
It is worth emphasizing that this restriction applies just
to the innermost loop that contains the ``continue`` or ``break`` statement:
this innermost loop, and therefore the ``continue`` or ``break`` statement,
may appear anywhere in an outer loop, possibly in an outer loop's initialization block or update block.
For example, the following is legal,
because the ``break`` occurs in the body block of the inner loop,
despite also occurring in the update block of the outer loop:

.. code-block:: yul

    for {} true { for {} true {} { break } }
    {
    }

The condition part of the for-loop has to evaluate to exactly one value.

The ``leave`` statement can only be used inside a function.

Functions cannot be defined anywhere inside for loop init blocks.

Literals cannot be larger than their type. The largest type defined is 256-bit wide.

During assignments and function calls, the types of the respective values have to match.
There is no implicit type conversion. Type conversion in general can only be achieved
if the dialect provides an appropriate built-in function that takes a value of one
type and returns a value of a different type.

Scoping Rules
-------------

Scopes in Yul are tied to Blocks (exceptions are functions and the for loop
as explained below) and all declarations
(``FunctionDefinition``, ``VariableDeclaration``)
introduce new identifiers into these scopes.

Identifiers are visible in
the block they are defined in (including all sub-nodes and sub-blocks):
Functions are visible in the whole block (even before their definitions) while
variables are only visible starting from the statement after the ``VariableDeclaration``.

In particular,
variables cannot be referenced in the right hand side of their own variable
declaration.
Functions can be referenced already before their declaration (if they are visible).

As an exception to the general scoping rule, the scope of the "init" part of the for-loop
(the first block) extends across all other parts of the for loop.
This means that variables (and functions) declared in the init part (but not inside a
block inside the init part) are visible in all other parts of the for-loop.

Identifiers declared in the other parts of the for loop respect the regular
syntactical scoping rules.

This means a for-loop of the form ``for { I... } C { P... } { B... }`` is equivalent
to ``{ I... for {} C { P... } { B... } }``.

The parameters and return parameters of functions are visible in the
function body and their names have to be distinct.

Inside functions, it is not possible to reference a variable that was declared
outside of that function.

Shadowing is disallowed, i.e. you cannot declare an identifier at a point
where another identifier with the same name is also visible, even if it is
not possible to reference it because it was declared outside the current function.

Formal Specification
--------------------

We formally specify Yul by providing an evaluation function E overloaded
on the various nodes of the AST. As builtin functions can have side effects,
E takes two state objects and the AST node and returns two new
state objects and a variable number of other values.
The two state objects are the global state object
(which in the context of the EVM is the memory, storage and state of the
blockchain) and the local state object (the state of local variables, i.e. a
segment of the stack in the EVM).

If the AST node is a statement, E returns the two state objects and a "mode",
which is used for the ``break``, ``continue`` and ``leave`` statements.
If the AST node is an expression, E returns the two state objects and
as many values as the expression evaluates to.


The exact nature of the global state is unspecified for this high level
description. The local state ``L`` is a mapping of identifiers ``i`` to values ``v``,
denoted as ``L[i] = v``.

For an identifier ``v``, let ``$v`` be the name of the identifier.

We will use a destructuring notation for the AST nodes.

.. code-block:: none

    E(G, L, <{St1, ..., Stn}>: Block) =
        let G1, L1, mode = E(G, L, St1, ..., Stn)
        let L2 be a restriction of L1 to the identifiers of L
        G1, L2, mode
    E(G, L, St1, ..., Stn: Statement) =
        if n is zero:
            G, L, regular
        else:
            let G1, L1, mode = E(G, L, St1)
            if mode is regular then
                E(G1, L1, St2, ..., Stn)
            otherwise
                G1, L1, mode
    E(G, L, FunctionDefinition) =
        G, L, regular
    E(G, L, <let var_1, ..., var_n := rhs>: VariableDeclaration) =
        E(G, L, <var_1, ..., var_n := rhs>: Assignment)
    E(G, L, <let var_1, ..., var_n>: VariableDeclaration) =
        let L1 be a copy of L where L1[$var_i] = 0 for i = 1, ..., n
        G, L1, regular
    E(G, L, <var_1, ..., var_n := rhs>: Assignment) =
        let G1, L1, v1, ..., vn = E(G, L, rhs)
        let L2 be a copy of L1 where L2[$var_i] = vi for i = 1, ..., n
        G, L2, regular
    E(G, L, <for { i1, ..., in } condition post body>: ForLoop) =
        if n >= 1:
            let G1, L, mode = E(G, L, i1, ..., in)
            // mode has to be regular or leave due to the syntactic restrictions
            if mode is leave then
                G1, L1 restricted to variables of L, leave
            otherwise
                let G2, L2, mode = E(G1, L1, for {} condition post body)
                G2, L2 restricted to variables of L, mode
        else:
            let G1, L1, v = E(G, L, condition)
            if v is false:
                G1, L1, regular
            else:
                let G2, L2, mode = E(G1, L, body)
                if mode is break:
                    G2, L2, regular
                otherwise if mode is leave:
                    G2, L2, leave
                else:
                    G3, L3, mode = E(G2, L2, post)
                    if mode is leave:
                        G2, L3, leave
                    otherwise
                        E(G3, L3, for {} condition post body)
    E(G, L, break: BreakContinue) =
        G, L, break
    E(G, L, continue: BreakContinue) =
        G, L, continue
    E(G, L, leave: Leave) =
        G, L, leave
    E(G, L, <if condition body>: If) =
        let G0, L0, v = E(G, L, condition)
        if v is true:
            E(G0, L0, body)
        else:
            G0, L0, regular
    E(G, L, <switch condition case l1:t1 st1 ... case ln:tn stn>: Switch) =
        E(G, L, switch condition case l1:t1 st1 ... case ln:tn stn default {})
    E(G, L, <switch condition case l1:t1 st1 ... case ln:tn stn default st'>: Switch) =
        let G0, L0, v = E(G, L, condition)
        // i = 1 .. n
        // Evaluate literals, context doesn't matter
        let _, _, v1 = E(G0, L0, l1)
        ...
        let _, _, vn = E(G0, L0, ln)
        if there exists smallest i such that vi = v:
            E(G0, L0, sti)
        else:
            E(G0, L0, st')

    E(G, L, <name>: Identifier) =
        G, L, L[$name]
    E(G, L, <fname(arg1, ..., argn)>: FunctionCall) =
        G1, L1, vn = E(G, L, argn)
        ...
        G(n-1), L(n-1), v2 = E(G(n-2), L(n-2), arg2)
        Gn, Ln, v1 = E(G(n-1), L(n-1), arg1)
        Let <function fname (param1, ..., paramn) -> ret1, ..., retm block>
        be the function of name $fname visible at the point of the call.
        Let L' be a new local state such that
        L'[$parami] = vi and L'[$reti] = 0 for all i.
        Let G'', L'', mode = E(Gn, L', block)
        G'', Ln, L''[$ret1], ..., L''[$retm]
    E(G, L, l: StringLiteral) = G, L, str(l),
        where str is the string evaluation function,
        which for the EVM dialect is defined in the section 'Literals' above
    E(G, L, n: HexNumber) = G, L, hex(n)
        where hex is the hexadecimal evaluation function,
        which turns a sequence of hexadecimal digits into their big endian value
    E(G, L, n: DecimalNumber) = G, L, dec(n),
        where dec is the decimal evaluation function,
        which turns a sequence of decimal digits into their big endian value

.. _opcodes:

EVM Dialect
-----------

The default dialect of Yul currently is the EVM dialect for the currently selected version of the EVM.
with a version of the EVM. The only type available in this dialect
is ``u256``, the 256-bit native type of the Ethereum Virtual Machine.
Since it is the default type of this dialect, it can be omitted.

The following table lists all builtin functions
(depending on the EVM version) and provides a short description of the
semantics of the function / opcode.
This document does not want to be a full description of the Ethereum virtual machine.
Please refer to a different document if you are interested in the precise semantics.

Opcodes marked with ``-`` do not return a result and all others return exactly one value.
Opcodes marked with ``F``, ``H``, ``B``, ``C``, ``I`` and ``L`` are present since Frontier, Homestead,
Byzantium, Constantinople, Istanbul or London respectively.

In the following, ``mem[a...b)`` signifies the bytes of memory starting at position ``a`` up to
but not including position ``b`` and ``storage[p]`` signifies the storage contents at slot ``p``.

Since Yul manages local variables and control-flow,
opcodes that interfere with these features are not available. This includes
the ``dup`` and ``swap`` instructions as well as ``jump`` instructions, labels and the ``push`` instructions.

+-------------------------+-----+---+-----------------------------------------------------------------+
| Instruction             |     |   | Explanation                                                     |
+=========================+=====+===+=================================================================+
| stop()                  | `-` | F | stop execution, identical to return(0, 0)                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| add(x, y)               |     | F | x + y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sub(x, y)               |     | F | x - y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mul(x, y)               |     | F | x * y                                                           |
+-------------------------+-----+---+-----------------------------------------------------------------+
| div(x, y)               |     | F | x / y or 0 if y == 0                                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sdiv(x, y)              |     | F | x / y, for signed numbers in two's complement, 0 if y == 0      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mod(x, y)               |     | F | x % y, 0 if y == 0                                              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| smod(x, y)              |     | F | x % y, for signed numbers in two's complement, 0 if y == 0      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| exp(x, y)               |     | F | x to the power of y                                             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| not(x)                  |     | F | bitwise "not" of x (every bit of x is negated)                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| lt(x, y)                |     | F | 1 if x < y, 0 otherwise                                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gt(x, y)                |     | F | 1 if x > y, 0 otherwise                                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| slt(x, y)               |     | F | 1 if x < y, 0 otherwise, for signed numbers in two's complement |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sgt(x, y)               |     | F | 1 if x > y, 0 otherwise, for signed numbers in two's complement |
+-------------------------+-----+---+-----------------------------------------------------------------+
| eq(x, y)                |     | F | 1 if x == y, 0 otherwise                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| iszero(x)               |     | F | 1 if x == 0, 0 otherwise                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| and(x, y)               |     | F | bitwise "and" of x and y                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| or(x, y)                |     | F | bitwise "or" of x and y                                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| xor(x, y)               |     | F | bitwise "xor" of x and y                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| byte(n, x)              |     | F | nth byte of x, where the most significant byte is the 0th byte  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| shl(x, y)               |     | C | logical shift left y by x bits                                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| shr(x, y)               |     | C | logical shift right y by x bits                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sar(x, y)               |     | C | signed arithmetic shift right y by x bits                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| addmod(x, y, m)         |     | F | (x + y) % m with arbitrary precision arithmetic, 0 if m == 0    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mulmod(x, y, m)         |     | F | (x * y) % m with arbitrary precision arithmetic, 0 if m == 0    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| signextend(i, x)        |     | F | sign extend from (i*8+7)th bit counting from least significant  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| keccak256(p, n)         |     | F | keccak(mem[p...(p+n)))                                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| pc()                    |     | F | current position in code                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| pop(x)                  | `-` | F | discard value x                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mload(p)                |     | F | mem[p...(p+32))                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mstore(p, v)            | `-` | F | mem[p...(p+32)) := v                                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| mstore8(p, v)           | `-` | F | mem[p] := v & 0xff (only modifies a single byte)                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sload(p)                |     | F | storage[p]                                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| sstore(p, v)            | `-` | F | storage[p] := v                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| msize()                 |     | F | size of memory, i.e. largest accessed memory index              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gas()                   |     | F | gas still available to execution                                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| address()               |     | F | address of the current contract / execution context             |
+-------------------------+-----+---+-----------------------------------------------------------------+
| balance(a)              |     | F | wei balance at address a                                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| selfbalance()           |     | I | equivalent to balance(address()), but cheaper                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| caller()                |     | F | call sender (excluding ``delegatecall``)                        |
+-------------------------+-----+---+-----------------------------------------------------------------+
| callvalue()             |     | F | wei sent together with the current call                         |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldataload(p)         |     | F | call data starting from position p (32 bytes)                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldatasize()          |     | F | size of call data in bytes                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| calldatacopy(t, f, s)   | `-` | F | copy s bytes from calldata at position f to mem at position t   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| codesize()              |     | F | size of the code of the current contract / execution context    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| codecopy(t, f, s)       | `-` | F | copy s bytes from code at position f to mem at position t       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodesize(a)          |     | F | size of the code at address a                                   |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodecopy(a, t, f, s) | `-` | F | like codecopy(t, f, s) but take code at address a               |
+-------------------------+-----+---+-----------------------------------------------------------------+
| returndatasize()        |     | B | size of the last returndata                                     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| returndatacopy(t, f, s) | `-` | B | copy s bytes from returndata at position f to mem at position t |
+-------------------------+-----+---+-----------------------------------------------------------------+
| extcodehash(a)          |     | C | code hash of address a                                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| create(v, p, n)         |     | F | create new contract with code mem[p...(p+n)) and send v wei     |
|                         |     |   | and return the new address; returns 0 on error                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| create2(v, p, n, s)     |     | C | create new contract with code mem[p...(p+n)) at address         |
|                         |     |   | keccak256(0xff . this . s . keccak256(mem[p...(p+n)))           |
|                         |     |   | and send v wei and return the new address, where ``0xff`` is a  |
|                         |     |   | 1 byte value, ``this`` is the current contract's address        |
|                         |     |   | as a 20 byte value and ``s`` is a big-endian 256-bit value;     |
|                         |     |   | returns 0 on error                                              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| call(g, a, v, in,       |     | F | call contract at address a with input mem[in...(in+insize))     |
| insize, out, outsize)   |     |   | providing g gas and v wei and output area                       |
|                         |     |   | mem[out...(out+outsize)) returning 0 on error (eg. out of gas)  |
|                         |     |   | and 1 on success                                                |
|                         |     |   | :ref:`See more <yul-call-return-area>`                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| callcode(g, a, v, in,   |     | F | identical to ``call`` but only use the code from a and stay     |
| insize, out, outsize)   |     |   | in the context of the current contract otherwise                |
|                         |     |   | :ref:`See more <yul-call-return-area>`                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| delegatecall(g, a, in,  |     | H | identical to ``callcode`` but also keep ``caller``              |
| insize, out, outsize)   |     |   | and ``callvalue``                                               |
|                         |     |   | :ref:`See more <yul-call-return-area>`                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| staticcall(g, a, in,    |     | B | identical to ``call(g, a, 0, in, insize, out, outsize)`` but do |
| insize, out, outsize)   |     |   | not allow state modifications                                   |
|                         |     |   | :ref:`See more <yul-call-return-area>`                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| return(p, s)            | `-` | F | end execution, return data mem[p...(p+s))                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| revert(p, s)            | `-` | B | end execution, revert state changes, return data mem[p...(p+s)) |
+-------------------------+-----+---+-----------------------------------------------------------------+
| selfdestruct(a)         | `-` | F | end execution, destroy current contract and send funds to a     |
+-------------------------+-----+---+-----------------------------------------------------------------+
| invalid()               | `-` | F | end execution with invalid instruction                          |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log0(p, s)              | `-` | F | log without topics and data mem[p...(p+s))                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log1(p, s, t1)          | `-` | F | log with topic t1 and data mem[p...(p+s))                       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log2(p, s, t1, t2)      | `-` | F | log with topics t1, t2 and data mem[p...(p+s))                  |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log3(p, s, t1, t2, t3)  | `-` | F | log with topics t1, t2, t3 and data mem[p...(p+s))              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| log4(p, s, t1, t2, t3,  | `-` | F | log with topics t1, t2, t3, t4 and data mem[p...(p+s))          |
| t4)                     |     |   |                                                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| chainid()               |     | I | ID of the executing chain (EIP-1344)                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| basefee()               |     | L | current block's base fee (EIP-3198 and EIP-1559)                |
+-------------------------+-----+---+-----------------------------------------------------------------+
| origin()                |     | F | transaction sender                                              |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gasprice()              |     | F | gas price of the transaction                                    |
+-------------------------+-----+---+-----------------------------------------------------------------+
| blockhash(b)            |     | F | hash of block nr b - only for last 256 blocks excluding current |
+-------------------------+-----+---+-----------------------------------------------------------------+
| coinbase()              |     | F | current mining beneficiary                                      |
+-------------------------+-----+---+-----------------------------------------------------------------+
| timestamp()             |     | F | timestamp of the current block in seconds since the epoch       |
+-------------------------+-----+---+-----------------------------------------------------------------+
| number()                |     | F | current block number                                            |
+-------------------------+-----+---+-----------------------------------------------------------------+
| difficulty()            |     | F | difficulty of the current block                                 |
+-------------------------+-----+---+-----------------------------------------------------------------+
| gaslimit()              |     | F | block gas limit of the current block                            |
+-------------------------+-----+---+-----------------------------------------------------------------+

.. _yul-call-return-area:

.. note::
  The ``call*`` instructions use the ``out`` and ``outsize`` parameters to define an area in memory where
  the return or failure data is placed. This area is written to depending on how many bytes the called contract returns.
  If it returns more data, only the first ``outsize`` bytes are written. You can access the rest of the data
  using the ``returndatacopy`` opcode. If it returns less data, then the remaining bytes are not touched at all.
  You need to use the ``returndatasize`` opcode to check which part of this memory area contains the return data.
  The remaining bytes will retain their values as of before the call.


In some internal dialects, there are additional functions:

datasize, dataoffset, datacopy
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The functions ``datasize(x)``, ``dataoffset(x)`` and ``datacopy(t, f, l)``
are used to access other parts of a Yul object.

``datasize`` and ``dataoffset`` can only take string literals (the names of other objects)
as arguments and return the size and offset in the data area, respectively.
For the EVM, the ``datacopy`` function is equivalent to ``codecopy``.


setimmutable, loadimmutable
^^^^^^^^^^^^^^^^^^^^^^^^^^^

The functions ``setimmutable(offset, "name", value)`` and ``loadimmutable("name")`` are
used for the immutable mechanism in Solidity and do not nicely map to pure Yul.
The call to ``setimmutable(offset, "name", value)`` assumes that the runtime code of the contract
containing the given named immutable was copied to memory at offset ``offset`` and will write ``value`` to all
positions in memory (relative to ``offset``) that contain the placeholder that was generated for calls
to ``loadimmutable("name")`` in the runtime code.


linkersymbol
^^^^^^^^^^^^
The function ``linkersymbol("library_id")`` is a placeholder for an address literal to be substituted
by the linker.
Its first and only argument must be a string literal and uniquely represents the address to be inserted.
Identifiers can be arbitrary but when the compiler produces Yul code from Solidity sources,
it uses a library name qualified with the name of the source unit that defines that library.
To link the code with a particular library address, the same identifier must be provided to the
``--libraries`` option on the command line.

For example this code

.. code-block:: yul

    let a := linkersymbol("file.sol:Math")

is equivalent to

.. code-block:: yul

    let a := 0x1234567890123456789012345678901234567890

when the linker is invoked with ``--libraries "file.sol:Math=0x1234567890123456789012345678901234567890``
option.

See :ref:`Using the Commandline Compiler <commandline-compiler>` for details about the Solidity linker.

memoryguard
^^^^^^^^^^^

This function is available in the EVM dialect with objects. The caller of
``let ptr := memoryguard(size)`` (where ``size`` has to be a literal number)
promises that they only use memory in either the range ``[0, size)`` or the
unbounded range starting at ``ptr``.

Since the presence of a ``memoryguard`` call indicates that all memory access
adheres to this restriction, it allows the optimizer to perform additional
optimization steps, for example the stack limit evader, which attempts to move
stack variables that would otherwise be unreachable to memory.

The Yul optimizer promises to only use the memory range ``[size, ptr)`` for its purposes.
If the optimizer does not need to reserve any memory, it holds that ``ptr == size``.

``memoryguard`` can be called multiple times, but needs to have the same literal as argument
within one Yul subobject. If at least one ``memoryguard`` call is found in a subobject,
the additional optimiser steps will be run on it.


.. _yul-verbatim:

verbatim
^^^^^^^^

The set of ``verbatim...`` builtin functions lets you create bytecode for opcodes
that are not known to the Yul compiler. It also allows you to create
bytecode sequences that will not be modified by the optimizer.

The functions are ``verbatim_<n>i_<m>o("<data>", ...)``, where

- ``n`` is a decimal between 0 and 99 that specifies the number of input stack slots / variables
- ``m`` is a decimal between 0 and 99 that specifies the number of output stack slots / variables
- ``data`` is a string literal that contains the sequence of bytes

If you for example want to define a function that multiplies the input
by two, without the optimizer touching the constant two, you can use

.. code-block:: yul

    let x := calldataload(0)
    let double := verbatim_1i_1o(hex"600202", x)

This code will result in a ``dup1`` opcode to retrieve ``x``
(the optimizer might directly re-use result of the
``calldataload`` opcode, though)
directly followed by ``600202``. The code is assumed to
consume the copied value of ``x`` and produce the result
on the top of the stack. The compiler then generates code
to allocate a stack slot for ``double`` and store the result there.

As with all opcodes, the arguments are arranged on the stack
with the leftmost argument on the top, while the return values
are assumed to be laid out such that the rightmost variable is
at the top of the stack.

Since ``verbatim`` can be used to generate arbitrary opcodes
or even opcodes unknown to the Solidity compiler, care has to be taken
when using ``verbatim`` together with the optimizer. Even when the
optimizer is switched off, the code generator has to determine
the stack layout, which means that e.g. using ``verbatim`` to modify
the stack height can lead to undefined behaviour.

The following is a non-exhaustive list of restrictions on
verbatim bytecode that are not checked by
the compiler. Violations of these restrictions can result in
undefined behaviour.

- Control-flow should not jump into or out of verbatim blocks,
  but it can jump within the same verbatim block.
- Stack contents apart from the input and output parameters
  should not be accessed.
- The stack height difference should be exactly ``m - n``
  (output slots minus input slots).
- Verbatim bytecode cannot make any assumptions about the
  surrounding bytecode. All required parameters have to be
  passed in as stack variables.

The optimizer does not analyze verbatim bytecode and always
assumes that it modifies all aspects of state and thus can only
do very few optimizations across ``verbatim`` function calls.

The optimizer treats verbatim bytecode as an opaque block of code.
It will not split it but might move, duplicate
or combine it with identical verbatim bytecode blocks.
If a verbatim bytecode block is unreachable by the control-flow,
it can be removed.


.. warning::

    During discussions about whether or not EVM improvements
    might break existing smart contracts, features inside ``verbatim``
    cannot receive the same consideration as those used by the Solidity
    compiler itself.

.. note::

    To avoid confusion, all identifiers starting with the string ``verbatim`` are reserved
    and cannot be used for user-defined identifiers.

.. _yul-object:

Specification of Yul Object
===========================

Yul objects are used to group named code and data sections.
The functions ``datasize``, ``dataoffset`` and ``datacopy``
can be used to access these sections from within code.
Hex strings can be used to specify data in hex encoding,
regular strings in native encoding. For code,
``datacopy`` will access its assembled binary representation.

.. code-block:: none

    Object = 'object' StringLiteral '{' Code ( Object | Data )* '}'
    Code = 'code' Block
    Data = 'data' StringLiteral ( HexLiteral | StringLiteral )
    HexLiteral = 'hex' ('"' ([0-9a-fA-F]{2})* '"' | '\'' ([0-9a-fA-F]{2})* '\'')
    StringLiteral = '"' ([^"\r\n\\] | '\\' .)* '"'

Above, ``Block`` refers to ``Block`` in the Yul code grammar explained in the previous chapter.

.. note::

    An object with a name that ends in ``_deployed`` is treated as deployed code by the Yul optimizer.
    The only consequence of this is a different gas cost heuristic in the optimizer.

.. note::

    Data objects or sub-objects whose names contain a ``.`` can be defined
    but it is not possible to access them through ``datasize``,
    ``dataoffset`` or ``datacopy`` because ``.`` is used as a separator
    to access objects inside another object.

.. note::

    The data object called ``".metadata"`` has a special meaning:
    It cannot be accessed from code and is always appended to the very end of the
    bytecode, regardless of its position in the object.

    Other data objects with special significance might be added in the
    future, but their names will always start with a ``.``.


An example Yul Object is shown below:

.. code-block:: yul

    // A contract consists of a single object with sub-objects representing
    // the code to be deployed or other contracts it can create.
    // The single "code" node is the executable code of the object.
    // Every (other) named object or data section is serialized and
    // made accessible to the special built-in functions datacopy / dataoffset / datasize
    // The current object, sub-objects and data items inside the current object
    // are in scope.
    object "Contract1" {
        // This is the constructor code of the contract.
        code {
            function allocate(size) -> ptr {
                ptr := mload(0x40)
                if iszero(ptr) { ptr := 0x60 }
                mstore(0x40, add(ptr, size))
            }

            // first create "Contract2"
            let size := datasize("Contract2")
            let offset := allocate(size)
            // This will turn into codecopy for EVM
            datacopy(offset, dataoffset("Contract2"), size)
            // constructor parameter is a single number 0x1234
            mstore(add(offset, size), 0x1234)
            pop(create(offset, add(size, 32), 0))

            // now return the runtime object (the currently
            // executing code is the constructor code)
            size := datasize("Contract1_deployed")
            offset := allocate(size)
            // This will turn into a memory->memory copy for Ewasm and
            // a codecopy for EVM
            datacopy(offset, dataoffset("Contract1_deployed"), size)
            return(offset, size)
        }

        data "Table2" hex"4123"

        object "Contract1_deployed" {
            code {
                function allocate(size) -> ptr {
                    ptr := mload(0x40)
                    if iszero(ptr) { ptr := 0x60 }
                    mstore(0x40, add(ptr, size))
                }

                // runtime code

                mstore(0, "Hello, World!")
                return(0, 0x20)
            }
        }

        // Embedded object. Use case is that the outside is a factory contract,
        // and Contract2 is the code to be created by the factory
        object "Contract2" {
            code {
                // code here ...
            }

            object "Contract2_deployed" {
                code {
                    // code here ...
                }
            }

            data "Table1" hex"4123"
        }
    }

Yul Optimizer
=============

The Yul optimizer operates on Yul code and uses the same language for input, output and
intermediate states. This allows for easy debugging and verification of the optimizer.

Please refer to the general :ref:`optimizer documentation <optimizer>`
for more details about the different optimization stages and how to use the optimizer.

If you want to use Solidity in stand-alone Yul mode, you activate the optimizer using ``--optimize``
and optionally specify the :ref:`expected number of contract executions <optimizer-parameter-runs>` with
``--optimize-runs``:

.. code-block:: sh

    solc --strict-assembly --optimize --optimize-runs 200

In Solidity mode, the Yul optimizer is activated together with the regular optimizer.

Optimization Step Sequence
--------------------------

By default the Yul optimizer applies its predefined sequence of optimization steps to the generated assembly.
You can override this sequence and supply your own using the ``--yul-optimizations`` option:

.. code-block:: sh

    solc --optimize --ir-optimized --yul-optimizations 'dhfoD[xarrscLMcCTU]uljmul'

The order of steps is significant and affects the quality of the output.
Moreover, applying a step may uncover new optimization opportunities for others that were already
applied so repeating steps is often beneficial.
By enclosing part of the sequence in square brackets (``[]``) you tell the optimizer to repeatedly
apply that part until it no longer improves the size of the resulting assembly.
You can use brackets multiple times in a single sequence but they cannot be nested.

The following optimization steps are available:

============ ===============================
Abbreviation Full name
============ ===============================
``f``        ``BlockFlattener``
``l``        ``CircularReferencesPruner``
``c``        ``CommonSubexpressionEliminator``
``C``        ``ConditionalSimplifier``
``U``        ``ConditionalUnsimplifier``
``n``        ``ControlFlowSimplifier``
``D``        ``DeadCodeEliminator``
``v``        ``EquivalentFunctionCombiner``
``e``        ``ExpressionInliner``
``j``        ``ExpressionJoiner``
``s``        ``ExpressionSimplifier``
``x``        ``ExpressionSplitter``
``I``        ``ForLoopConditionIntoBody``
``O``        ``ForLoopConditionOutOfBody``
``o``        ``ForLoopInitRewriter``
``i``        ``FullInliner``
``g``        ``FunctionGrouper``
``h``        ``FunctionHoister``
``F``        ``FunctionSpecializer``
``T``        ``LiteralRematerialiser``
``L``        ``LoadResolver``
``M``        ``LoopInvariantCodeMotion``
``r``        ``RedundantAssignEliminator``
``R``        ``ReasoningBasedSimplifier`` - highly experimental
``m``        ``Rematerialiser``
``V``        ``SSAReverser``
``a``        ``SSATransform``
``t``        ``StructuralSimplifier``
``u``        ``UnusedPruner``
``p``        ``UnusedFunctionParameterPruner``
``d``        ``VarDeclInitializer``
============ ===============================

Some steps depend on properties ensured by ``BlockFlattener``, ``FunctionGrouper``, ``ForLoopInitRewriter``.
For this reason the Yul optimizer always applies them before applying any steps supplied by the user.

The ReasoningBasedSimplifier is an optimizer step that is currently not enabled
in the default set of steps. It uses an SMT solver to simplify arithmetic expressions
and boolean conditions. It has not received thorough testing or validation yet and can produce
non-reproducible results, so please use with care!

.. _erc20yul:

Complete ERC20 Example
======================

.. code-block:: yul

    object "Token" {
        code {
            // Store the creator in slot zero.
            sstore(0, caller())

            // Deploy the contract
            datacopy(0, dataoffset("runtime"), datasize("runtime"))
            return(0, datasize("runtime"))
        }
        object "runtime" {
            code {
                // Protection against sending Ether
                require(iszero(callvalue()))

                // Dispatcher
                switch selector()
                case 0x70a08231 /* "balanceOf(address)" */ {
                    returnUint(balanceOf(decodeAsAddress(0)))
                }
                case 0x18160ddd /* "totalSupply()" */ {
                    returnUint(totalSupply())
                }
                case 0xa9059cbb /* "transfer(address,uint256)" */ {
                    transfer(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                case 0x23b872dd /* "transferFrom(address,address,uint256)" */ {
                    transferFrom(decodeAsAddress(0), decodeAsAddress(1), decodeAsUint(2))
                    returnTrue()
                }
                case 0x095ea7b3 /* "approve(address,uint256)" */ {
                    approve(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                case 0xdd62ed3e /* "allowance(address,address)" */ {
                    returnUint(allowance(decodeAsAddress(0), decodeAsAddress(1)))
                }
                case 0x40c10f19 /* "mint(address,uint256)" */ {
                    mint(decodeAsAddress(0), decodeAsUint(1))
                    returnTrue()
                }
                default {
                    revert(0, 0)
                }

                function mint(account, amount) {
                    require(calledByOwner())

                    mintTokens(amount)
                    addToBalance(account, amount)
                    emitTransfer(0, account, amount)
                }
                function transfer(to, amount) {
                    executeTransfer(caller(), to, amount)
                }
                function approve(spender, amount) {
                    revertIfZeroAddress(spender)
                    setAllowance(caller(), spender, amount)
                    emitApproval(caller(), spender, amount)
                }
                function transferFrom(from, to, amount) {
                    decreaseAllowanceBy(from, caller(), amount)
                    executeTransfer(from, to, amount)
                }

                function executeTransfer(from, to, amount) {
                    revertIfZeroAddress(to)
                    deductFromBalance(from, amount)
                    addToBalance(to, amount)
                    emitTransfer(from, to, amount)
                }


                /* ---------- calldata decoding functions ----------- */
                function selector() -> s {
                    s := div(calldataload(0), 0x100000000000000000000000000000000000000000000000000000000)
                }

                function decodeAsAddress(offset) -> v {
                    v := decodeAsUint(offset)
                    if iszero(iszero(and(v, not(0xffffffffffffffffffffffffffffffffffffffff)))) {
                        revert(0, 0)
                    }
                }
                function decodeAsUint(offset) -> v {
                    let pos := add(4, mul(offset, 0x20))
                    if lt(calldatasize(), add(pos, 0x20)) {
                        revert(0, 0)
                    }
                    v := calldataload(pos)
                }
                /* ---------- calldata encoding functions ---------- */
                function returnUint(v) {
                    mstore(0, v)
                    return(0, 0x20)
                }
                function returnTrue() {
                    returnUint(1)
                }

                /* -------- events ---------- */
                function emitTransfer(from, to, amount) {
                    let signatureHash := 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
                    emitEvent(signatureHash, from, to, amount)
                }
                function emitApproval(from, spender, amount) {
                    let signatureHash := 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925
                    emitEvent(signatureHash, from, spender, amount)
                }
                function emitEvent(signatureHash, indexed1, indexed2, nonIndexed) {
                    mstore(0, nonIndexed)
                    log3(0, 0x20, signatureHash, indexed1, indexed2)
                }

                /* -------- storage layout ---------- */
                function ownerPos() -> p { p := 0 }
                function totalSupplyPos() -> p { p := 1 }
                function accountToStorageOffset(account) -> offset {
                    offset := add(0x1000, account)
                }
                function allowanceStorageOffset(account, spender) -> offset {
                    offset := accountToStorageOffset(account)
                    mstore(0, offset)
                    mstore(0x20, spender)
                    offset := keccak256(0, 0x40)
                }

                /* -------- storage access ---------- */
                function owner() -> o {
                    o := sload(ownerPos())
                }
                function totalSupply() -> supply {
                    supply := sload(totalSupplyPos())
                }
                function mintTokens(amount) {
                    sstore(totalSupplyPos(), safeAdd(totalSupply(), amount))
                }
                function balanceOf(account) -> bal {
                    bal := sload(accountToStorageOffset(account))
                }
                function addToBalance(account, amount) {
                    let offset := accountToStorageOffset(account)
                    sstore(offset, safeAdd(sload(offset), amount))
                }
                function deductFromBalance(account, amount) {
                    let offset := accountToStorageOffset(account)
                    let bal := sload(offset)
                    require(lte(amount, bal))
                    sstore(offset, sub(bal, amount))
                }
                function allowance(account, spender) -> amount {
                    amount := sload(allowanceStorageOffset(account, spender))
                }
                function setAllowance(account, spender, amount) {
                    sstore(allowanceStorageOffset(account, spender), amount)
                }
                function decreaseAllowanceBy(account, spender, amount) {
                    let offset := allowanceStorageOffset(account, spender)
                    let currentAllowance := sload(offset)
                    require(lte(amount, currentAllowance))
                    sstore(offset, sub(currentAllowance, amount))
                }

                /* ---------- utility functions ---------- */
                function lte(a, b) -> r {
                    r := iszero(gt(a, b))
                }
                function gte(a, b) -> r {
                    r := iszero(lt(a, b))
                }
                function safeAdd(a, b) -> r {
                    r := add(a, b)
                    if or(lt(r, a), lt(r, b)) { revert(0, 0) }
                }
                function calledByOwner() -> cbo {
                    cbo := eq(owner(), caller())
                }
                function revertIfZeroAddress(addr) {
                    require(addr)
                }
                function require(condition) {
                    if iszero(condition) { revert(0, 0) }
                }
            }
        }
    }
>>>>>>> abaa5c0eb321aab4cd09617598696172378a4b83
