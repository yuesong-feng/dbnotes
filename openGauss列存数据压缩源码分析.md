# openGauss列存压缩算法

在openGauss数据库中，相对于行存以页为单元进行压缩，列存以CU为单元具有天然的压缩优势。

在openGauss中有三种压缩级别：LOW, MIDDLE, HIGH。指定的压缩等级越高，则数据的压缩率越高。除此之外还可以选择不开启压缩。

```c
typedef enum OptCompress {
    COMPRESS_NO = 0,
    COMPRESS_LOW,
    COMPRESS_MIDDLE,
    COMPRESS_HIGH,
} OptCompress;
```

对于压缩算法，一共有：

```c++
#define CU_DeltaCompressed 0x0001
#define CU_DicEncode 0x0002
// CU_CompressExtend is used for extended compression.
// For compression added afterwards, it can be defined as CU_CompressExtend + 0x0001~0x0008
#define CU_CompressExtend 0x0004    // Used for extended compression
#define CU_Delta2Compressed 0x0005  // CU_Delta2Compressed equals CU_CompressExtend plus 0x0001
#define CU_XORCompressed 0x0006     // CU_XORCompressed equals CU_CompressExtend plus 0x0002
#define CU_RLECompressed 0x0008
#define CU_LzCompressed 0x0010
#define CU_ZlibCompressed 0x0020
#define CU_BitpackCompressed 0x0040
#define CU_IntLikeCompressed 0x0080
```

对于一个CU的压缩，代码在`CU::Compress(int valCount, int16 compress_modes, int align_size)`函数里：

```c++
void CU::Compress(int valCount, int16 compress_modes, int align_size)
{
    errno_t rc;

    // Step1: 初始化，为compress_buffer分配内存，原数据大小 + NULL值位图大小 + 头部大小，压缩后大小不会超过分配的compress_buffer大小
    m_compressedBufSize = CUAlignUtils::AlignCuSize(m_srcDataSize + m_bpNullRawSize + sizeof(CU), align_size);
    m_compressedBuf = (char*)CStoreMemAlloc::Palloc(m_compressedBufSize, !m_inCUCache);

    int16 headerLen = GetCUHeaderSize();
    char* buf = m_compressedBuf + headerLen;

    // Step 2: 填充CU的NULL值 bitmap 位图
    buf = CompressNullBitmapIfNeed(buf);

    // Step 3: 压缩CU内的数据
    bool compressed = false;
    if (COMPRESS_NO != heaprel_get_compression_from_modes(compress_modes))
        compressed = CompressData(buf, valCount, compress_modes, align_size);

    // case 1: 用户定义数据不需要被加密
    // case 2: 用户定义了加密，但加密后数据大小比未加密数据还大，所以直接使用原来的数据
    if (compressed == false) {
        rc = memcpy_s(buf, m_srcDataSize, m_srcData, m_srcDataSize);
        securec_check(rc, "\0", "\0");
        m_cuSizeExcludePadding = headerLen + m_bpNullCompressedSize + m_srcDataSize;
        m_cuSize = CUAlignUtils::AlignCuSize(m_cuSizeExcludePadding, align_size);
        PADDING_CU(buf + m_srcDataSize, m_cuSize - m_cuSizeExcludePadding);
    }

    // 压缩后进行加密
    CUDataEncrypt(buf);

    // Step 4: 填充 compress_buffer 的头部
    FillCompressBufHeader();

    m_cache_compressed = true;

    // Step 5; 释放原数据buf
    FreeSrcBuf();
}
```

实际的数据压缩发生在`CU::CompressData(_out_ char* outBuf, _in_ int nVals, _in_ int16 compress_modes, int align_size)`函数里：

```c++
bool CU::CompressData(_out_ char* outBuf, _in_ int nVals, _in_ int16 compress_modes, int align_size)
{
    int compressOutSize = 0;
    bool beDelta2Compressed = false;
    bool beXORCompressed = false;

    // 获取压缩级别
    int8 compression = heaprel_get_compression_from_modes(compress_modes);

    // 压缩结果输出到outBuf
    CompressionArg2 output = {0};
    output.buf = outBuf;
    output.sz = (m_compressedBuf + m_compressedBufSize) - outBuf;

    // 压缩的输入，就是CU数据
    CompressionArg1 input = {0};
    input.sz = m_srcDataSize;
    input.buf = m_srcData;
    input.mode = compress_modes;

    // 为当前CU的数据设置 compression filter
    compression_options* ref_filter = (compression_options*)m_tmpinfo->m_options;

    // 如果允许tsdb，并且属性是timestamp或者float类型:
    // tsdb: 时序数据库
    if (g_instance.attr.attr_common.enable_tsdb && (ATT_IS_TIMESTAMP(m_atttypid) || ATT_IS_FLOAT(m_atttypid))) {
        SequenceCodec sequenceCoder(m_eachValSize, m_atttypid);     // 由SequenceCodec类完成
        compressOutSize = sequenceCoder.compress(input, output);
        if (ATT_IS_TIMESTAMP(m_atttypid)) {
            beDelta2Compressed = true;  // timestamp则使用了Delta2压缩
        } else if (ATT_IS_FLOAT(m_atttypid)) {
            beXORCompressed = true;     // float则使用了XOR压缩
        }
    }
    // 如果压缩结果小于0或者没有用Delta2和XOR压缩，说明要么上一步没有成功压缩，要么不是timestamp和float类型
    if (compressOutSize < 0 || (!beDelta2Compressed && !beXORCompressed)) {
        output = {0};
        output.buf = outBuf;
        output.sz = (m_compressedBuf + m_compressedBufSize) - outBuf;

        if (m_infoMode & CU_IntLikeCompressed) {   // 是类整型
            if (ATT_IS_CHAR_TYPE(m_atttypid)) {     // 属性是char类型的
                IntegerCoder intCoder(8);            // 由IntegerCoder类完成

                /* set min/max value */
                if (m_tmpinfo->m_valid_minmax) {
                    intCoder.SetMinMaxVal(m_tmpinfo->m_min_value, m_tmpinfo->m_max_value);
                }
                // 告诉IntegerCoder是否使用RLE压缩
                intCoder.m_adopt_rle = ref_filter->m_adopt_rle;
                compressOutSize = intCoder.Compress(input, output);
            } else if (ATT_IS_NUMERIC_TYPE(m_atttypid)) {       // 属性是numeric类型的
                if (compression > COMPRESS_LOW) {               // 压缩级别大于LOW，是MIDDLE或者HIGH
                    /// numeric data type compression.
                    /// lz4/zlib is used directly.              // 直接使用lz4/zlib压缩
                    input.buildGlobalDict = false;
                    input.useGlobalDict = false;
                    input.globalDict = NULL;
                    input.useDict = false;
                    input.numVals = HasNullValue() ? (nVals - CountNullValuesBefore(nVals)) : nVals;

                    StringCoder strCoder;                       // 由StringCoder类完成
                    compressOutSize = strCoder.Compress(input, output);
                }
            } else {
                // for future, another type             // 未来的其他类型
            }
        } else if (m_eachValSize > 0 && m_eachValSize <= 8) {           // 类型大小在(0, 8]范围
            IntegerCoder intCoder(m_eachValSize);                       // 由IntegerCoder完成
            /* set min/max value */
            if (m_tmpinfo->m_valid_minmax) {
                intCoder.SetMinMaxVal(m_tmpinfo->m_min_value, m_tmpinfo->m_max_value);
            }
            // 告诉IntegerCoder是否使用RLE压缩
            intCoder.m_adopt_rle = ref_filter->m_adopt_rle;         
            compressOutSize = intCoder.Compress(input, output);     // 由IntegerCoder类完成
        } else {
            // FUTURE CASE: complete global dictionary              // 未来会支持的：全局字典压缩
            Assert(-1 == m_eachValSize || m_eachValSize > 8);       // 类型大小大于8，或者为-1（变长字符）
            input.buildGlobalDict = false;
            input.useGlobalDict = false;
            input.globalDict = NULL;

            // 类型大小大于8的直接使用lz4/zlib，不用字典方法

            // 类型大小为-1的，说明是变长类型，如果压缩级别不是LOW则使用字典压缩
            input.useDict = (m_eachValSize > 8) ? false : (COMPRESS_LOW != compression);

            // values的数量是除掉了空值的
            input.numVals = HasNullValue() ? (nVals - CountNullValuesBefore(nVals)) : nVals;

            // 由StringCoder类完成
            StringCoder strCoder;
            // 提示StringCoder类是否使用RLE和字典压缩
            strCoder.m_adopt_rle = ref_filter->m_adopt_rle;
            strCoder.m_adopt_dict = ref_filter->m_adopt_dict;
            compressOutSize = strCoder.Compress(input, output);
        }
    }

    if (compressOutSize > 0) {
        // 压缩成功，计算CU大小，设置相关压缩信息
        Assert((uint32)compressOutSize < m_srcDataSize);
        Assert((0 == (output.modes & CU_INFOMASK2)) && (0 != (output.modes & CU_INFOMASK1)));
        m_infoMode |= (output.modes & CU_INFOMASK1);

        m_cuSizeExcludePadding = (outBuf - m_compressedBuf) + compressOutSize;
        m_cuSize = CUAlignUtils::AlignCuSize(m_cuSizeExcludePadding, align_size);
        Assert(m_cuSize <= m_compressedBufSize);
        PADDING_CU(m_compressedBuf + m_cuSizeExcludePadding, m_cuSize - m_cuSizeExcludePadding);

        if (!ref_filter->m_sampling_fihished) {
            /* sample and set adopted compression methods */
            ref_filter->set_common_flags(output.modes);
        }

        return true;
    }

    return false;
}
```

对于SequenceCodec类的压缩，会先判断类型，如果是timestamp则使用Delta2压缩，是float类型则使用XOR压缩。然后还会判断压缩级别，如果是MIDDLE或HIGH则还需要分别使用lz4和zlib进行压缩。

对于IntegerCoder类的压缩，会先进行Delta压缩，如果使用RLE的话再使用RLE压缩，然后对压缩级别为MIDDLE和HIGH的情况分别使用lz4和zlib压缩。

对于StringCoder类的压缩，会先判断是否使用字典压缩，如果使用则先进行字典压缩，然后对数字部分使用IntegerCoder进行压缩。如果字典压缩失败或者没有允许使用字典压缩，则直接对于LOW和MIDDLE级别使用lz4压缩，对于HIGH级别使用zlib压缩。

-----------------------------

可以看到整个流程为：
先判断是否允许了tsdb（时序数据库）并且类型为timestamp或者float类型，如果是，则压缩是由SequenceCodec类完成的，timestamp则使用了Delta2压缩，float则使用了XOR压缩。如果压缩级别为MIDDLE或HIGH，则还需要进行lz4或zlib压缩。
对于timestamp和float类型：

|           | LOW    | MIDDLE       | HIGH           |
| --------- | ------ | ------------ | -------------- |
| timestamp | Delta2 | Delta2 + lz4 | Delta2  + zlib |
| float     | XOR    | XOR   + lz4  | XOR  + zlib    |

如果不允许tsdb或者上一步没有成功压缩，则进入以下的流程:

如果是IntLike（类整型）：char类型则由IntegerCoder类完成，先使用Delta压缩，如果开启了RLE则使用RLE压缩，如果压缩级别为MIDDLE或HIGH，则还需要进行lz4或zlib压缩。

|      | LOW               | MIDDLE                  | HIGH                      |
| ---- | ----------------- | ----------------------- | ------------------------- |
| char | Delta + RLE(可选) | Delta + RLE(可选) + lz4 | Delta + RLE(可选)  + zlib |

numeric类型则要判断压缩级别，如果级别为LOW，则不压缩numeric类型。当级别为MIDDLE或HIGH时，直接使用lz4/zlib压缩，由StringCoder类完成。

|         | LOW    | MIDDLE | HIGH |
| ------- | ------ | ------ | ---- |
| numeric | 不压缩 | lz4    | zlib |

对于不是IntLike（类整型）的情况，如果类型大小在(0, 8]范围，则由IntegerCoder类完成，并使用RLE压缩，如果级别为MIDDLE或HIGH还需要分别使用lz4和zlib压缩。

|                       | LOW               | MIDDLE                  | HIGH                      |
| --------------------- | ----------------- | ----------------------- | ------------------------- |
| 长度不大于8的定长字符 | Delta + RLE(可选) | Delta + RLE(可选) + lz4 | Delta + RLE(可选)  + zlib |

类型大小大于8的直接使用lz4/zlib，不使用字典压缩：

|                     | LOW | MIDDLE | HIGH |
| ------------------- | --- | ------ | ---- |
| 长度大于8的定长字符 | lz4 | lz4    | zlib |

类型大小为-1的，说明是变长类型，如果压缩级别为LOW则不使用字典，直接使用lz4。如果压缩级别为MIDDLE或HIGH，尝试用字典方法，然后对数字部分使用IntegerCodr进行压缩：

|          | LOW | MIDDLE                         | HIGH                            |
| -------- | --- | ------------------------------ | ------------------------------- |
| 变长字符 | lz4 | 字典 + Delta + RLE(可选) + lz4 | 字典 + Delta + RLE(可选) + zlib |

综上：

|                       | LOW               | MIDDLE                         | HIGH                            |
| --------------------- | ----------------- | ------------------------------ | ------------------------------- |
| timestamp             | Delta2            | Delta2 + lz4                   | Delta2  + zlib                  |
| float                 | XOR               | XOR   + lz4                    | XOR  + zlib                     |
| char                  | Delta + RLE(可选) | Delta + RLE(可选) + lz4        | Delta + RLE(可选)  + zlib       |
| numeric               | 不压缩            | lz4                            | zlib                            |
| 长度不大于8的定长字符 | Delta + RLE(可选) | Delta + RLE(可选) + lz4        | Delta + RLE(可选)  + zlib       |
| 长度大于8的定长字符   | lz4               | lz4                            | zlib                            |
| 变长字符              | lz4               | 字典 + Delta + RLE(可选) + lz4 | 字典 + Delta + RLE(可选) + zlib |
