# Mifare DESFire

## About

The [Mifare DESfire](https://www.nxp.com/products/rfid-nfc/mifare-hf/mifare-desfire:MC_53450) card is the latest Mifare version supported by the Gallagher system. When a site has been configured with a non-default *Mifare Site Key*, the use of such cards is considered secure by Gallagher (and by this research!)

These cards hold data in a different structure than the Mifare Classic and Mifare Plus cards; instead of a sector/block-based memory layout, they hold individual files and keys in application (i.e. card use-case) specific structures.


## Keys

The keys used in the applications are either fixed or generated by diversifying the Mifare Site Key using application- and key-specific data, as outlined below.

The algorithm appears to have meant to be the NXP-defined [AN10922](https://www.nxp.com/docs/en/application-note/AN10922.pdf) algorithm, but it has a slight difference: the last output block of diversification input is re-ciphered before XORing with K<sub>2</sub> (using the notation in the document linked above). Whether this is intentional or a mistake is hard to tell.

For most keys, the input to the diversification algorithm can take one of two forms:

* Card serial number (CSN) included: in which case, the input is the 4 byte CSN, followed by the 1 byte key number, followed by the 3 byte AID.

* Card serial number excluded: in which case, the input is the 1 byte key number, followed by the 3 byte AID.

Finally, the default application key is the result of diversifiying the bytes `0x03 0x00 0x00 0x00`.


## Applications

A Gallagher-encoded DESFire card will have at least two applications, and potentially more if more than one site is supported by the card.

### Card application

The card-wide card application has the NXP-defined AID of `0x000000`. It has one key:

* Key 0x0: Card Master Key (CMK): diversified without the CSN.

### Cardax card data application

These applications have an application ID (AID) between `0xF48120` and `0xF4812B`, inclusive. Note that these use an NXP-defined mapping of Gallagher's 2 byte AIDs used in the Mifare Classic and Mifare Plus Mifare Application Directory (MAD) into the 3 byte AIDs used in DESFire.

The application has a configuration byte of `0x0B`.

There are three keys defined for the application:

* Key 0x0: Application Master Key (AMK): diversified with the CSN.
* Key 0x1: "UID Discovery": diversified without the CSN.
* Key 0x2: Cardax read: diversified with the CSN.

There are two files:

* File 0x0: "Cardax standard". This contains an 8 byte [cardholder credential data](../cardholder/cardholder.md) block, followed by its bitwise inverse (like block 0 of a [Mifare Classic](mifare-classic.md) card). It has permissions `0x2000`.
* File 0x1: "Cardax enhanced". This contains a 16 byte [Mifare Enhanced Security](../mes.md) block, if enabled for the site (otherwise the file is not present). It also has permissions `0x2000`.

### Card Application Directory (CAD)

The card application directory application (!) has AID `0xF4812F` (again, this is a mapped 2 byte ID). The application holds data similarly to (but *not* the same format as) the Mifare Classic and Mifare Plus [card application directories](cad.md).

The application has a configuration byte of `0x0B`.

There is one key defined for the application:

* Key 0x0: Application Master Key (AMK).

There is at least one and up to three files:

* Files 0x0 - 0x2: CAD. This contains up to six entries, each of which contain:
  - 1 byte RC
  - 2 byte FC
  - 3 byte AID, written in reverse byte order.

It has permissions `0xE000`.

### General user info

This application, defined by NXP in [AN10787](https://www.nxp.com/docs/en/application-note/AN10787.pdf), contains general informtion on the holder and uses of the card. It has AID `0xFFFFFF`.

There is one key defined for the application:

* Key 0x0: Application Master Key (AMK): diversified with the CSN.

There are two files defined:

* File 0x0: MAD version. Contains the bytes `0x00 0x00 0x03` and has permissions `0xE000`.
* File 0x2: card publisher. Contains the string <tt>www.cardax.com&nbsp;&nbsp;</tt> followed by a `0x00` byte (note the two spaces, and has permissions `0xE000`.


## Example

Here is the contents of an example Mifare DESFire card:

### Application `0xF48120`

#### File `0x0`

Contains `A3 B4 B0 C1 51 B0 A3 34 5C 4B 4F 3E AE 4F 5C CB`.

We can see the cardholder credential block `0xA3B4B0C151B0A334` (followed by its bitwise inverse), which [decodes to](../cardholder/cardholder.md) (RC 12 (M), FC 0x1337 = 4919, CN 0xF00D = 61453, IL = 3).

#### File `0x1`

Contains `1A D0 8D D6 2F B3 E4 38 BE 7A 05 E7 CB 0B 1B C7`.

This is a [MES block](../mes.md).

### Application `0xF48121`

#### File `0x0`

Contains `A3 B4 B0 C8 51 B0 A3 A2 5C 4B 4F 37 AE 4F 5C 5D`.

We can see the cardholder credential block `0xA3B4B0C851B0A3A2` (followed by its bitwise inverse), which [decodes to](../cardholder/cardholder.md) (RC 13 (N), FC 0x1338 = 4920, CN 0xF00E = 61454, IL = 1).

#### File `0x1`

Contains `1A D0 8D D6 2F B3 E4 38 BE 7A 05 E7 CB 0B 1B C7`.

This is a [MES block](../mes.md).

### Application `0xF4812F`

Contains:

```
0C 13 37 20 81 F4 0D 13 38 21 81 F4 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00
```

We can see 2 6-byte entries: `0x0C13372081F4` and `0x0D13382181F4`, which [decode to](#card-application-directory-cad):

| RC  | FC       | AID        |
|-----|----------|------------|
| `C` | `0x1337` | `0x2081F4` |
| `D` | `0x1338` | `0x2181F4` |

This corresponds with what we read in the previous application files.

### Application `0xFFFFFF`

This file contains the [general user info](#general-user-info):

#### File `0x0`

Contains `00 00 03`, as expected.

#### File `0x2`

Contains `77 77 77 2e 63 61 72 64 61 78 2e 63 6f 6d 20 20 00`, as expected.