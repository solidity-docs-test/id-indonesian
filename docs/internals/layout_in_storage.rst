<<<<<<< HEAD
.. index:: storage, state variable, mapping

************************************
Layout Variabel State dalam Storage
************************************

.. _storage-inplace-encoding:

Variabel state kontrak disimpan dalam penyimpanan dengan cara yang ringkas
sehingga beberapa nilai terkadang menggunakan slot penyimpanan yang sama.
Kecuali untuk dynamically-sized array dan mapping (lihat di bawah), data disimpan
secara berurutan, item setelah item, dimulai dengan variabel state pertama,
yang disimpan di slot ``0``. Untuk setiap variabel,
ukuran dalam byte ditentukan menurut jenisnya.
Beberapa item bersebelahan yang membutuhkan kurang dari 32 byte dikemas menjadi satu
slot penyimpanan jika memungkinkan, sesuai dengan aturan berikut:

- Item pertama dalam slot penyimpanan disimpan dengan urutan yang lebih rendah.
- Tipe nilai hanya menggunakan byte sebanyak yang diperlukan untuk menyimpannya.
- Jika jenis nilai tidak sesuai dengan bagian yang tersisa dari slot penyimpanan, maka akan disimpan di slot penyimpanan berikutnya.
- Struct dan data array selalu memulai slot baru dan itemnya dikemas dengan ketat sesuai dengan aturan ini.
- Item yang mengikuti data struct atau array selalu memulai slot penyimpanan baru.

Untuk kontrak yang menggunakan inheritance, urutan variabel state ditentukan oleh urutan kontrak
C3-linearized yang dimulai dengan kontrak paling dasar. Jika diizinkan
oleh aturan di atas, variabel state dari kontrak yang berbeda berbagi slot penyimpanan yang sama.

Elemen struct dan array disimpan setelah satu sama lain, seolah-olah mereka diberikan
sebagai nilai individual.

.. warning::
    Saat menggunakan elemen yang lebih kecil dari 32 byte, penggunaan gas kontrak Anda mungkin lebih tinggi.
    Ini karena EVM beroperasi pada 32 byte sekaligus. Oleh karena itu, jika elemennya lebih kecil
    dari itu, EVM harus menggunakan lebih banyak operasi untuk mengurangi ukuran elemen dari 32
    byte ke ukuran yang diinginkan.

    Mungkin bermanfaat untuk menggunakan tipe ukuran yang diperkecil jika Anda berurusan dengan nilai penyimpanan
    karena kompiler akan mengemas beberapa elemen ke dalam satu slot penyimpanan, dan dengan demikian, menggabungkan
    beberapa pembacaan atau penulisan ke dalam satu operasi.
    Jika Anda tidak membaca atau menulis semua nilai dalam slot pada saat yang sama, ini dapat
    memiliki efek sebaliknya, meskipun: Ketika satu nilai ditulis ke slot penyimpanan
    multi-nilai, slot penyimpanan harus dibaca terlebih dahulu dan kemudian
    digabungkan dengan nilai baru sehingga data lain di slot yang sama tidak dihancurkan.

    Ketika berhadapan dengan argumen fungsi atau nilai
    memori, tidak ada manfaat yang melekat karena kompiler tidak mengemas nilai-nilai ini.

    Terakhir, agar EVM dapat mengoptimalkannya, pastikan Anda mencoba mengurutkan
    variabel penyimpanan dan anggota ``struct`` sedemikian rupa sehingga dapat dikemas dengan rapat. Misalnya,
    mendeklarasikan variabel storage Anda dalam urutan ``uint128, uint128, uint256`` alih-alih
    ``uint128, uint256, uint128``, karena yang pertama hanya akan menggunakan dua slot penyimpanan sedangkan
    yang terakhir akan menggunakan tiga.

.. note::
     Tata letak variabel state dalam penyimpanan dianggap sebagai bagian dari
     antarmuka eksternal Solidity karena fakta bahwa pointer penyimpanan dapat
     diteruskan ke library. Ini berarti bahwa setiap perubahan pada aturan yang
     diuraikan dalam bagian ini dianggap sebagai perubahan bahasa yang melanggar
     dan karena sifatnya yang kritis harus dipertimbangkan dengan sangat hati-hati
     sebelum dieksekusi.


Mapping dan Array Dinamis
===========================

.. _storage-hashed-encoding:

Karena ukurannya yang tidak dapat diprediksi, mapping dan tipe array dynamically-sized tidak dapat
disimpan "di antara" variabel state sebelum dan sesudahnya.
Sebaliknya, mereka dianggap hanya menempati 32 byte sehubungan dengan
:ref:`rules di atas <storage-inplace-encoding>` dan elemen yang dikandungnya
disimpan mulai dari slot penyimpanan berbeda yang dihitung menggunakan hash Keccak-256 .

Asumsikan lokasi penyimpanan mapping atau array akhirnya menjadi slot ``p``
setelah menerapkan :ref:`aturan storage layout <storage-inplace-encoding>`.
Untuk array dinamis,
slot ini menyimpan jumlah elemen dalam array (array byte dan
string adalah pengecualian, lihat :ref:`di bawah <byte-dan-string>`).
Untuk mapping, slot tetap kosong, tetapi masih diperlukan untuk memastikan bahwa meskipun ada
dua mapping yang bersebelahan, kontennya berakhir di lokasi penyimpanan yang berbeda.

Data array terletak mulai dari ``keccak256(p)`` dan ditata dengan cara yang sama seperti
data array berukuran statis akan: Satu demi satu elemen, berpotensi berbagi
slot penyimpanan jika elemen tidak lebih dari 16 byte. Array dinamis dari array dinamis menerapkan
aturan ini secara rekursif. Lokasi elemen ``x[i][j]``, dengan tipe ``x`` adalah ``uint24[][]``, adalah
dihitung sebagai berikut (sekali lagi, dengan asumsi ``x`` sendiri disimpan di slot ``p``):
Slotnya adalah ``keccak256(keccak256(p) + i) + floor(j / floor(256 / 24))`` dan
elemen dapat diperoleh dari data slot ``v`` menggunakan ``(v >> ((j % floor(256 / 24)) * 24)) & type(uint24).max``.

Nilai yang sesuai dengan kunci pemetaan ``k`` terletak di ``keccak256(h(k) . p)``
di mana ``.`` adalah rangkaian dan ``h`` adalah fungsi yang diterapkan ke kunci tergantung pada jenisnya:

- untuk tipe nilai, ``h`` memasukkan nilai ke 32 byte dengan cara yang sama seperti saat menyimpan nilai dalam memori.
- untuk string dan array byte, ``h`` menghitung hash ``keccak256`` dari data yang tidak diisi.

Jika nila mapping adalah
tipe non-value, slot yang dihitung menandai awal dari data. Jika nilainya bertipe struct,
misalnya, Anda harus menambahkan offset yang sesuai dengan anggota struct untuk menjangkau anggota.

Sebagai contoh, perhatikan kontrak berikut:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;


    contract C {
        struct S { uint16 a; uint16 b; uint256 c; }
        uint x;
        mapping(uint => mapping(uint => S)) data;
    }

Mari kita hitung lokasi penyimpanan ``data[4][9].c``.
Posisi mapping itu sendiri adalah ``1`` (variabel ``x`` dengan 32 byte mendahuluinya).
Ini berarti ``data[4]`` disimpan pada ``keccak256(uint256(4) . uint256(1))``. Tipe dari ``data[4]`` sekali
lagi adalah sebuah mapping dan data untuk ``data[4][9]`` dimulai dari slot
``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1)))``.
Slot offset anggota ``c`` di dalam struct ``S`` adalah ``1`` karena ``a`` dan ``b`` dikemas
dalam satu slot. Ini berarti slot untuk
``data[4][9].c`` adalah ``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1``.
Jenis nilainya adalah ``uint256``, sehingga menggunakan satu slot.


.. _bytes-and-string:

``bytes`` dan ``string``
------------------------

``bytes`` dan ``string`` dikodekan secara identik.
Secara umum, pengkodean mirip dengan ``bytes1[]``, dalam arti bahwa ada slot untuk array itu sendiri dan
area data yang dihitung menggunakan hash ``keccak256`` dari posisi slot tersebut.
Namun, untuk nilai pendek (lebih pendek dari 32 byte) elemen array disimpan bersama dengan panjangnya di slot yang sama.

Khususnya: jika panjang data paling banyak ``31`` byte, elemen disimpan dalam byte tingkat tinggi (rata kiri) dan byte urutan terendah menyimpan nilai ``panjang * 2``.
Untuk array byte yang menyimpan data dengan panjang ``32`` atau lebih byte, slot utama ``p`` menyimpan ``length * 2 + 1`` dan data disimpan seperti biasa di ``keccak256(p) ``.
Ini berarti bahwa Anda dapat membedakan array pendek dari array panjang dengan memeriksa apakah bit terendah disetel: pendek (tidak disetel) dan panjang (diatur).

.. note::
  Menangani slot yang disandikan secara tidak valid saat ini tidak didukung tetapi dapat ditambahkan di masa mendatang.
  Jika Anda mengompilasi melalui pipeline compiler berbasis IR eksperimental, membaca slot yang dikodekan secara tidak
  valid akan menghasilkan kesalahan ``Panic(0x22)``.

JSON Output
===========

.. _storage-layout-top-level:

Tata letak penyimpanan kontrak dapat diminta melalui :ref:`antarmuka JSON standar <compiler-api>`.
Outputnya adalah objek JSON yang berisi dua kunci, ``storage`` dan ``types``.
Objek ``storage`` adalah array di mana setiap elemen memiliki bentuk berikut:


.. code::


    {
        "astId": 2,
        "contract": "fileA:A",
        "label": "x",
        "offset": 0,
        "slot": "0",
        "type": "t_uint256"
    }

Contoh di atas adalah tata letak penyimpanan ``contract A { uint x; }`` dari unit sumber ``fileA``
dan

- ``astId`` adalah id dari node AST dari deklarasi variabel state
- ``kontrak`` adalah nama kontrak termasuk jalurnya sebagai awalan
- ``label`` adalah nama variabel state
- ``offset`` adalah offset dalam byte dalam slot penyimpanan sesuai dengan pengkodean
- ``slot`` adalah slot penyimpanan tempat variabel state berada atau dimulai. Angka ini mungkin sangat besar dan oleh karena itu nilai JSON-nya direpresentasikan sebagai string.
- ``type`` adalah pengidentifikasi yang digunakan sebagai kunci untuk informasi tipe variabel (dijelaskan berikut ini)

``type`` yang diberikan, dalam hal ini ``t_uint256`` mewakili elemen dalam
``types``, yang berbentuk:


.. code::

    {
        "encoding": "inplace",
        "label": "uint256",
        "numberOfBytes": "32",
    }

dimana

- ``encoding`` bagaimana data dikodekan dalam penyimpanan, di mana nilai yang mungkin adalah:

  - ``inplace``: Data diletakkan secara berurutan dalam penyimpanan (lihat :ref:`di atas <storage-inplace-encoding>`).
  - ``mapping``: Metode Keccak-256 hash-based (lihat :ref:`di atas <storage-hashed-encoding>`).
  - ``dynamic_array``: Metode Keccak-256 hash-based (lihat :ref:`di atas <storage-hashed-encoding>`).
  - ``bytes``: single slot atau Keccak-256 hash-based tergantung dengan ukuran data (lihat :ref:`di atas <bytes-and-string>`).

- ``label`` adalah nama tipe canonical.
- ``numberOfBytes`` adalah jumlah byte yang digunakan (sebagai string desimal).
  Perhatikan bahwa jika ``numberOfBytes > 32`` ini berarti lebih dari satu slot yang digunakan.

Beberapa tipe memiliki informasi tambahan selain keempat di atas. Mapping mengandung
tipe ``key`` dan ``value``-nya (sekali lagi merujuk entri dalam Mapping ini
jenis), array memiliki tipe ``base``, dan struct mencantumkan ``members`` mereka di
format yang sama dengan ``storage`` tingkat atas (lihat :ref:`di atas
<storage-layout-top-level>`).

.. note ::
  Format output JSON dari layout penyimpanan kontrak masih dianggap eksperimental
  dan dapat berubah dalam rilis Solidity yang tidak melanggar.

Contoh berikut menunjukkan kontrak dan tata letak penyimpanannya, yang berisi:
tipe nilai dan referensi, tipe yang encoded packed, dan tipe nested.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;
    contract A {
        struct S {
            uint128 a;
            uint128 b;
            uint[2] staticArray;
            uint[] dynArray;
        }

        uint x;
        uint y;
        S s;
        address addr;
        mapping (uint => mapping (address => bool)) map;
        uint[] array;
        string s1;
        bytes b1;
    }

.. code:: json

    {
      "storage": [
        {
          "astId": 15,
          "contract": "fileA:A",
          "label": "x",
          "offset": 0,
          "slot": "0",
          "type": "t_uint256"
        },
        {
          "astId": 17,
          "contract": "fileA:A",
          "label": "y",
          "offset": 0,
          "slot": "1",
          "type": "t_uint256"
        },
        {
          "astId": 20,
          "contract": "fileA:A",
          "label": "s",
          "offset": 0,
          "slot": "2",
          "type": "t_struct(S)13_storage"
        },
        {
          "astId": 22,
          "contract": "fileA:A",
          "label": "addr",
          "offset": 0,
          "slot": "6",
          "type": "t_address"
        },
        {
          "astId": 28,
          "contract": "fileA:A",
          "label": "map",
          "offset": 0,
          "slot": "7",
          "type": "t_mapping(t_uint256,t_mapping(t_address,t_bool))"
        },
        {
          "astId": 31,
          "contract": "fileA:A",
          "label": "array",
          "offset": 0,
          "slot": "8",
          "type": "t_array(t_uint256)dyn_storage"
        },
        {
          "astId": 33,
          "contract": "fileA:A",
          "label": "s1",
          "offset": 0,
          "slot": "9",
          "type": "t_string_storage"
        },
        {
          "astId": 35,
          "contract": "fileA:A",
          "label": "b1",
          "offset": 0,
          "slot": "10",
          "type": "t_bytes_storage"
        }
      ],
      "types": {
        "t_address": {
          "encoding": "inplace",
          "label": "address",
          "numberOfBytes": "20"
        },
        "t_array(t_uint256)2_storage": {
          "base": "t_uint256",
          "encoding": "inplace",
          "label": "uint256[2]",
          "numberOfBytes": "64"
        },
        "t_array(t_uint256)dyn_storage": {
          "base": "t_uint256",
          "encoding": "dynamic_array",
          "label": "uint256[]",
          "numberOfBytes": "32"
        },
        "t_bool": {
          "encoding": "inplace",
          "label": "bool",
          "numberOfBytes": "1"
        },
        "t_bytes_storage": {
          "encoding": "bytes",
          "label": "bytes",
          "numberOfBytes": "32"
        },
        "t_mapping(t_address,t_bool)": {
          "encoding": "mapping",
          "key": "t_address",
          "label": "mapping(address => bool)",
          "numberOfBytes": "32",
          "value": "t_bool"
        },
        "t_mapping(t_uint256,t_mapping(t_address,t_bool))": {
          "encoding": "mapping",
          "key": "t_uint256",
          "label": "mapping(uint256 => mapping(address => bool))",
          "numberOfBytes": "32",
          "value": "t_mapping(t_address,t_bool)"
        },
        "t_string_storage": {
          "encoding": "bytes",
          "label": "string",
          "numberOfBytes": "32"
        },
        "t_struct(S)13_storage": {
          "encoding": "inplace",
          "label": "struct A.S",
          "members": [
            {
              "astId": 3,
              "contract": "fileA:A",
              "label": "a",
              "offset": 0,
              "slot": "0",
              "type": "t_uint128"
            },
            {
              "astId": 5,
              "contract": "fileA:A",
              "label": "b",
              "offset": 16,
              "slot": "0",
              "type": "t_uint128"
            },
            {
              "astId": 9,
              "contract": "fileA:A",
              "label": "staticArray",
              "offset": 0,
              "slot": "1",
              "type": "t_array(t_uint256)2_storage"
            },
            {
              "astId": 12,
              "contract": "fileA:A",
              "label": "dynArray",
              "offset": 0,
              "slot": "3",
              "type": "t_array(t_uint256)dyn_storage"
            }
          ],
          "numberOfBytes": "128"
        },
        "t_uint128": {
          "encoding": "inplace",
          "label": "uint128",
          "numberOfBytes": "16"
        },
        "t_uint256": {
          "encoding": "inplace",
          "label": "uint256",
          "numberOfBytes": "32"
        }
      }
    }
=======
.. index:: storage, state variable, mapping

************************************
Layout of State Variables in Storage
************************************

.. _storage-inplace-encoding:

State variables of contracts are stored in storage in a compact way such
that multiple values sometimes use the same storage slot.
Except for dynamically-sized arrays and mappings (see below), data is stored
contiguously item after item starting with the first state variable,
which is stored in slot ``0``. For each variable,
a size in bytes is determined according to its type.
Multiple, contiguous items that need less than 32 bytes are packed into a single
storage slot if possible, according to the following rules:

- The first item in a storage slot is stored lower-order aligned.
- Value types use only as many bytes as are necessary to store them.
- If a value type does not fit the remaining part of a storage slot, it is stored in the next storage slot.
- Structs and array data always start a new slot and their items are packed tightly according to these rules.
- Items following struct or array data always start a new storage slot.

For contracts that use inheritance, the ordering of state variables is determined by the
C3-linearized order of contracts starting with the most base-ward contract. If allowed
by the above rules, state variables from different contracts do share the same storage slot.

The elements of structs and arrays are stored after each other, just as if they were given
as individual values.

.. warning::
    When using elements that are smaller than 32 bytes, your contract's gas usage may be higher.
    This is because the EVM operates on 32 bytes at a time. Therefore, if the element is smaller
    than that, the EVM must use more operations in order to reduce the size of the element from 32
    bytes to the desired size.

    It might be beneficial to use reduced-size types if you are dealing with storage values
    because the compiler will pack multiple elements into one storage slot, and thus, combine
    multiple reads or writes into a single operation.
    If you are not reading or writing all the values in a slot at the same time, this can
    have the opposite effect, though: When one value is written to a multi-value storage
    slot, the storage slot has to be read first and then
    combined with the new value such that other data in the same slot is not destroyed.

    When dealing with function arguments or memory
    values, there is no inherent benefit because the compiler does not pack these values.

    Finally, in order to allow the EVM to optimize for this, ensure that you try to order your
    storage variables and ``struct`` members such that they can be packed tightly. For example,
    declaring your storage variables in the order of ``uint128, uint128, uint256`` instead of
    ``uint128, uint256, uint128``, as the former will only take up two slots of storage whereas the
    latter will take up three.

.. note::
     The layout of state variables in storage is considered to be part of the external interface
     of Solidity due to the fact that storage pointers can be passed to libraries. This means that
     any change to the rules outlined in this section is considered a breaking change
     of the language and due to its critical nature should be considered very carefully before
     being executed. In the event of such a breaking change, we would want to release a
     compatibility mode in which the compiler would generate bytecode supporting the old layout.


Mappings and Dynamic Arrays
===========================

.. _storage-hashed-encoding:

Due to their unpredictable size, mappings and dynamically-sized array types cannot be stored
"in between" the state variables preceding and following them.
Instead, they are considered to occupy only 32 bytes with regards to the
:ref:`rules above <storage-inplace-encoding>` and the elements they contain are stored starting at a different
storage slot that is computed using a Keccak-256 hash.

Assume the storage location of the mapping or array ends up being a slot ``p``
after applying :ref:`the storage layout rules <storage-inplace-encoding>`.
For dynamic arrays,
this slot stores the number of elements in the array (byte arrays and
strings are an exception, see :ref:`below <bytes-and-string>`).
For mappings, the slot stays empty, but it is still needed to ensure that even if there are
two mappings next to each other, their content ends up at different storage locations.

Array data is located starting at ``keccak256(p)`` and it is laid out in the same way as
statically-sized array data would: One element after the other, potentially sharing
storage slots if the elements are not longer than 16 bytes. Dynamic arrays of dynamic arrays apply this
rule recursively. The location of element ``x[i][j]``, where the type of ``x`` is ``uint24[][]``, is
computed as follows (again, assuming ``x`` itself is stored at slot ``p``):
The slot is ``keccak256(keccak256(p) + i) + floor(j / floor(256 / 24))`` and
the element can be obtained from the slot data ``v`` using ``(v >> ((j % floor(256 / 24)) * 24)) & type(uint24).max``.

The value corresponding to a mapping key ``k`` is located at ``keccak256(h(k) . p)``
where ``.`` is concatenation and ``h`` is a function that is applied to the key depending on its type:

- for value types, ``h`` pads the value to 32 bytes in the same way as when storing the value in memory.
- for strings and byte arrays, ``h(k)`` is just the unpadded data.

If the mapping value is a
non-value type, the computed slot marks the start of the data. If the value is of struct type,
for example, you have to add an offset corresponding to the struct member to reach the member.

As an example, consider the following contract:

.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;


    contract C {
        struct S { uint16 a; uint16 b; uint256 c; }
        uint x;
        mapping(uint => mapping(uint => S)) data;
    }

Let us compute the storage location of ``data[4][9].c``.
The position of the mapping itself is ``1`` (the variable ``x`` with 32 bytes precedes it).
This means ``data[4]`` is stored at ``keccak256(uint256(4) . uint256(1))``. The type of ``data[4]`` is
again a mapping and the data for ``data[4][9]`` starts at slot
``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1)))``.
The slot offset of the member ``c`` inside the struct ``S`` is ``1`` because ``a`` and ``b`` are packed
in a single slot. This means the slot for
``data[4][9].c`` is ``keccak256(uint256(9) . keccak256(uint256(4) . uint256(1))) + 1``.
The type of the value is ``uint256``, so it uses a single slot.


.. _bytes-and-string:

``bytes`` and ``string``
------------------------

``bytes`` and ``string`` are encoded identically.
In general, the encoding is similar to ``bytes1[]``, in the sense that there is a slot for the array itself and
a data area that is computed using a ``keccak256`` hash of that slot's position.
However, for short values (shorter than 32 bytes) the array elements are stored together with the length in the same slot.

In particular: if the data is at most ``31`` bytes long, the elements are stored
in the higher-order bytes (left aligned) and the lowest-order byte stores the value ``length * 2``.
For byte arrays that store data which is ``32`` or more bytes long, the main slot ``p`` stores ``length * 2 + 1`` and the data is
stored as usual in ``keccak256(p)``. This means that you can distinguish a short array from a long array
by checking if the lowest bit is set: short (not set) and long (set).

.. note::
  Handling invalidly encoded slots is currently not supported but may be added in the future.
  If you are compiling via IR, reading an invalidly encoded slot results in a ``Panic(0x22)`` error.

JSON Output
===========

.. _storage-layout-top-level:

The storage layout of a contract can be requested via
the :ref:`standard JSON interface <compiler-api>`.  The output is a JSON object containing two keys,
``storage`` and ``types``.  The ``storage`` object is an array where each
element has the following form:


.. code::


    {
        "astId": 2,
        "contract": "fileA:A",
        "label": "x",
        "offset": 0,
        "slot": "0",
        "type": "t_uint256"
    }

The example above is the storage layout of ``contract A { uint x; }`` from source unit ``fileA``
and

- ``astId`` is the id of the AST node of the state variable's declaration
- ``contract`` is the name of the contract including its path as prefix
- ``label`` is the name of the state variable
- ``offset`` is the offset in bytes within the storage slot according to the encoding
- ``slot`` is the storage slot where the state variable resides or starts. This
  number may be very large and therefore its JSON value is represented as a
  string.
- ``type`` is an identifier used as key to the variable's type information (described in the following)

The given ``type``, in this case ``t_uint256`` represents an element in
``types``, which has the form:


.. code::

    {
        "encoding": "inplace",
        "label": "uint256",
        "numberOfBytes": "32",
    }

where

- ``encoding`` how the data is encoded in storage, where the possible values are:

  - ``inplace``: data is laid out contiguously in storage (see :ref:`above <storage-inplace-encoding>`).
  - ``mapping``: Keccak-256 hash-based method (see :ref:`above <storage-hashed-encoding>`).
  - ``dynamic_array``: Keccak-256 hash-based method (see :ref:`above <storage-hashed-encoding>`).
  - ``bytes``: single slot or Keccak-256 hash-based depending on the data size (see :ref:`above <bytes-and-string>`).

- ``label`` is the canonical type name.
- ``numberOfBytes`` is the number of used bytes (as a decimal string).
  Note that if ``numberOfBytes > 32`` this means that more than one slot is used.

Some types have extra information besides the four above. Mappings contain
its ``key`` and ``value`` types (again referencing an entry in this mapping
of types), arrays have its ``base`` type, and structs list their ``members`` in
the same format as the top-level ``storage`` (see :ref:`above
<storage-layout-top-level>`).

.. note ::
  The JSON output format of a contract's storage layout is still considered experimental
  and is subject to change in non-breaking releases of Solidity.

The following example shows a contract and its storage layout, containing
value and reference types, types that are encoded packed, and nested types.


.. code-block:: solidity

    // SPDX-License-Identifier: GPL-3.0
    pragma solidity >=0.4.0 <0.9.0;
    contract A {
        struct S {
            uint128 a;
            uint128 b;
            uint[2] staticArray;
            uint[] dynArray;
        }

        uint x;
        uint y;
        S s;
        address addr;
        mapping (uint => mapping (address => bool)) map;
        uint[] array;
        string s1;
        bytes b1;
    }

.. code:: json

    {
      "storage": [
        {
          "astId": 15,
          "contract": "fileA:A",
          "label": "x",
          "offset": 0,
          "slot": "0",
          "type": "t_uint256"
        },
        {
          "astId": 17,
          "contract": "fileA:A",
          "label": "y",
          "offset": 0,
          "slot": "1",
          "type": "t_uint256"
        },
        {
          "astId": 20,
          "contract": "fileA:A",
          "label": "s",
          "offset": 0,
          "slot": "2",
          "type": "t_struct(S)13_storage"
        },
        {
          "astId": 22,
          "contract": "fileA:A",
          "label": "addr",
          "offset": 0,
          "slot": "6",
          "type": "t_address"
        },
        {
          "astId": 28,
          "contract": "fileA:A",
          "label": "map",
          "offset": 0,
          "slot": "7",
          "type": "t_mapping(t_uint256,t_mapping(t_address,t_bool))"
        },
        {
          "astId": 31,
          "contract": "fileA:A",
          "label": "array",
          "offset": 0,
          "slot": "8",
          "type": "t_array(t_uint256)dyn_storage"
        },
        {
          "astId": 33,
          "contract": "fileA:A",
          "label": "s1",
          "offset": 0,
          "slot": "9",
          "type": "t_string_storage"
        },
        {
          "astId": 35,
          "contract": "fileA:A",
          "label": "b1",
          "offset": 0,
          "slot": "10",
          "type": "t_bytes_storage"
        }
      ],
      "types": {
        "t_address": {
          "encoding": "inplace",
          "label": "address",
          "numberOfBytes": "20"
        },
        "t_array(t_uint256)2_storage": {
          "base": "t_uint256",
          "encoding": "inplace",
          "label": "uint256[2]",
          "numberOfBytes": "64"
        },
        "t_array(t_uint256)dyn_storage": {
          "base": "t_uint256",
          "encoding": "dynamic_array",
          "label": "uint256[]",
          "numberOfBytes": "32"
        },
        "t_bool": {
          "encoding": "inplace",
          "label": "bool",
          "numberOfBytes": "1"
        },
        "t_bytes_storage": {
          "encoding": "bytes",
          "label": "bytes",
          "numberOfBytes": "32"
        },
        "t_mapping(t_address,t_bool)": {
          "encoding": "mapping",
          "key": "t_address",
          "label": "mapping(address => bool)",
          "numberOfBytes": "32",
          "value": "t_bool"
        },
        "t_mapping(t_uint256,t_mapping(t_address,t_bool))": {
          "encoding": "mapping",
          "key": "t_uint256",
          "label": "mapping(uint256 => mapping(address => bool))",
          "numberOfBytes": "32",
          "value": "t_mapping(t_address,t_bool)"
        },
        "t_string_storage": {
          "encoding": "bytes",
          "label": "string",
          "numberOfBytes": "32"
        },
        "t_struct(S)13_storage": {
          "encoding": "inplace",
          "label": "struct A.S",
          "members": [
            {
              "astId": 3,
              "contract": "fileA:A",
              "label": "a",
              "offset": 0,
              "slot": "0",
              "type": "t_uint128"
            },
            {
              "astId": 5,
              "contract": "fileA:A",
              "label": "b",
              "offset": 16,
              "slot": "0",
              "type": "t_uint128"
            },
            {
              "astId": 9,
              "contract": "fileA:A",
              "label": "staticArray",
              "offset": 0,
              "slot": "1",
              "type": "t_array(t_uint256)2_storage"
            },
            {
              "astId": 12,
              "contract": "fileA:A",
              "label": "dynArray",
              "offset": 0,
              "slot": "3",
              "type": "t_array(t_uint256)dyn_storage"
            }
          ],
          "numberOfBytes": "128"
        },
        "t_uint128": {
          "encoding": "inplace",
          "label": "uint128",
          "numberOfBytes": "16"
        },
        "t_uint256": {
          "encoding": "inplace",
          "label": "uint256",
          "numberOfBytes": "32"
        }
      }
    }
>>>>>>> 724af73fb8203779ce835ed77c4070c2b80d0aaa
