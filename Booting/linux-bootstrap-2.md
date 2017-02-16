�ekirdek �ny�kleme S�reci B�l�m 2

�ekirdek Kurulumunun �lk Ad�mlar�

�nceki b�l�mde linux �ekirde�i i�erisine dalmaya ba�lad�k ve �ekirdek kurulum kodunun ilk b�l�m�n� g�rd�k. Arch/x86/boot/main.c'den main fonksiyonuna (C'de yaz�lan ilk fonksiyon) ilk �a�r�da durduk.
Bu b�l�mde, �ekirdek kurulum kodunu ara�t�rmaya ve 
- korumal� modun ne oldu�unu, 
- ge�i� i�in baz� haz�rl�klar�, 
- heap ve konsol ba�latma, 
- bellek alg�lama, CPU do�rulama, klavye ba�latma 
- ve �ok daha fazlas�n� g�rece�iz. 

�yleyse, devam edelim.

Korumal� Mod

Yerel Intel64 Long Mode'a ge�meden �nce, �ekirdek CPU'yu korumal� moda ge�irmelidir.

Korumal� mod nedir? Korumal� mod, ilk olarak 1982'de x86 mimarisine eklendi ve 80286 i�lemciden Intel 64'e ve Long Mode'a gelene kadar ana i�lemci modlar�yd�.

 Real Mode'dan uzakla�man�n ba�l�ca nedeni RAM'e �ok s�n�rl� eri�im olmas�d�r. �nceki b�l�mden hat�rlad���n�z gibi, Real Mode'da yaln�zca 2 �zeri 20 byte veya 1 Megabayt, bazen sadece 640 Kilobayt RAM mevcut.

 Korumal� mod bir�ok de�i�iklik getirdi, ancak ana farklardan biri bellek y�netimindeki farkt�r. 20 bit adres veriyolu yerine 32 bitlik adres veri yolu getirildi. 4 Gigabyte bellek ve 1 Megabayt ger�ek moda eri�ime izin verildi. Ayr�ca, sonraki b�l�mlerde okuyabilece�iniz sayfalama deste�i eklendi.

Korumal� modda bellek y�netimi, neredeyse ba��ms�z iki par�aya b�l�n�r:

- B�l�tleme
- Sayfalama
Burada sadece b�l�tlemeyi g�rece�iz. Disk belle�i bir sonraki b�l�mde ele al�nacakt�r. 

�nceki b�l�mde okuyabilece�iniz gibi, adresler ger�ek modda iki k�s�mdan olu�ur: 

- Segmentin taban adresi
- Segment taban�ndan ofset

Ve e�er bu iki par�ay� biliyorsak fiziksel adresi bulabiliriz:

            PhysicalAddress = Segment Selector * 16 + Offset


Bellek b�l�mlemesi korumal� modda tamamen yeniden yap�land�r�ld�. 64 Kilobayt sabit boyutlu b�l�m yok. Bunun yerine, her bir b�l�m�n boyutu ve konumu, B�l�m Tan�mlay�c�lar� ad� verilen ili�kili bir veri yap�s� taraf�ndan tan�mlan�r. B�l�m tan�mlay�c�lar�, Genel Tan�mlay�c� Tablosu (GDT) ad� verilen bir veri yap�s�nda saklan�r.

GDT, haf�zada bulunan bir yap�d�r. Belle�inde sabit bir yeri yoktur, b�ylece adresi GDTR �zel kayd�nda saklan�r. Daha sonra, GDT'nin Linux �ekirde�i kodunda y�klendi�ini g�rece�iz. Belle�e y�klemek a�a��daki gibi bir i�lem olacak:

            lgdt gdt

Burada lgdt komutu global tan�mlay�c� tablosunun taban adresini ve s�n�r�n� (b�y�kl���) GDTR kayd�na y�kler. GDTR, 48 bitlik bir kay�tt�r ve iki b�l�mden olu�ur:

- Genel tan�mlay�c� tablosunun boyutu (16-bit);
- K�resel tan�mlay�c� tablosunun adresi (32-bit).

Yukar�da belirtildi�i gibi GDT, bellek b�l�mlerini tan�mlayan b�l�m tan�mlay�c�lar� i�erir. Her tan�mlay�c� 64-bit boyutundad�r. Bir tan�mlay�c�n�n genel �emas� ��yledir:

         31          24        19      16              7            0
         ------------------------------------------------------------
         |             | |B| |A|       | |   | |0|E|W|A|            |
         | BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 | 4
         |             | |D| |L| 19:16 | |   | |1|C|R|A|            |
         ------------------------------------------------------------
         |                             |                            |
         |        BASE 15:0            |       LIMIT 15:0           | 0
         |                             |                            |
         ------------------------------------------------------------

Endi�elenmeyin, Real Mode'dan sonra biraz korkutucu oldu�unu biliyorum, ama kolay. �rne�in LIMIT 15: 0, Tan�mlay�c�n�n 0-15 bitinin limit de�eri i�erdi�i anlam�na gelir. Geri kalan k�sm� LIMIT 19: 16'da. Yani Limit boyutu 0-19, yani 20 bittir. �imdi ona bir g�z atal�m:

   S�n�r [20 bit] 0-15,16-19 bit. Bu, length_of_segment - 1'i tan�mlar. G (Granularity) bitine ba�l�d�r.
   - G (bit 55) 0 ise ve segment s�n�r� 0 ise, segmentin boyutu 1 Byte'dir
   - G; 1 ve segment s�n�r� 0 ise, segmentin boyutu 4096 Byte'dir
   - G; 0 ve segment s�n�r� 0xfffff ise, segmentin boyutu 1 Megabayt
   - G; 1, segment s�n�r� 0xfffff ise, segmentin boyutu 4 Gigabyte'd�r

Bu demektir ki, e�er



