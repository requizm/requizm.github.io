+++
author = "requizm"
title = "Crackme-1"
date = "2020-06-10"
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

Selam, yeni bir seriye başlamak istedim. Bu seride yapmak istediğim şey, CrackMe uygulamalarının çözümlerini göstermek.
  
  
### Uygulama Bilgileri  
[Uygulama Linki](https://crackmes.one/crackme/5ed3735f33c5d449d91ae6a9 )  
**Yaratıcı:** SD4RK  
**Zorluk:** Çok Kolay
  
  
### Kullandığım Programlar  
**OllyDbg 1.10:** Dinamik Analiz için.  
  
  
  
### Başlangıç
Öncelikle programımızı tanıyalım. Basit bir konsol ekranında bizden veri girişi bekliyor, dana sonra girdiğimiz değerin doğru veya yanlış olduğuna dair bir mesaj vererek program bitiyor.
![](/crackme-1/wrong_1.png)
  
  
Programımızı ollydbg ile açalım. Veri giriş ve kontrol işleminin gerçekleştiği yerleri bulalım. Birden fazla yol ile bulabiliriz.  
-**String Search:** Konsolda gördüğümüz stringlerden herhangi birini arayabiliriz.   
-**API Breakpoint:** Veri girişi fonksiyonu ile alakalı API fonksiyonlarına breakpoint koymak.
  
  
String search daha kolay olduğu için onu yapalım. Dump penceresindeyken sağ tıklayıp *Search For -> Binary String* diyerek, aramak istediğimiz string metninin bir kısmını veya tamamını ASCII bölümüne yazıp *Okey* diyelim.
![](/crackme-1/dump_string_enter.png)
String'in olduğu yere gitmiş olduk, artık bu stringe nereden erişildiğine bakarak aradığımız kod satırlarını bulabiliriz. Bulduğum string'in ilk harfini seçip, sağ tılayıp *Find Referances* diyerek nerelerde kullanıldığına bakabiliriz. 
![](/crackme-1/dump_findreferances.png)
Buradaki adrese bakalım:
![](/crackme-1/cpu_ex1.png)
EDX'e *Enter The Key* string'i atılıyor ve bir fonksiyon çağırılıyor. Ardından başka bir fonksiyon çağırılıyor. Debugging yaparak görebiliriz ki, ilk fonksiyon sadece konsola *Enter The Key* yazdırıyor, ikincisi ise bizden veri girişi bekliyor.   
İkinci fonksiyona breakpoint koyarak girdiğimiz verinin hangi adrese kayıt olduğunu ve sonradan nasıl kontrol işlemlerinden geçtiğini görelim. *abcde* girişi yaptım. 
```
00A720AD   .  BA 74A9A900   MOV EDX,CrackMe.00A9A974        ;  ASCII "Enter the key: "
00A720B2   .  C745 FC 00000>MOV DWORD PTR SS:[EBP-4],0              
00A720B9   .  E8 E2030000   CALL CrackMe.00A724A0           ; console.write(edx)
00A720BE   .  8D55 D8       LEA EDX,DWORD PTR SS:[EBP-28]
00A720C1   .  B9 78E0A900   MOV ECX,CrackMe.00A9E078
00A720C6   .  E8 150B0000   CALL CrackMe.00A72BE0           ; veri girişi fonksiyonu
00A720CB   .  837D EC 10    CMP DWORD PTR SS:[EBP-14],10             
00A720CF   .  8D5D D8       LEA EBX,DWORD PTR SS:[EBP-28]
00A720D2   .  8B7D D8       MOV EDI,DWORD PTR SS:[EBP-28]
00A720D5   .  0F43DF        CMOVNB EBX,EDI
00A720D8   .  837D E8 10    CMP DWORD PTR SS:[EBP-18],10    ; oops, burada bir koşullu atlama yeri kodu bulduk. 
00A720DC   .  75 6D         JNZ SHORT CrackMe.00A7214B      ; bakalım A7214B adresinde ne var
```

![](/crackme-1/cpu_a7214b.png)

Gördüğümüz gibi, EDX'e girilen verinin yanlış olduğuna dair bir string atılıyor ve yazdırılıyor. Yani buraya gelmemeliyiz. Peki, yukarıdaki kontrol işleminin tam olarak ne yaptığını görmek için oraya breakpoint koyalım ve inceleyelim.
![](/crackme-1/cpu_brk_sizecontrol.png)
*[EBP-18]* adresindeki değerin 0x10 olup olmadığını kontrol ediyor. 0x10 değilse, kötü yere atlıyor. Peki kontrol edilen değer ne? Üstteki fotoğrafın altında gösterdiğine göre 5 olduğunu gösteriyor. Girilen değerimiz *abcde* idi. Tesadüfe bakın ki, string uzunluğu da 5. Yani burada girilen verinin uzunluğunun kontrol edildiğini gördük. 0x10 uzunluğunda bir veri girmemiz gerektiğini öğrendik. Onluk sistemde 16 karakter ediyor.   
Programı yeniden başlatarak, *0123456789abcdef* verisini girdim. İlk kontrol aşamasını atlattım. Şimdi asıl kontrol işlemlerine geçiyoruz. Geçmeden önce, birazcık aşağıdaki kodların genel bir görüntüsüne bakalım.
![](/crackme-1/cpu_kontrol_genelgoruntu.png)
Kontrol işlemleri buralarda gerçekleşiyor olmalı. Çok fazla koşullu atlama kodu var ve bu atlamaların çoğu girilen verinin doğru veya yanlış olduğuna dair yere gidiyor.  
  
Son kaldığımız yerden debugging yaparak devam edelim. 
```
00A720DE   .  33F6          XOR ESI,ESI                   ; ESI = 0
00A720E0   .  C745 D4 03000>MOV DWORD PTR SS:[EBP-2C],3   ; [EBP-2C] = 0x3
00A720E7   .  33C9          XOR ECX,ECX                   ; ECX = 0
```
Bu kodlar bir döngü hazırlığı gibi duruyor. Bu kodları direk geçelim. 
![](/crackme-1/cpu_al_letter.png)
Gördüğümüz gibi, AL'e [ECX + EBX] değerini attık. Register penceresinden gördüğümüz üzere EBX string adresi, ECX ise muhtemelen kontrol ettiğimiz index. Az önce sıfırlamıştık zaten.  
Peki ECX nerede artıyor?
```
00A7213E   > \41            INC ECX       
00A7213F   .  83F9 0F       CMP ECX,0F
```
*A7213E* adresinde ECX'i bir arttırıp 0xF'ten küçük mü diye kontrol ediyoruz. Küçük ise, yukarıdaki *AL, [ECX + EBX]* adresine tekrar gidip sonraki harf için kontrol işlemlerini gerçekleştiriyoruz.
  
  
  
Şimdilik ECX = 0 olduğu için [AFB3E8 + 0] değerini kontrol ediyoruz. Hex olarak 0x30, ASCII tablosuna göre "0" karakterine işaret ediyor. Yani girdiğimiz verinin ilk değeri. Daha sonra AL'i EDX'e atıyoruz ve EDX'te yine 0x30 değeri atılmış oluyor.  
Şimdi koşullu bir atlama yerine geldik.
```
00A720F6   .  83F9 05       CMP ECX,5                    
00A720F9   .  7F 06         JG SHORT CrackMe.00A72101
00A720FB   .  24 E0         AND AL,0E0
00A720FD   .  3C 20         CMP AL,20
00A720FF   .  EB 3B         JMP SHORT CrackMe.00A7213C
00A72101   >  83F9 09       CMP ECX,9
...
...
00A7213C   > \75 0D         JNZ SHORT CrackMe.00A7214B
00A7213E   > \41            INC ECX                                
00A7213F   .  83F9 0F       CMP ECX,0F
00A72142   .^\7C AC         JL SHORT CrackMe.00A720F0
...
...
00A7214B   > \BA A8A9A900   MOV EDX,CrackMe.00A9A9A8                 ;  ASCII 0A,"No, this o"
00A72150   >  E8 4B030000   CALL CrackMe.00A724A0
```

Burada ECX 5'ten büyükse, *A72101* adresine atlamayı, değilse devam ederek bazı işlemler yaptırıyor. Önce AL değeri ile 0xE0 değerini *and* işlemine sokuyoruz, çıkan değer 20'den farklı ise yanlış sonuca gidiyoruz. Değil ise ECX'i 1 arttırıp sonraki değer için aynı kontroller tekrar gerçekleştiriliyor. Bu kontrol işlemi ECX = 5 dahil olarak 5'e kadar gerçekleşecek. Bu kodları pseudo-code formatında yazarsak şöyle olur:
```c
AL = [ECX + EBX] "0" "1" "2"
EDX = AL
if (ECX <= 5)
{
	AL = AL && E0
	if(AL == 0x20)
	{
		ECX++
		if(ECX == 0xF)
		{
			good_boy
		}
		else
		{
			başa dön
		}
	}
	else
	{
		bad_boy
	}
}
```
  

ECX = 6 olana kadar ilerletelim. Sonraki kodları inceleyelim.

```
00A72101   > \83F9 09       CMP ECX,9
00A72104   .  7D 04         JGE SHORT CrackMe.00A7210A
00A72106   .  03F2          ADD ESI,EDX                            
00A72108   .  EB 34         JMP SHORT CrackMe.00A7213E
00A7210A   >  75 13         JNZ SHORT CrackMe.00A7211F
...
...
00A7213E   > \41            INC ECX                
00A7213F   .  83F9 0F       CMP ECX,0F
00A72142   .^\7C AC         JL SHORT CrackMe.00A720F0
```
Burada ECX >= 9 ise ileri atla diyor. Önce ECX'in küçük olduğu durumları anlayalım. Burada *ESI = ESI + EDX* diyoruz, ardından ECX'i arttırıp döngünün tekrar başına dönüyoruz. ESI değerimiz başta sıfırlamıştık. Bu kodu şöyle yazabiliriz:
```c
else if(ECX < 9 )
{
	ESI = ESI + EDX	
	ECX++
	if(ECX == 0xF)
	{
		good_boy
	}
	else
	{
		başa dön
	}
}
```
Okunabilirliği arttırmak için en baştan küçük bir kod düzenlemesi yapalım:
```c
ESI = 0   
ECX = 0  ; index
EBX = input_string
while(true)
{
	AL = [EBX + ECX] ; "0" "1" "2"
	EDX = AL  

	if (ECX <= 5)
	{
		AL = AL && E0
		if(AL != 0x20)
		{
			bad_boy
		}
	}
	else if(ECX < 9 )
	{
		ESI = ESI + EDX
	}
	
	ECX++
	if(ECX >= 0F)
	{
		good_boy
	}
}
```
  
  
Şimdi ECX'in 9 ve 9 dan büyük olduğu durumlara bakalım.

```
00A72101   > \83F9 09       CMP ECX,9
00A72104   .  7D 04         JGE SHORT CrackMe.00A7210A  ;  
...
...
00A7210A   > \75 13         JNZ SHORT CrackMe.00A7211F  ; 9'a eşit değil ise atla
00A7210C   .  8BC6          MOV EAX,ESI                              
00A7210E   .  25 00FFFFFF   AND EAX,FFFFFF00
00A72113   .  3D 00010000   CMP EAX,100
00A72118   .  75 31         JNZ SHORT CrackMe.00A7214B   ; EAX != 0x100 ise atla
00A7211A   .  80FA 40       CMP DL,40
00A7211D   .  EB 1D         JMP SHORT CrackMe.00A7213C
...
...
00A7213C   > \75 0D         JNZ SHORT CrackMe.00A7214B   ; DL != 0x40 ise atla
00A7213E   >  41            INC ECX                                  
00A7213F   .  83F9 0F       CMP ECX,0F
00A72142   .^ 7C AC         JL SHORT CrackMe.00A720F0
...
...
00A7214B   > \BA A8A9A900   MOV EDX,CrackMe.00A9A9A8    ;  ASCII 0A,"No, this o"
00A72150   >  E8 4B030000   CALL CrackMe.00A724A0
```
Öncelikle ECX'in 9 olduğu durumu inceleyelim. *0A7210C* adresine geliyor, ESI değerimizi EAX'e atıyoruz ve EAX ile FFFFFF00 değerini *and* işlemine sokuyoruz. Sonuç 0x100'den farklı ise yanlış sonuca atlıyoruz. ESI değerimizi bir önceki koşul kontrolünde doldurmuştuk. Her şey bununla bitmiyor, EAX == 0x100 olduktan sonra *CMP DL, 0x40* komut satırı ile, DL(EDX) 0x40'tan farklı ise yanlış sonuca atlıyoruz. Yani buradaki amacımız, EAX'in belli işlemler sonucu 0x100 olmasını sağlamak ve DL'nin 0x40 yapmak. (0x40 = "@")
```c
ESI = 0   
ECX = 0  ; index
EBX = input_string
while(true)
{
	AL = [EBX + ECX] ; "0" "1" "2"
	EDX = AL  

	if (ECX <= 0x5)
	{
		AL = AL && 0xE0
		if(AL != 0x20)
		{
			bad_boy
		}
	}
	else if(ECX < 0x9 )
	{
		ESI = ESI + EDX
	}
	else if (ECX == 9)
	{
		EAX = ESI
		EAX = EAX && 0xFFFFFF00
		
		if(EAX != 0x100 && DL != 0x40)
		{
			bad_boy
		}
	}

	ECX++
	if(ECX >= 0F)
	{
		good_boy
	}
}
```
  
  
Şimdi ECX'in 9'dan büyük olduğu durumlara bakalım.
```
00A72101   > \83F9 09       CMP ECX,9
00A72104   .  7D 04         JGE SHORT CrackMe.00A7210A
...
...
00A7210A   > \75 13         JNZ SHORT CrackMe.00A7211F
...
...
00A7211F   > \83F9 0D       CMP ECX,0D
00A72122   .  7C 0F         JL SHORT CrackMe.00A72133      ; ECX < 0xD ise atla
...
...
00A72133   > \8D0412        LEA EAX,DWORD PTR DS:[EDX+EDX] ; EAX = EDX + EDX       
00A72136   .  99            CDQ
00A72137   .  F77D D4       IDIV DWORD PTR SS:[EBP-2C]     ; EAX = EAX / [EBP-2C]
                                                           ; EDX = EAX % [EBP-2C]
00A7213A   .  85D2          TEST EDX,EDX                   
00A7213C   >  75 0D         JNZ SHORT CrackMe.00A7214B     ; EDX != 0 ise atla
00A7213E   > \41            INC ECX                                  
00A7213F   .  83F9 0F       CMP ECX,0F
00A72142   .^ 7C AC         JL SHORT CrackMe.00A720F0
...
...
00A7214B   > \BA A8A9A900   MOV EDX,CrackMe.00A9A9A8    ;  ASCII 0A,"No, this o"
00A72150   >  E8 4B030000   CALL CrackMe.00A724A0
```
Önce ECX'in 0xD'den küçük mü diye kontrol ediyoruz. Küçük ise, *EAX = EDX + EDX* diyoruz ve EAX'i [EBP-2C] değerine bölüyoruz. Komut gereği sonuç EAX'e, kalan EDX'e yazılıyor. Daha sonra *EDX != 0* ise kötü sonuca atlıyoruz. Değil ise ECX'i bir arttırıp sonraki döngüye geçiyoruz. 
```c
ESI = 0   
ECX = 0  ; index
EBX = input_string
while(true)
{
	AL = [EBX + ECX] ; "0" "1" "2"
	EDX = AL  

	if (ECX <= 0x5)
	{
		AL = AL && 0xE0
		if(AL != 0x20)
		{
			bad_boy
		}
	}
	else if(ECX < 0x9 )
	{
		ESI = ESI + EDX
	}
	else if (ECX == 9)
	{
		EAX = ESI
		EAX = EAX && 0xFFFFFF00
		
		if(EAX != 0x100 && DL != 0x40)
		{
			bad_boy
		}
	}
	else if(ECX < D)
	{
		EAX = EDX + EDX
		if(EAX % 3 != 0)
		{
			bad_boy
		}
	}
	ECX++
	if(ECX >= 0F)
	{
		good_boy
	}
}
```
  
  
ECX'in 0xD'ye geldiği durumları inceleyelim.
```
00A7211F   > \83F9 0D       CMP ECX,0D
00A72122   .  7C 0F         JL SHORT CrackMe.00A72133   ; ECX < 0xD ise atla
00A72124   . |8BC2          MOV EAX,EDX                 ; EAX = EDX       
00A72126   . |99            CDQ                         ; EDX = 0
00A72127   . |2BC2          SUB EAX,EDX                 ; EAX = EAX - 0
00A72129   . |D1F8          SAR EAX,1                   ; EAX = EAX >> 1
00A7212B   . |83E0 F0       AND EAX,FFFFFFF0            ; EAX = EAX && FFFFFFF0
00A7212E   . |83F8 20       CMP EAX,20              
00A72131   . |EB 09         JMP SHORT CrackMe.00A7213C   
...
...
00A7213C   > \75 0D         JNZ SHORT CrackMe.00A7214B  ; EAX != 20 ise atla
00A7213E   >  41            INC ECX                                 
00A7213F   .  83F9 0F       CMP ECX,0F
00A72142   .^ 7C AC         JL SHORT CrackMe.00A720F0
00A72144   .  BA 84A9A900   MOV EDX,CrackMe.00A9A984     ;  ASCII 0A,"Good job! "
00A72149   .  EB 05         JMP SHORT CrackMe.00A72150
```
Bu kod satırında yapılan şeyleri şöyle yazabiliriz:
```c
ESI = 0   
ECX = 0  ; index
EBX = input_string
while(true)
{
	AL = [EBX + ECX] ; "0" "1" "2"
	EDX = AL  

	if (ECX <= 5)
	{
		AL = AL && E0
		if(AL == 0x20)
		{
			bad_boy
		}
	}
	else if(ECX < 9 )
	{
		ESI = ESI + EDX
		
	}
	else if (ECX == 9)
	{
		EAX = ESI
		EAX = EAX && FFFFFF00
		
		if(EAX != 0x100 && DL != 0x40)
		{
			bad_boy
		}
	}
	else if(ECX < D)
	{
		EAX = EDX + EDX
		if(EAX % 3 != 0)
		{
			bad_boy
		}
	}
	else
	{
		EAX = EDX	
		EAX = EAX >> 1
		EAX = EAX && FFFFFFF0
		if(EAX != 0x20)
		{
			bad_boy
		}
	}
	
	ECX++
	if(ECX >= 0F)
	{
		good_boy
	}
}
```
  
  
  
  
Kodumuzun son hali böyle oldu. Ancak yer örnek bir string olarak şunu deneyebilirsiniz: *012345dd8@fff@@@*

Kendim manuel olarak çözdüm ancak böyle kodları manuel olarak çözmek biraz zaman kaybıdır. Başlangıçta IDA gibi statik analiz programlarında analiz etmek tavsiye edilir. 
