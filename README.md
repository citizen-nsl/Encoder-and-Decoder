# obfuscator
## การออกแบบตัวเข้ารหัสขั้นสูงใน C++ สำหรับ Lua
## การตั้งค่าไลบรารีการเข้ารหัส
## ติดตั้ง OpenSSL หากยังไม่ได้ติดตั้ง

```sh
sudo apt-get install libssl-dev
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

## คอมไพล์และรัน
```sh
g++ -o `path` lua_.cpp -lssl -lcrypto
```

## การใช้งาน
```sh
./path myscript.lua encrypted_script mysecretkey
```


# โปรดเข้าใจว่านี่เป็นเพียงตัวอย่างการออกแบบเท่านั้น ไม่เหมาะสำหรับใช้งานจริงๆ
