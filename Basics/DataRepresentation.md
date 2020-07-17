# 数据的储存和表示

计算机系统中的数据储存是通过物理芯片中储存单元的电位高低来实现的。

**bit（位）**

每一个储存单元都可以表示高或低 2 种情况，表现在计算机系统中即为 1 bit。

**byte（字节）**

每 8 个储存单元构成一个块，即 byte，它可以表示 2^8 种情况，1 byte = 8 bit。

**word size（字长）**

C 语言定义了一些基本的数据类型，每种数据类型通过固定的字节（byte）长度来表示，即 word size。

下表是 C 中数据结构的典型大小：

| 有符号数据类型 | 无符号数据类型 | 字长 32 位 | 字长 64 位 |
| ---- | ---- | ---- | ---- |
| [signed] char | unsigned char | 1 | 1 |
| short | unsigned short | 2 | 2 |
| int | unsigned | 4 | 4 |
| long | unsigned long | 4 | 8 |
| int32_t | uint32_t | 4 | 4 |
| int64_t | uint64_t | 8 | 8 |
| char * | | 4 | 8 |
| float | | 4 | 4 |
| double | | 8 | 8 |

> char 表示一个单独的字节，虽然它被用来储存字符串中的单个字符而得名，但它也能用来储存整数。

**整型数据的表示范围**

根据各种数据类型的字长不同，其可以表示的范围也不一样：

| 数据类型 | 最小值 | 最大值 |
| ---- | ----: | ----: |
| [signed] char | -128 | 127 |
| unsigned char | 0 | 255 |
| short | -32768 | 32767 |
| unsigned short | 0 | 65535 |
| int32_t | -2147483648 | 2147483647 |
| uint32_t | 0 | 4294967295 |
| int64_t | -9223372036854775808 | 9223372036854775807 |
| uint64_t | 0 | 18446744073709551615 |

特殊的，在 32 位机器上：

| 数据类型 | 最小值 | 最大值 |
| ---- | ----: | ----: |
| long | -2147483648 | 2147483647 |
| unsigned long | 0 | 4294967295 |

在 64 位机器上：

| 数据类型 | 最小值 | 最大值 |
| ---- | ----: | ----: |
| long | -9223372036854775808 | 9223372036854775807 |
| unsigned long | 0 | 18446744073709551615 |

> 在 16 位机器上，int 或 unsigned 使用两个字节来表示，取值范围分别为 -32768 ~ 32767 和 0 ~ 65535。









