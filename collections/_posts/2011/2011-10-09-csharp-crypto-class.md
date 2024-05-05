---
layout: post
title: "A simple static cryptography class"
categories: blog
---

Implement things once, never think about them again. Cryptography is not something I'm constantly working with, so when I do need it for something I usually have to look up how things worked again and I end up reinventing the wheel. Well no more.

I've gathered some frequently used cryptography functions and I've combined them in a static class for easy access. Functionality includes:

*   Fetching a cryptographically strong sequence of random bytes
*   SHA-512 hashing of strings
*   RSA encryption/decryption of strings
*   AES encryption/decryption of strings and files
*   Securely erasing files

Full code listing:

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;
using System.Security.Cryptography;
using System.Text;

namespace Com.BartWolff.Util
{
    /// <summary>
    /// Static class providing easy access to strong, proven encryption schemes.
    /// </summary>
    public static class Cryptography
    {
        private static readonly byte[] AES_IV = new byte[]
                                        {
                                            0x7a, 0x42, 0x59, 0x24, 0x6c, 0x70, 0x3e, 0x39,
                                            0x13, 0x4b, 0x71, 0x1f, 0x52, 0x35, 0x61, 0x2d
                                        };

        private static readonly byte[] AES_SALT = new byte[]
                                        {
                                            0x49, 0x76, 0x21, 0x6e, 0x20, 0x4d, 0x65, 0x18,
                                            0x76, 0x65, 0x3a, 0x5c, 0x29, 0x7f, 0x10, 0x62
                                        };

        /// <summary>
        /// Gets a cryptographically strong sequence of random bytes.
        /// </summary>
        /// <param name="length">The amount of bytes.</param>
        /// <returns></returns>
        public static byte[] GetRandomBytes(int length)
        {
            var rngProvider = new RNGCryptoServiceProvider();
            var bytes = new byte[length];
            rngProvider.GetBytes(bytes);
            return bytes;
        }

        /// <summary>
        /// Computes the SHA-512 hash for a given string.
        /// </summary>
        /// <param name="text">The string to hash.</param>
        /// <returns></returns>
        public static String SHA512Hash(String text)
        {
            var textBytes = Encoding.UTF8.GetBytes(text);
            var shaProvider = new SHA512Managed();
            var hashBytes = shaProvider.ComputeHash(textBytes);
            return Convert.ToBase64String(hashBytes);
        }

        /// <summary>
        /// Generates a new RSA keypair.
        /// </summary>
        /// <returns></returns>
        public static RSACryptoServiceProvider RSAGenerateNewKeypair()
        {
            return new RSACryptoServiceProvider(4096);
        }

        /// <summary>
        /// Loads an RSA keypair for a given XML string.
        /// </summary>
        /// <param name="xmlString">The XML string.</param>
        /// <returns></returns>
        public static RSACryptoServiceProvider RSALoadKeypairFromXml(String xmlString)
        {
            var rsaProvider = new RSACryptoServiceProvider();
            rsaProvider.FromXmlString(xmlString);
            return rsaProvider;
        }

        /// <summary>
        /// Encrypts the given string using the public key of the given RSA provider.
        /// </summary>
        /// <param name="text">The text to encrypt.</param>
        /// <param name="rsaProvider">The RSA provider.</param>
        /// <returns></returns>
        public static String RSAEncryptString(String text, RSACryptoServiceProvider rsaProvider)
        {
            var textBytes = Encoding.UTF8.GetBytes(text);
            var cipherBytes = rsaProvider.Encrypt(textBytes, true);
            return Convert.ToBase64String(cipherBytes);
        }

        /// <summary>
        /// Decrypts the given string using the private key of the given RSA provider.
        /// </summary>
        /// <param name="text">The text to decrypt.</param>
        /// <param name="rsaProvider">The RSA provider.</param>
        /// <returns></returns>
        public static String RSADecryptString(String text, RSACryptoServiceProvider rsaProvider)
        {
            var cipherBytes = Convert.FromBase64String(text);
            var textBytes = rsaProvider.Decrypt(cipherBytes, true);
            return Encoding.UTF8.GetString(textBytes);
        }

        /// <summary>
        /// Generates a new 256-bit AES key.
        /// </summary>
        /// <returns></returns>
        public static String AESGenerateNewKey()
        {
            return Convert.ToBase64String(GetRandomBytes(32));
        }

        /// <summary>
        /// Encrypts the given string using AES and the given key.
        /// </summary>
        /// <param name="text">The text to encrypt.</param>
        /// <param name="key">The encryption key.</param>
        /// <returns></returns>
        public static String AESEncryptString(String text, String key)
        {
            var keyBytes = new Rfc2898DeriveBytes(Encoding.UTF8.GetBytes(key), AES_SALT, 16);
            var textBytes = Encoding.UTF8.GetBytes(text);
            var aesProvider = new RijndaelManaged();

            aesProvider.Mode = CipherMode.CFB;
            aesProvider.Padding = PaddingMode.ISO10126;
            var cryptor = aesProvider.CreateEncryptor(keyBytes.GetBytes(32), AES_IV);
            var memoryStream = new MemoryStream();
            var cryptoStream = new CryptoStream(memoryStream, cryptor, CryptoStreamMode.Write);
            var writer = new StreamWriter(cryptoStream, Encoding.UTF8);

            writer.Write(text);
            writer.Flush();
            cryptoStream.FlushFinalBlock();

            var cipherBytes = memoryStream.ToArray();

            writer.Close();
            cryptoStream.Close();
            memoryStream.Close();

            return Convert.ToBase64String(cipherBytes);
        }

        /// <summary>
        /// Decrypts the given string using AES and the given key.
        /// </summary>
        /// <param name="text">The text to decrypt.</param>
        /// <param name="key">The encryption key.</param>
        /// <returns></returns>
        public static String AESDecryptString(String text, String key)
        {
            var keyBytes = new Rfc2898DeriveBytes(Encoding.UTF8.GetBytes(key), AES_SALT, 16);
            var cipherBytes = Convert.FromBase64String(text);
            var aesProvider = new RijndaelManaged();

            aesProvider.Mode = CipherMode.CFB;
            aesProvider.Padding = PaddingMode.ISO10126;
            var cryptor = aesProvider.CreateDecryptor(keyBytes.GetBytes(32), AES_IV);
            var memoryStream = new MemoryStream(cipherBytes);
            var cryptoStream = new CryptoStream(memoryStream, cryptor, CryptoStreamMode.Read);
            var reader = new StreamReader(cryptoStream, Encoding.UTF8);

            var textStr = reader.ReadToEnd();

            reader.Close();
            cryptoStream.Close();
            memoryStream.Close();

            return textStr;
        }

        /// <summary>
        /// Encrypts the given file using AES and the given key.
        /// Writes the encrypted file to the given location.
        /// </summary>
        /// <param name="fileInPath">The file to encrypt.</param>
        /// <param name="fileOutPath">The path to write the encrypted file to.</param>
        /// <param name="key">The encryption key.</param>
        /// <returns></returns>
        public static void AESEncryptFile(String fileInPath, String fileOutPath, String key)
        {
            var keyBytes = new Rfc2898DeriveBytes(Encoding.UTF8.GetBytes(key), AES_SALT, 16);
            var aesProvider = new RijndaelManaged();

            aesProvider.Mode = CipherMode.CFB;
            aesProvider.Padding = PaddingMode.ISO10126;
            var cryptor = aesProvider.CreateEncryptor(keyBytes.GetBytes(32), AES_IV);
            var inStream = new FileStream(fileInPath, FileMode.Open, FileAccess.Read);
            var outStream = new FileStream(fileOutPath, FileMode.Create, FileAccess.Write);
            var cryptoStream = new CryptoStream(outStream, cryptor, CryptoStreamMode.Write);

            int data;
            while ((data = inStream.ReadByte()) != -1)
            {
                cryptoStream.WriteByte((byte)data);
            }
            cryptoStream.FlushFinalBlock();
            outStream.Flush();

            inStream.Close();
            cryptoStream.Close();
            outStream.Close();
        }

        /// <summary>
        /// Decrypts the given file using AES and the given key.
        /// Writes the decrypted file to the given location.
        /// </summary>
        /// <param name="fileInPath">The file to decrypt.</param>
        /// <param name="fileOutPath">The path to write the decrypted file to.</param>
        /// <param name="key">The encryption key.</param>
        /// <returns></returns>
        public static void AESDecryptFile(String fileInPath, String fileOutPath, String key)
        {
            var keyBytes = new Rfc2898DeriveBytes(Encoding.UTF8.GetBytes(key), AES_SALT, 16);
            var aesProvider = new RijndaelManaged();

            aesProvider.Mode = CipherMode.CFB;
            aesProvider.Padding = PaddingMode.ISO10126;
            var cryptor = aesProvider.CreateDecryptor(keyBytes.GetBytes(32), AES_IV);
            var inStream = new FileStream(fileInPath, FileMode.Open, FileAccess.Read);
            var outStream = new FileStream(fileOutPath, FileMode.Create, FileAccess.Write);
            var cryptoStream = new CryptoStream(inStream, cryptor, CryptoStreamMode.Read);

            int data;
            while ((data = cryptoStream.ReadByte()) != -1)
            {
                outStream.WriteByte((byte)data);
            }
            outStream.Flush();

            outStream.Close();
            cryptoStream.Close();
            inStream.Close();
        }

        /// <summary>
        /// Securely erases the given file by overwriting it with random garbage.
        /// </summary>
        /// <param name="file">The file to erase.</param>
        /// <param name="passes">The number of write passes.</param>
        /// <returns></returns>
        public static void EraseFile(FileInfo file, int passes)
        {
            uint sectorsPerCluster;
            uint bytesPerSector;
            uint numberOfFreeClusters;
            uint totalNumberOfClusters;
            GetDiskFreeSpace(String.Format("{0}:\\", file.FullName[0]),
                                out sectorsPerCluster,
                                out bytesPerSector,
                                out numberOfFreeClusters,
                                out totalNumberOfClusters);

            if (file.Exists)
            {
                file.Attributes = FileAttributes.Normal;

                var sectors = Math.Ceiling(file.Length / Convert.ToDouble(bytesPerSector));
                var buffer = new byte[bytesPerSector];
                var rng = new RNGCryptoServiceProvider();

                var inputStream = new FileStream(file.FullName, FileMode.Open);
                for (int i = 0; i < passes; i++)
                {
                    inputStream.Position = 0;
                    for (int j = 0; j < sectors; j++)
                    {
                        rng.GetBytes(buffer);
                        inputStream.Write(buffer, 0, buffer.Length);
                    }
                }
                inputStream.SetLength(0);
                inputStream.Close();

                var dt = new DateTime(2037, 1, 1, 0, 0, 0);
                file.CreationTime = dt;
                file.LastAccessTime = dt;
                file.LastWriteTime = dt;

                file.Delete();
            }
        }

        [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Auto)]
        static extern bool GetDiskFreeSpace(string lpRootPathName,
            out uint lpSectorsPerCluster,
            out uint lpBytesPerSector,
            out uint lpNumberOfFreeClusters,
            out uint lpTotalNumberOfClusters);
    }
}
```

Happy coding!
