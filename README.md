# Encoder and Decoder
# การเข้ารหัสและถอดรหัสสคริปต์ Lua

โปรเจกต์นี้มีเป้าหมายในการเข้ารหัสและถอดรหัสสคริปต์ Lua เพื่อป้องกันการเข้าถึงหรือการแก้ไขจากผู้ที่ประสงค์ร้าย โดยใช้ C++ ร่วมกับไลบรารี OpenSSL

โปรเจกต์นี้มีเป้าหมายเพื่อศึกษาตัวอย่างการทำงานพื้นฐานของการเข้ารหัสรูปแบบ encrypt หรือ กุญแจ เพื่อเข้ารหัสให้เข้าใจยากขึ้น จำเป็นต้องมี key เพื่อถอด การทำงานดัดแปลงจาก xor string เท่านั้น (เพื่อการศึกษา)
## ข้อกำหนด

- C++20 หรือเวอร์ชันที่ใหม่กว่า
- OpenSSL (สำหรับการเข้ารหัสและถอดรหัส)
- Lua (สำหรับการรันสคริปต์ Lua)

## การติดตั้ง

### บน Ubuntu

ติดตั้งไลบรารีที่จำเป็น:

```sh
sudo apt-get install libssl-dev
sudo apt-get install liblua5.3-dev
```
## ตัวเข้ารหัสสคริปต์ Lua ใน C++
## ไฟล์ `lua_.cpp`

```cpp
#include <iostream>
#include <fstream>
#include <openssl/evp.h>
#include <openssl/aes.h>
#include <openssl/rand.h>

bool encrypt(const std::string& input, const std::string& output, const std::string& key) {
    std::ifstream inFile(input, std::ios::binary);
    std::ofstream outFile(output, std::ios::binary);

    if (!inFile || !outFile) {
        std::cerr << "Error opening file" << std::endl;
        return false;
    }

    unsigned char iv[EVP_MAX_IV_LENGTH];
    if (!RAND_bytes(iv, sizeof(iv))) {
        std::cerr << "Error generating IV" << std::endl;
        return false;
    }

    outFile.write(reinterpret_cast<char*>(iv), sizeof(iv));

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    EVP_EncryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, reinterpret_cast<const unsigned char*>(key.c_str()), iv);

    const size_t bufferSize = 4096;
    unsigned char buffer[bufferSize];
    unsigned char cipherBuffer[bufferSize + EVP_MAX_BLOCK_LENGTH];
    int outLen;

    while (inFile.read(reinterpret_cast<char*>(buffer), bufferSize)) {
        EVP_EncryptUpdate(ctx, cipherBuffer, &outLen, buffer, inFile.gcount());
        outFile.write(reinterpret_cast<char*>(cipherBuffer), outLen);
    }
    EVP_EncryptFinal_ex(ctx, cipherBuffer, &outLen);
    outFile.write(reinterpret_cast<char*>(cipherBuffer), outLen);

    EVP_CIPHER_CTX_free(ctx);
    return true;
}

int main(int argc, char* argv[]) {
    if (argc != 4) {
        std::cerr << "Usage: " << argv[0] << " <input> <output> <key>" << std::endl;
        return 1;
    }

    if (encrypt(argv[1], argv[2], argv[3])) {
        std::cout << "Encryption successful" << std::endl;
    } else {
        std::cerr << "Encryption failed" << std::endl;
    }

    return 0;
}
```

```cpp
#include <iostream>
#include <fstream>
#include <lua.hpp>
#include <openssl/evp.h>
#include <openssl/aes.h>

bool decrypt(const std::string& input, std::string& decryptedContent, const std::string& key) {
    std::ifstream inFile(input, std::ios::binary);
    if (!inFile) {
        std::cerr << "Error opening file" << std::endl;
        return false;
    }

    unsigned char iv[EVP_MAX_IV_LENGTH];
    inFile.read(reinterpret_cast<char*>(iv), sizeof(iv));

    EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
    EVP_DecryptInit_ex(ctx, EVP_aes_256_cbc(), NULL, reinterpret_cast<const unsigned char*>(key.c_str()), iv);

    const size_t bufferSize = 4096;
    unsigned char buffer[bufferSize];
    unsigned char plainBuffer[bufferSize + EVP_MAX_BLOCK_LENGTH];
    int outLen;

    while (inFile.read(reinterpret_cast<char*>(buffer), bufferSize)) {
        EVP_DecryptUpdate(ctx, plainBuffer, &outLen, buffer, inFile.gcount());
        decryptedContent.append(reinterpret_cast<char*>(plainBuffer), outLen);
    }
    if (!EVP_DecryptFinal_ex(ctx, plainBuffer, &outLen)) {
        std::cerr << "Decryption failed" << std::endl;
        return false;
    }
    decryptedContent.append(reinterpret_cast<char*>(plainBuffer), outLen);

    EVP_CIPHER_CTX_free(ctx);
    return true;
}

void runLuaScript(const std::string& scriptContent) {
    lua_State* L = luaL_newstate();
    luaL_openlibs(L);

    if (luaL_loadbuffer(L, scriptContent.c_str(), scriptContent.size(), "script") || lua_pcall(L, 0, LUA_MULTRET, 0)) {
        std::cerr << "Error running Lua script: " << lua_tostring(L, -1) << std::endl;
    }

    lua_close(L);
}

int main(int argc, char* argv[]) {
    if (argc != 3) {
        std::cerr << "Usage: " << argv[0] << " <input> <key>" << std::endl;
        return 1;
    }

    std::string decryptedContent;
    if (decrypt(argv[1], decryptedContent, argv[2])) {
        runLuaScript(decryptedContent);
    } else {
        std::cerr << "Decryption failed" << std::endl;
    }

    return 0;
}

```
## คอมไพล์และรัน
```sh
g++ -o `path` lua_.cpp -lssl -lcrypto
```

## การใช้งาน
```sh
./path myscript.lua encrypted_script mysecretkey
```


# โปรดเข้าใจว่านี่เป็นเพียงตัวอย่างการออกแบบเท่านั้น ไม่เหมาะสำหรับใช้งานจริงๆ
