+++
author = "requizm"
title = "Crackme-2"
date = "2020-06-15"
description = ""
tags = [
    "Assembly",
    "CrackMe",
]
categories = [
    "Reverse Engineering",
]
series = ["Crackme's"]
aliases = ["migrate-from-jekyl"]
+++
  
  
### Uygulama Bilgileri  
[Uygulama Linki](https://crackmes.one/crackme/5ecad51b33c5d449d91ae60c )  
**Yaratıcı:** xy0   
**Zorluk:** Kolay
  
  
### Kullandığım Programlar  
**OllyDbg 1.10:** Dinamik Analiz için.  
  
  
  
### Başlangıç
Basit bir konsol ekranında bizden veri girişi bekliyor, dana sonra girdiğimiz değerin doğru veya yanlış olduğuna dair bir mesaj vererek program bitiyor.
![](/crackme-2/wrong_1.png)
  
  
Bir önceki serideki gibi string search yapalım ve *Enter Key:* parametresinin push edildiği yere gidelim.
![](/crackme-2/cpu_ex1.png)

*Enter Key:* string'i stack'e atılıyor ve *AB3FFC* fonksiyonu çağırılıyor. EAX'e atılan adresten anlayabiliriz ki, çağırılan fonksiyon *std::cout*  
Ardından başka bir fonksiyon çağırılıyor. Tekrar ECX'e atılan adresten anlayabiliriz ki, çağırılan fonksiyon *std::cin*  
İkinci fonksiyon bitimine breakpoint koyarak girdiğimiz verinin hangi adrese kayıt olduğunu ve sonradan nasıl kontrol işlemlerinden geçtiğini görelim. *abcd* girişi yaptım.

Alt taraftaki *Success!* ve *Failed!* işlemlerine nasıl ulaşacağımıza incelersek, ECX'in 0 olması gerektiğini anlıyoruz. Peki, ECX'i 0 yapacak şey ne? Oradaki kodu alıntılayalım:
```
00AA80F0    50              PUSH EAX                                  
00AA80F1    6A 00           PUSH 0
00AA80F3    68 5C050000     PUSH 55C
00AA80F8    E8 3C92FFFF     CALL CrackMe.00AA1339  
00AA80FD    83C4 10         ADD ESP,10
00AA8100    0FB6C8          MOVZX ECX,AL
00AA8103    85C9            TEST ECX,ECX
00AA8105    74 0C           JE SHORT CrackMe.00AA8113
00AA8107    C785 F0FEFFFF 0>MOV DWORD PTR SS:[EBP-110],CrackMe.00AB4>; ASCII "Success!"
00AA8111    EB 0A           JMP SHORT CrackMe.00AA811D
00AA8113    C785 F0FEFFFF 1>MOV DWORD PTR SS:[EBP-110],CrackMe.00AB4>; ASCII "Failed!"
```
*AA1339* fonksiyonu çağırılırken girilen parametrelere dikkat edelim. Debugging yaparak oradaki EAX'in input stringimiz olduğunu görebiliriz. Peki 0 ve 0x55C ne? Belki 0x55C bizi sonuca yaklaştıracak bir şey? Bu fonksiyonu inceleyelim. 
![](/crackme-2/cpu_controlfunc_start.png)
Oops, alt tarafta iç içe for döngüsü bulduk galiba. Orada önemli şeylerin yapıldığı kesin, ancak amacımızı unutmayalım. Bu fonksiyonun 0 dönmesini sağlamak zorundayız. O yüzden for döngülerini şimdilik geçelim ve altlara bakalım.  
![](/crackme-2/cpu_controlfunc_end.png)
Döngü biter bitmez bazı kontrol işlemleri görüyoruz. *[ARG1] == [LOCAL4]* işlemi oluyor. *[ARG1]* 0x55C idi, *[LOCAL4]* bir şekilde 0xCB olmuş. Bu şekilde devam edersek, sonraki JNZ komutunda ileri atlayıp, *[LOCAL63]* değişkenine 0 atmış olacağız ve EAX'te 0'dan farklı bir değer döndürmüş olacağız. Yani burada JNZ'yi pas geçip *[LOCAL63]* değişkenine 1'i atarsak, başarılı mı olmuş olacağız? Evet. Demek ki yapmamız gereken şey, *[ARG1] == [LOCAL4]* denklemini sağlamak. Artık iç içe for döngülerine gelip *LOCAL4* değişkenine neler olduğunu gözlemleyebiliriz. Öncelikle for döngülerinin tamamının bir görüntüsünü görelim.
![](/crackme-2/cpu_for_ex1.png)
  
Komutların solundaki ok işaretlerinden for döngülerinin başlangıç ve bitiş yerlerini görebiliyoruz. 
```
00AA343A  |.  C745 E4 00000>MOV [LOCAL.7],0             ;LOCAL7 = 0
00AA3441  |.  EB 09         JMP SHORT CrackMe.00AA344C  ;FOR başlangıcı
00AA3443  |> /8B45 E4       /MOV EAX,[LOCAL.7]             
00AA3446  |. |83C0 01       |ADD EAX,1                 
00AA3449  |. |8945 E4       |MOV [LOCAL.7],EAX          ;üstteki üç komut ile beraber
                                                        ;LOCAL7 += 1
00AA344C  |> |8B45 10        MOV EAX,[ARG.3]           
00AA344F  |.  50            |PUSH EAX
00AA3450  |.  E8 46DCFFFF   |CALL CrackMe.00AA109B      ;strlen(EAX)     
                                                        ;fonksiyon içine bakarak strlen yazısını görebiliriz 
00AA3455  |.  83C4 04       |ADD ESP,4
00AA3458  |.  3945 E4       |CMP [LOCAL.7],EAX
00AA345B  |.  73 6A         |JNB SHORT CrackMe.00AA34C7 ;LOCAL7 >= EAX ise döngüyü bitir
...
...
00AA34C7  |> \8B45 08       MOV EAX,[ARG.1]             ;FOR bitişi
```
*LOCAL7*'nin index değişkeni olduğunu görebiliyoruz. Pseudo-code örneklerine başlayabiliriz. 
```cpp
for(int LOCAL7 = 0; LOCAL7 < input.length(); i++)
{
	//bla bla
}
```


Şimdi ikinci for döngüsünü inceleyelim.
```
00AA3471  |.  C745 CC 00000>|MOV [LOCAL.13],0            ;LOCAL13 = 0
00AA3478  |.  EB 09         |JMP SHORT CrackMe.00AA3483  ;FOR başlangıcı
00AA347A  |>  8B45 CC       |/MOV EAX,[LOCAL.13]
00AA347D  |.  83C0 01       ||ADD EAX,1
00AA3480  |.  8945 CC       ||MOV [LOCAL.13],EAX         ;üstteki üç komut ile beraber
                                                         ;LOCAL13 += 1
00AA3483  |>  8D4D D8       | LEA ECX,[LOCAL.10]
00AA3486  |.  E8 C6DFFFFF   ||CALL CrackMe.00AA1451      
00AA348B  |.  3945 CC       ||CMP [LOCAL.13],EAX
00AA348E  |.  73 32         ||JNB SHORT CrackMe.00AA34C2 ;LOCAL13 <= EAX ise döngüyü bitir
```
Burada *AA1451* fonksiyonun döndürdüğü değere göre bir döngü tekrarı var. İnceleyelim o zaman.
![](/crackme-2/cpu_j_size.png)
Ortalarda şöyle bir komut görüyoruz:
```
00AA380C   .  B8 08000000   MOV EAX,8
```
Debugging yaparak kontrol edebiliriz ki, bu fonksiyonun tek amacı 8 döndürmek. Yaratıcı burada sadece kafa karıştırmak istemiş. For döngülerimizin temelini anlamış olduk.
```cpp
for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
{
	//bla bla
	for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
	{
		//bla bla
	}
}
```

Şimdi gelelim asıl kontrol işlemlerine. İkinci for döngüsüne girmeden azıcık yukarıya bakarsak, bir fonksiyon çağırıldığını görüyoruz. 
```
00AA345D  |.  8B45 10       |MOV EAX,[ARG.3]             
00AA3460  |.  0345 E4       |ADD EAX,[LOCAL.7]           
00AA3463  |.  0FBE00        |MOVSX EAX,BYTE PTR DS:[EAX] ;üstteki 2 komut ile beraber
                                                         ;EAX = (byte)ARG3[LOCAL7]
00AA3466  |.  99            |CDQ                         ;EDX = 0
00AA3467  |.  52            |PUSH EDX
00AA3468  |.  50            |PUSH EAX
00AA3469  |.  8D4D D8       |LEA ECX,[LOCAL.10]
00AA346C  |.  E8 FFDEFFFF   |CALL CrackMe.00AA1370       ; func_AA1370(EAX, 0)
```
*AA1370* fonksiyonundan önce ECX'e atılan *LOCAL10* değişkenine dikkat edelim.  
![](/crackme-2/cpu_local10.png)
Altta gösterdiğine göre kendisi bir pointer. Peki, buradaki adreste önemli bir şey mi var? Fonksiyondan önce bakarsak, CC olduğunu görüyoruz. Pek bir anlam ifade ettiğini söyleyemem. Peki, fonksiyondan sonra bakalım? Vee *(byte)ARG3[LOCAL7]*  
Yani parametre olarak girdiğimiz EAX'in aynısı. Sadece kafa karıştırmak için yazılmış. Toparlarsak:
```cpp
string input;
for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
{
	int *LOCAL10 = func_AA1370(input[LOCAL7], 0);
	for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
	{
		//bla bla
	}
}
```

Şimdi ikinci for döngüsündeki kontrol işlemlerine bakalım. Döngünün sonunda *LOCAL4* değişkenin değiştiğini görüyoruz. 
```
00AA34AB  |.  56            ||PUSH ESI
00AA34AC  |.  51            ||PUSH ECX
00AA34AD  |.  52            ||PUSH EDX
00AA34AE  |.  50            ||PUSH EAX
00AA34AF  |.  E8 E9DEFFFF   ||CALL CrackMe.00AA139D ;func_AA139D(EAX, EDX, ECX, ESI)
00AA34B4  |.  0345 F0       ||ADD EAX,[LOCAL.4]
00AA34B7  |.  1355 F4       ||ADC EDX,[LOCAL.3]
00AA34BA  |.  8945 F0       ||MOV [LOCAL.4],EAX     ; LOCAL4 += EAX
00AA34BD  |.  8955 F4       ||MOV [LOCAL.3],EDX
```
Demek ki, son kararı veren fonksiyon *func_AA139D*. Ve belli ki, EAX döndürüyor. Peki bu fonksiyon parametrelerini belirleyen şeyler nedir? Birazcık yukarıya bakalım. 
```
00AA3490  |.  8B45 CC       ||MOV EAX,[LOCAL.13]
00AA3493  |.  50            ||PUSH EAX
00AA3494  |.  8D4D D8       ||LEA ECX,[LOCAL.10]    
00AA3497  |.  E8 F2DEFFFF   ||CALL CrackMe.00AA138E  ;func_AA138E(LOCAL13)
00AA349C  |.  0FB6C0        ||MOVZX EAX,AL
00AA349F  |.  99            ||CDQ                    ; EDX = 0
00AA34A0  |.  83F0 02       ||XOR EAX,2              ; EAX = EAX XOR 2
00AA34A3  |.  83F2 00       ||XOR EDX,0              ; EDX = EDX XOR 0 (yine 0)
00AA34A6  |.  8B4D CC       ||MOV ECX,[LOCAL.13]     ; ECX = [LOCAL13]
00AA34A9  |.  33F6          ||XOR ESI,ESI            ; ESI = 0
00AA34AB  |.  56            ||PUSH ESI
00AA34AC  |.  51            ||PUSH ECX
00AA34AD  |.  52            ||PUSH EDX
00AA34AE  |.  50            ||PUSH EAX
00AA34AF  |.  E8 E9DEFFFF   ||CALL CrackMe.00AA139D  ;func_AA139D(EAX, 0, LOCAL13, 0)
```
*EAX = EAX XOR 2* komutu gerçekleştiğine göre, EAX'in önceden belirlenmiş olması lazım. XOR komutunun birazcık yukarısında *AA138E* fonksiyonu çağırıldığını ve ardından *EAX = (byte)EAX* komutundan EAX'in belirlendiğini görüyoruz. Ayrıca EDX'in de bir değer alıyor gibi. Demekki yapmamız gereken şey, *AA138E* fonksiyonunu çözmek. Son pseudo-code'u yazalım ve yeni fonksiyonu çözmeye başlayalım. 

```cpp
string input;
for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
{
	int *LOCAL10 = func_AA1370(input[LOCAL7], 0);
	for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
	{
		ECX = LOCAL10;             
		EAX = func_AA138E(LOCAL13);
		
		EAX = EAX XOR 2;
		EDX = EDX XOR 0;
		ESI = 0;
		ECX = LOCAL13;
		
		EAX = func_AA139D(EAX, 0, ECX, 0);
		LOCAL4 += EAX;
	}
}
```
  
Çözmemiz gereken fonksiyon:
![](/crackme-2/cpu_func1.png)
İlk komutlarda önemli bir şey gözükmüyor. Şu kodları inceleyelim
```
00AA3380  |.  894D F8       MOV [LOCAL.2],ECX
00AA3383  |.  B9 27A0AB00   MOV ECX,CrackMe.00ABA027
00AA3388  |.  E8 B0E0FFFF   CALL CrackMe.00AA143D       ;Bu fonksiyonun yaptığı önemli bir şey yok
00AA338D  |.  837D 08 08    CMP [ARG.1],8             
00AA3391  |.  72 15         JB SHORT CrackMe.00AA33A8   ;ARG1 sürekli 8'den küçük olacağı için 
                                                        ;atlama her zaman gerçekleşecek  
...
...
00AA33A8  |> \8B4D 08       MOV ECX,[ARG.1]           
00AA33AB  |.  51            PUSH ECX        
00AA33AC  |.  8B4D F8       MOV ECX,[LOCAL.2]           ;ECX = LOCAL10
00AA33AF  |.  E8 37DDFFFF   CALL CrackMe.00AA10EB       ;func_AA10EB(ARG1)
00AA33B4  |.  8885 33FFFFFF MOV BYTE PTR SS:[EBP-CD],AL 
00AA33BA  |>  8A85 33FFFFFF MOV AL,BYTE PTR SS:[EBP-CD] 
```

Bu fonksiyonu şöyle yazabiliriz.
```c++
int func_AA138E(int index)
{
	if(index < 8)
	{
		return func_AA10EB(index);
	}
}
```

Şimdi yeni bir bilinmeyen fonksiyonumuz var. *AA10EB*. Bunu inceleyelim.
![](/crackme-2/cpu_func1_1.png)
Ortalarda aritmatiksel işlemler olduğunu görebiliyoruz. Burada ilk fonksiyon çağrılana kadar önemli bir şey yapılmıyor. *JE* ve *JMP* komutlarının olduğu yerde ise *LOCAL52* değişkenine 1 veya 0 değeri atılıyor ve bu değer AL'ye atılarak döndürülmüş oluyor. (AL'ye atılan değer *[EBP-D0]* olarak gözüküyor ancak orada küçük bir hata olmuş)  
Şimdi aritmatiksel işlemleri inceleyelim
```
00AA35CD  |.  8B75 08       MOV ESI,[ARG.1]   ;ESI = [ARG1]
00AA35D0  |.  C1EE 05       SHR ESI,5         ;ESI = ESI >> 5
00AA35D3  |.  8B45 08       MOV EAX,[ARG.1]   ;EAX = [ARG1]
00AA35D6  |.  33D2          XOR EDX,EDX       ;EDX = 0
00AA35D8  |.  B9 20000000   MOV ECX,20        ;ECX = 0x20
00AA35DD  |.  F7F1          DIV ECX           ;EAX = EAX / ECX
                                              ;EDX = EAX % ECX
00AA35DF  |.  B8 01000000   MOV EAX,1         ;EAX = 1
00AA35E4  |.  8BCA          MOV ECX,EDX       ;ECX = EDX
00AA35E6  |.  D3E0          SHL EAX,CL        ;EAX = EAX << CL     
00AA35E8  |.  8B4D F8       MOV ECX,[LOCAL.2] ;ECX = [LOCAL2]    [LOCAL2] = [LOCAL10]  
00AA35EB  |.  2304B1        AND EAX,DWORD PTR DS:[ECX+ESI*4]    ;EAX = EAX & [ECX + ESI * 4]   
00AA35EE  |. /74 0C         JE SHORT CrackMe.00AA35FC  ;EAX == 0 ise bu adrese git
00AA35F0  |. |C785 30FFFFFF>MOV [LOCAL.52],1
00AA35FA  |. |EB 0A         JMP SHORT CrackMe.00AA3606
00AA35FC  |> \C785 30FFFFFF>MOV [LOCAL.52],0
00AA3606  |>  8A85 30FFFFFF MOV AL,BYTE PTR SS:[EBP-D0]
```

Biraz karışık gözüktüğünün farkındayım. Önce bunları bir yere yazalım.

```
ESI = [ARG1]
ESI = ESI >> 5
EAX = [ARG1]
EDX = 0
ECX = 0X20
EAX = EAX / ECX
EDX = EAX % ECX
EAX = 1
ECX = EDX
EAX = EAX << CL 
ECX = [LOCAL2] 
EAX = EAX & [ECX + ESI * 4]
```

ESI başta *ARG1* değerini alıyor. Daha sonra 5 bit sağa kaydırılıyor. Burada şöyle bir gerçek var ki, *ARG1* değerimiz 0 ile 7 arasında bir sayı olacağı için, bu işlem sonucu her türlü 0 olacaktır. ESI ileriki adımlarda değişmediği için ESI yazan yerlere 0 yazabiliriz ya da silebiliriz. Yeni işlem sırası böyle oldu.

```
ESI = 0
EAX = [ARG1]
EDX = 0
ECX = 0X20
EAX = EAX / ECX
EDX = EAX % ECX
EAX = 1
ECX = EDX
EAX = EAX << CL 
ECX = [LOCAL2] 
EAX = EAX & [ECX]
```

EAX, *ARG1* değerini alıyor. ECX, 0x20 değerini alıyor. Daha sonra EAX, ECX'e bölünüyor. EDX ise kalan sayıyı alıyor. Ancak yine bir gerçek var ki, *ARG1* değerimiz 0 ile 7 arasında bir sayı olacağı için EDX her zaman *ARG1* değerini almış olacak. EAX ise 0 olacak. EDX daha sonra değişmediği için  EDX yazan yerlere *ARG1* yazabiliriz. Yeni işlem sırası:

```
EDX = [ARG1]
ECX = 0X20
EAX = 1
ECX = EDX
EAX = EAX << CL 
ECX = [LOCAL2] 
EAX = EAX & [ECX]
```

EAX'e 1 atandıktan sonra, *ECX = EDX* oluyor ve EAX = *EAX << CL* aritmatiksel işlemi gerçekleşiyor. Burayı şöyle kısaltabiliriz.

```
EDX = [ARG1]
EAX = 1 << EDX 
ECX = [LOCAL2] 
EAX = EAX & [ECX]
```

ECX'e *LOCAL2* değişkeni atanıyor. *LOCAL2* bir pointer'dı ve for döngüsündeki *[LOCAL10]* değişkenine eşitti. Bu demek oluyor ki, kontrol edilen harf *a* ise, *LOCAL2*'de yer alan değer bunun ASCII karşılığıdır. (0x61)  
Biraz daha sadeleştirebiliriz:
```
EAX = 1 << [ARG1]
EAX = EAX & [LOCAL2] 
```

Son işlemden sonra *EAX == 0* ise 0 döndürüyor. 0'dan farklı ise 1 döndürüyor. Pseudo-code olarak yazarsak şöyle olur:
```c++
int func_AA10EB(int index)
{
	int a = 1 << index  //EAX = pow(2, index]);
	if(a == 0)
	{
		return 0;
	}
	return 1;
}
```

Bu fonksiyonu tamamladık. Yaptığımız kadarını yazarsak:
```c++
string input;
for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
{
	int *LOCAL10 = func_AA1370(input[LOCAL7], 0);
	for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
	{
		ECX = LOCAL10;             
		EAX = func_AA138E(LOCAL13);
		EAX = EAX XOR 2;
		EDX = 0;
		ESI = 0;
		ECX = LOCAL13;
		
		EAX = func_AA139D(EAX, 0, ECX, 0);
		LOCAL4 += EAX;
	}
}

int func_AA138E(int index)
{
	if(index < 8)
	{
		return func_AA10EB(index);
	}
}

int func_AA10EB(int index)
{
	int a = 1 << index  //EAX = pow(2, index]);
	if(a == 0)
	{
		return 0;
	}
	return 1;
}
```

Baştaki for döngülerine döndük. Artık ilk EAX'in ne olduğunu biliyoruz. Şimdi sıra geldi alttaki fonksiyona. *AA139D*
![](/crackme-2/cpu_func2.png)
Kısa bir fonksiyona benziyor. OllyDbg yeteri kadar iyi analiz edemedi ancak şunları yazabiliriz ki:  
[ESP + 0x4]  = [ARG1]  
[ESP + 0x8]  = [ARG2]  
[ESP + 0xC]  = [ARG3]  
[ESP + 0x10] = [ARG4] 
  
 
Kodların ilk RETN komutuna kadar olan kısmını *[ARG]* şeklinde düzenleyerek inceleyelim:
```
00AAAD30   > \8B4424 08     MOV EAX, [ARG2]
00AAAD34   .  8B4C24 10     MOV ECX, [ARG4]
00AAAD38   .  0BC8          OR ECX,EAX                  ;ECX = [ARG2] | [ARG4]
00AAAD3A   .  8B4C24 0C     MOV ECX, [ARG3]
00AAAD3E   .  75 09         JNZ SHORT CrackMe.00AAAD49  ;ECX != 0 ise adrese git
00AAAD40   .  8B4424 04     MOV EAX, [ARG1]             ;EAX = [ARG1]
00AAAD44   .  F7E1          MUL ECX                     ;EAX = EAX * ECX 
00AAAD46   .  C2 1000       RETN 10
```
İlk 3 komuttan anlayacağımız üzere, *ECX = [ARG2] | [ARG4]* oluyor. Burada Z bayrağımız bir değere sahip oluyor. Daha sonra *ECX = [ARG3]* komutu işleniyor, *ECX == 0* ise atlama işlemi gerçekleşmiyor. Burayı şöyle bir koda dökebiliriz
```c++
int func_AA139D(int ARG1, int ARG2, int ARG3, int ARG4) //ARG2 = 0;  ARG4 = 0
{
	ECX = [ARG2] | [ARG4];
	if(ECX == 0)
	{
		EAX = [ARG3] * [ARG1];
		return EAX;
	}
}

```
İlk satırdaki veya işlemine odaklanalım. *ARG2* ve *ARG4* her zaman 0 idi. Yani bu işlemin sonucu her zaman 0 olacak ve if koşulunu sağlıyor olacağız. Şöyle yazabiliriz:
```c++
int func_AA139D(int ARG1, int ARG2, int ARG3, int ARG4)
{
	return ([ARG3] * [ARG1]);
}
```

Kodların tamamını tekrar yazarsak:
```c++
string input;
for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
{
	int *LOCAL10 = func_AA1370(input[LOCAL7], 0);
	for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
	{
		ECX = LOCAL10;             
		EAX = func_AA138E(LOCAL13);
		EAX = EAX XOR 2;
		EDX = 0;
		ESI = 0;
		ECX = LOCAL13;
		
		EAX = func_AA139D(EAX, 0, ECX, 0);
		LOCAL4 += EAX;
	}
}

int func_AA138E(int index)
{
	if(index < 8)
	{
		return func_AA10EB(index);
	}
}

int func_AA10EB(int index)
{
	int a = 1 << index  //EAX = pow(2, index]);
	if(a == 0)
	{
		return 0;
	}
	return 1;
}

int func_AA139D(int ARG1, int ARG2, int ARG3, int ARG4) //ARG2 = 0;  ARG4 = 0
{
	return ([ARG3] * [ARG1]);
}
```


Bitti satılır, for döngüsünü biraz sadeleştirip döngü sonrası kontrol işlemini de eklersek şöyle olur:
```c++
int func_AA1339(int a, int b, string input) a = 0x55C;   b = 0
{
	string input;
	for(int LOCAL7 = 0; LOCAL7 < input.length(); LOCAL7++)
	{
		int *LOCAL10 = func_AA1370(input[LOCAL7], 0);
		for(int LOCAL13 = 0; LOCAL13 < 8; LOCAL13++)
		{          
			EAX = func_AA138E(LOCAL13);
			EAX = EAX XOR 2;
			EAX = func_AA139D(EAX, 0, LOCAL10, 0);
			LOCAL4 += EAX;
		}
	}
	if([LOCAL4] == a)
		return 0;
}


int func_AA138E(int index)
{
	if(index < 8)
	{
		return func_AA10EB(index);
	}
}

int func_AA10EB(int index)
{
	int a = 1 << index  //EAX = pow(2, index]);
	if(a == 0)
	{
		return 0;
	}
	return 1;
}

int func_AA139D(int ARG1, int ARG2, int ARG3, int ARG4) //ARG2 = 0;  ARG4 = 0
{
	return ([ARG3] * [ARG1]);
}
```


  
  
  
  
Örnek bir string olarak şunu deneyebilirsiniz: ~~~~~~~~~~~~~~~~~B
