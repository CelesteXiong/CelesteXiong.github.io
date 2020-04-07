---
layout: post
title: LevelDB Print Log
date: 2020-04-07 19:27
categories: 数据库
tags: [LevelDB]
author: sh.xiong
---

变长编码

<!--more-->

```c
#include "leveldb/db.h"
#include <stdint.h>
#include <stdio.h>
#include <iostream>
#include <fstream>
#include "db/log_format.h"

using namespace std;
using namespace leveldb;
using namespace log;

// 定义valuetype
enum ValueType {typedelete = 0x0, typevalue = 0x1};

// 定义变长解码函数
void getVariant32ptr(ifstream *fp, uint32_t *key_length){
    uint32_t result = 0;
    /* 
        解码的两种情况:
        a. 与128相与为0表示待解码的result<=127，所以长度就是一个字节
        直接赋值给*key_length;
        b. 对于超过一个字节的长度编码，解码的过程就是按小端顺序，
        每7位取出，然后移位来组装最后的实际长度，组装接受的表示就是MSB位为0.
    */
    for (uint32_t shift = 0; shift <= 28; shift += 7)
    {
        uint64_t byte = 0;
        fp->read((char*)&byte, sizeof(uint8_t));
        if (byte & 128)
        {
            /// more bytes are present
            result |= ((byte & 127) << shift);
        }
        else
        {
            result |= (byte << shift);
            *key_length = result;
            return ;
        }
    }
}

int main(){
    string filename = "testdb_2/000124.log";
    ifstream fp;
    fp.open(filename, ifstream::in);
    // ifstream.peek(): Returns the next character in the input sequence,
    // without extracting it
    if (fp.peek() == EOF)
    {
        cout << "File not open!"<<endl;
    }
    while (fp.peek() != EOF)
    {
       /* 
            read record header:
            header = checksum(4 byte) + length(2 byte) + type(1 byte)
       */
       // checksum
       uint32_t checksum = 0;
       fp.read((char*) &checksum, sizeof(uint32_t));
       cout << "CheckSum: " << checksum << "\t";

       // length 
       uint16_t length = 0;
       fp.read((char*) &length, sizeof(uint16_t));
       cout << "RecordLength: " << length << "\t";

       // uint32_t expected_crc = crc32c::Unmask(DecodeFiced32(head));
       // uint32_t actual_crc = crc32c::Value(header + 6, 1 + length);

       uint8_t record_type = 0;
       fp.read((char *) &record_type, sizeof(uint8_t));
       cout << "RecordType: ";
       switch (static_cast<RecordType>(record_type & 0xff))
       {
            case kZeroType:
                cout << "kZeroType\t";
                return 0;
            case kFullType:
                cout << "kFullType\t";
                break;
            case kFirstType:
                cout << "kFirstType\t";
                break;
            case kMiddleType:
                cout << "kMiddleType\t";
                break;
            case kLastType:
                cout << "kLastType\t";
                break;
       }
       
       // seqnumber
       uint64_t seqnumber = 0;
       fp.read((char *) &seqnumber, sizeof(uint64_t));
       // cout << "seqnumber:" << seqnumber << "\t";

       // entrynumber
       uint32_t entrynumer = 0;
       fp.read((char *) &entrynumer, sizeof(uint32_t));
       // cout << "entrynumber" << entrynumber << "\t";

       // valuetype
       uint8_t valuetype;
       fp.read((char *) &valuetype, sizeof(uint8_t));
       
       // keylength 
       uint32_t keylength;
       getVariant32ptr(&fp, &keylength);
       cout << "keylength: " << keylength << "\t";
       
       // key
       char *key = new char[keylength];
       fp.read(key, keylength);
       cout << "key: " << key << "\t";       
       
       // type
       switch (static_cast<ValueType>(valuetype & 0xff))
       {
            case typedelete:
                cout << "del"<< endl;
                break;
            case typevalue:
                // valuelength
                uint32_t valuelength;
                getVariant32ptr(&fp, &valuelength);
                cout << "valuelength: " << valuelength << "\t";
                
                // value
                char *value = new char(valuelength);
                fp.read(value, valuelength);
                cout << "value: " << value << endl;
                break;
       }
    }
    fp.close();
    return 0;
}
```