Daha onceki blog yazilarimi okuduysaniz, bir suredir low level programlamayla ilgilendigimi gorursunuz. Linux icin x86_64 Assembly programlamayla ilgili yaz�lar yazd�m ayn� zamanda Linux kaynak koduna dalmaya basladim. low-level programlarin; nas�l isledigini, bilgisayar�mda nas�l calistigini, bellekte nasil yer aldiklarini, cekirdegin surecleri ve bellegi nasil yonettigini, network stack'in low-level'da nasil calistigini ve diger pek cok seyi anlamaya buyuk bir ilgim var. Bu nedenle, x86_64 icin Linux cekirdegi hakk�nda bir dizi yazi yazmaya karar verdim. 
Profesyonel bir cekirdek hacker'� degilim. Cekirdek kodu yazmak benim gercek isim degil. Bu benim icin sadece bir hobi. Low-level'dan hoslaniyorum ve bunun nasil calistigini gormek ilgimi cekiyor. Bu nedenle kafa karistirici bir sey gorurseniz veya herhangi bir sorunuz/fikriniz varsa, @0xAX twitter hesabimban, mail yoluyla veya GitHub'da issue olusturarak bana ulasabilirsiniz. Buna minnettar olurum. Butun yazilarim linux-insides GitHub sayfasindan erisilebilir olacak. �ngilizce dil bilgimle veya yazi icerigi ile ilgili bir hata fark ederseniz, Pull Request gondermekten cekinmeyin. 
Unutmayin ki; bu resmi bir dokumantasyon degildir, sadece ogrendiklerimi paylasiyorum. 
Size gerekli olan beceriler;
- C programlama dili bilgisi
- Assembly kod bilgisi (AT&T soz dizimi)
Bazi araclar� ogrenmeye baslarsaniz, yazilarim s�ras�nda bazi k�s�mlari zaten aciklamaya calisacagim. Pekala, giri� k�sm�n burada son buluyor. �imdi �ekirdek ve low-level'a dalmaya ba�layabiliriz. 
Kodlar�n tamam� asl�nda 3.18 �ekirde�i i�in. De�i�iklikler olursa yaz�lar�m� buna g�re g�ncelleyece�im. 
Sihirli G�� D��mesi, Sonras�nda Neler Oluyor?
Bu Linux �ekirde�i ile ilgili bir dizi yaz� olsa da, �ekirdek kodundan ba�layaca��z - en az�ndan bu paragrafta. Diz�st� veya masa�st� bilgisayar�n�zdaki sihirli g�� d��mesine bast���n�z anda �al��maya ba�lar. Anakart g�� kayna��na bir sinyal g�nderiyor. Sinyali ald�ktan sonra g�� kayna�� bilgisayara do�ru miktarda elektrik sa�lar. Anakart g�� iyi sinyalini ald�ktan sonra, CPU'yu ba�latmaya �al���r. ��lemci t�m kalan veriyi kay�tlar�nda s�f�rlar ve her biri i�in �nceden tan�mlanm�� de�erleri ayarlar.
80386 ve sonraki CPU'lar, bilgisayar s�f�rland�ktan sonra CPU kay�tlar�nda a�a��daki �nceden tan�ml� verileri tan�mlar:

IP          0xfff0
CS selector 0xf000
CS base     0xffff0000


��lemci Real Mode'da �al��maya ba�lar. Biraz geriye d�nelim ve bu modda bellek b�l�tlemeyi anlamaya �al��al�m. Real Mode, t�m x86 uyumlu i�lemcilerde, 8086'dan modern Intel 64 bit CPU'lara kadar desteklenir. 8086 i�lemci, 20 bitlik bir adres veri yoluna sahiptir, bu da 0-0x100000 adres alan� (1 megabayt) ile �al��abilece�i anlam�na gelir. Ancak, yaln�zca maksimum 2 ^ 16 - 1 adresine veya 0xffff (64 kilobayt) olan 16 bitlik yazma�lara sahiptir. Bellek b�l�tleme, mevcut t�m adres alan�n� kullanmak i�in kullan�l�r. T�m bellek, 65536 bayt (64 KB) k���k, sabit boyutlu b�l�mlere ayr�lm��t�r. 16 KB'l�k yazma�larla 64 KB'�n �st�ndeki haf�zay� ele alamayaca��m�zdan, alternatif bir y�ntem tasarlanm��t�r. Bir adres, iki b�l�mden olu�ur: bir taban adresi olan bir Segment Selector ve bu taban adresinden bir uzakl�k. Real Mode'da, bir Segment Selector'�n ili�kili taban adresi Segment Selector * 16'd�r. Dolay�s�yla, bellekte fiziksel bir adres almak i�in Segment Selector par�ay� 16 ile �arp�p ofset eklemeliyiz:

PhysicalAddress = Segment Selector * 16 + Offset

�rne�in, CS: IP 0x2000: 0x0010 ise, kar��l�k gelen fiziksel adres:

>>> hex((0x2000 << 4) + 0x0010)
'0x20010'

Ancak, en b�y�k Segment Selector'�n� ve offsetini 0xffff:0xffff olarak al�rsak, sonu�ta adres:

>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'

 Real Mode'da yaln�zca bir megabayta eri�ilebildi�inden; 0x10ffef, A20'nin devre d��� kalmas�yla 0x00ffef'e d�n��ecek.

Tamam, Real Mode ve bellek adreslemeyi biliyoruz. Reset'lemeden sonra Register de�erlerini tart��maya geri d�nelim:

CS kayd� iki b�l�mden olu�ur: G�r�n�r Segment Selector ve gizli taban adresi. Taban adresi genellikle 16 ile Segment Selector de�er �arp�larak olu�turulurken, bir donan�m s�f�rlama s�ras�nda CS kay�ttaki Segment Selector 0xf000 ile y�klenir ve taban adresi 0xffff0000 ile y�klenir; I�lemci, CS de�i�tirilinceye kadar bu �zel taban adresini kullan�r.

Ba�lang�� adresi; taban adresi, EIP kayd�ndaki de�ere eklenerek olu�turulmu�tur:

>>> 0xffff0000 + 0xfff0
'0xfffffff0'

4GB (16 bayt) olan 0xfffffff0 elde ediyoruz. Bu noktaya S�f�rlama vekt�r� denir. Bu, CPU'nun s�f�rlamadan sonra y�r�t�lecek ilk talimat� bulmas�n� bekledi�i haf�za yeridir. Genellikle BIOS giri� noktas�na i�aret eden bir atlama (jmp) y�nergesi i�erir. �rne�in, coreboot kaynak koduna bakarsak, �unu g�r�r�z:


    .section ".reset"
    .code16
.globl  reset_vector
reset_vector:
    .byte  0xe9
    .int   _start - ( . + 2 )
    ...

Burada, 0xe9 olan jmp komutunun opcode'u ve _start-(. + 2) adresindeki hedef adresi g�r�lebilir. S�f�rlama b�l�m�n�n 16 bayt oldu�unu ve 0xfffffff0'da ba�lad���n� da g�rebiliriz.

SECTIONS {
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset)
        . = 15 ;
        BYTE(0x00);
    }
}

�imdi BIOS ba�at�l�yor; BIOS'u ba�lat�p denetledikten sonra BIOS'un �ny�klenebilir bir ayg�t bulmas� gerekir. Bir �ny�kleme emri, BIOS'un hangi ayg�tlardan �ny�kleme yapmaya �al��t���n� kontrol eden BIOS yap�land�rmas�nda saklan�r. BIOS bir sabit diskten �ny�kleme yapmaya �al���rken bir �ny�kleme sekt�r� bulmaya �al���yor. MBR b�l�m d�zeniyle b�l�nm�� sabit s�r�c�ler �zerinde �ny�kleme sekt�r�, her sekt�r 512 bayt olan ilk sekt�r�n ilk 446 bayt�nda depolan�r. �lk sekt�r�n son iki bayt� 0x55 ve 0xaa'd�r ve bu, BIOS'a bu ayg�t�n �ny�klenebilir oldu�unu belirtir. �rne�in:

;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]
[ORG  0x7c00]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa

�u komutla derleyin ve �al��t�r�n: 

nasm -f bin boot.nasm && qemu-system-x86_64 boot

Bu, QEMU'ya yeni bir disk imaj� olarak olu�turdu�umuz �ny�kleme ikili dosyas�n� kullanmas�n� s�yleyecektir. Yukar�daki montaj koduyla olu�turulan ikili, �ny�kleme sekt�r�n�n gerekliliklerini yerine getirdi�inden (orijin 0x7c00 olarak ayarlan�r ve sihirli diziyle biter) ikili dosyay� bir disk imaj�n�n ana �ny�kleme kayd� (MBR) olarak de�erlendirecektir.

�unu g�receksiniz;

resim



Bu �rnekte, kodun 16 bit ger�ek modda y�r�t�lece�ini ve bellekte 0x7c00'de ba�layaca��n� g�rebilirsiniz. Ba�lad�ktan sonra; sadece "!" sembol�n� yazd�ran, 0x10 i�lemini �a��r�r; kalan 510 bayt'� s�f�rlarla doldurur ve iki sihirli bayt 0xaa ve 0x55 ile bitirir.

Objdump kullanarak bunun bir ikili d�k�m�n� g�rebilirsiniz:

nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot

Ger�ek d�nyadaki bir �ny�kleme sekt�r�, �ny�kleme i�lemini devam ettirmek i�in bir koda ve bir bit say�s� ve bir �nlem i�areti yerine bir b�l�m tablosuna sahiptir :) Bu noktadan sonra, BIOS, kontrol� �ny�kleyiciye devreder.
NOT: Yukar�da a��kland��� gibi, CPU Real Mode'dad�r; Real Mode'da, haf�zadaki fiziksel adresi hesaplama �u �ekilde yap�l�r:

PhysicalAddress = Segment Selector * 16 + Offset

T�pk� daha �nce a��kland��� gibi. Sadece 16 bit genel ama�l� kay�tlar�m�z var; 16 bitlik bir kayd�n maksimum de�eri 0xffff, yani en b�y�k de�erleri al�rsak sonu� ��yle olacakt�r:

>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'

Burada 0x10ffef 1MB + 64KB - 16b'ye e�ittir. Bunun tersine, 8086 i�lemci (Real Mode'lu ilk i�lemci), 20 bitlik bir adres sat�r�na sahiptir. 2 ^ 20 = 1048576, 1MB oldu�u i�in, mevcut kullan�labilir belle�in 1MB oldu�u anlam�na gelir. Genel Real Mode'un haf�za haritas� a�a��daki gibidir:

0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS

Bu yaz�n�n ba��nda CPU taraf�ndan y�r�t�len ilk talimat�n 0xFFFFFFF0 adresinde oldu�unu yazm��t�m, 0xFFFFFF'den (1MB) daha b�y�kt�r. CPU Real Mode'da bu adrese nas�l eri�ebilir? Cevap coreboot belgelerinde verilmi�tir:

0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space

�al��t�rman�n ba�lang�c�nda BIOS RAM'de de�il, ROM'da bulunur. 


Bootloader


GRUB 2 ve syslinux gibi Linux'u �ny�kleyebilen bir dizi �ny�kleyici var. Linux �ekirde�inin Linux deste�i uygulamak i�in bir �ny�kleyicinin gereksinimlerini belirten bir �ny�kleme protokol� vard�r. Bu �rnek, GRUB 2'yi a��klayacakt�r.

BIOS, bir �ny�kleme ayg�t� se�ti ve kontrol� �ny�kleme kesimi koduna aktard��� i�in y�r�tme boot.img'den ba�lat�l�r. Bu kod, mevcut s�n�rl� miktarda alan nedeniyle �ok basittir ve GRUB 2'nin temel g�r�nt�s�n�n konumuna atlamak i�in kullan�lan bir i�aret�i i�erir. �ekirdek imaj� diskboot.img ile ba�lar ve genellikle ilk b�l�mden hemen sonra kullan�lmayan alana ilk b�l�mden �nce kaydedilir. Yukar�daki kod, GRUB 2'nin �ekirde�ini ve dosya sistemlerini i�lemek i�in kullan�lan s�r�c�leri i�eren �ekirdek g�r�nt�s�n�n geri kalan�n� belle�e y�kler. �ekirdek imaj�n�n geri kalan�n� y�kledikten sonra, grub_main'i �al��t�r�r.
Grub_main konsolu ba�lat�r, mod�llerin temel adresini al�r, k�k ayg�t�n� ayarlar, grub yap�land�rma dosyas�n� y�kler / ayr��t�r�r, mod�lleri y�kler vb. �al��t�rma bitince, grub_main grub'� normal moda ta��r. Grub_normal_execute (grub-core / normal / main.c'den) son haz�rl�klar� tamamlar ve bir i�letim sistemi se�mek i�in bir men� g�sterir. Grub men� giri�lerinden birini se�ti�imizde grub_menu_execute_entry �al��t�r�l�r, grub'�n �ny�kleme komutunun �al��t�r�lmas� ve se�ilen i�letim sisteminin �ny�klenmesi.
�ekirdek �ny�kleme protokol�n� okuyabilece�imiz gibi, �ny�kleyici, �ekirdek kurulum kodundan 0x01f1 ofsetten ba�layan �ekirdek kurulum header'�n�n baz� alanlar�n� okumal� ve doldurmal�d�r. �ekirdek header'�  arch/86/boot/header.S,  a�a��dakilerden ba�l�yor:

 .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55

�ny�kleyicinin bunu komut sat�r�ndan al�nan ya da hesaplanan de�erlerle header'lar�n geri kalan�n� doldurmas� gerekir. (�ekirdek kurulum header'�n�n t�m alanlar�na ili�kin a��klamalar� tam olarak ele almayaca��z, bunun yerine �ekirde�in bunlar� nas�l kulland���n� tart��t���m�zda �ny�kleme protokol�ndeki t�m alanlar�n bir a��klamas�n� bulabilirsiniz.)

�ekirdek �ny�kleme protokol�nde g�rebilece�iniz gibi, �ekirdek y�klendikten sonra bellek haritas� a�a��daki gibi olacakt�r:

         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+


Yani, �ny�kleyici kontrol� �ekirde�e aktard���nda �uradan ba�lar:

X + sizeof(KernelBootSector) + 1

Burada X, �ekirdek �ny�kleme sekt�r�n�n y�klenmekte oldu�u adresidir. Benim durumumda; X,  0x10000'd�r. Bellek d�k�m�nde g�rebilece�imiz gibi:

resim

�ny�kleyici, Linux �ekirde�ini belle�e y�kledi, header alanlar�n� doldurdu ve kar��l�k gelen bellek adresine atlad�. Art�k do�rudan �ekirdek kurulum koduna ge�ebiliriz.

�ekirdek Kurulumunun Ba�lang�c�

Son olarak, �ekirdekteyiz! Teknik olarak, �ekirdek hen�z �al��m�yor; �lk olarak �ekirde�i, bellek y�neticisini, s�re� y�neticisini vb. Kurmam�z gerekir. �ekirdek kurulumunun �al��mas�, _start'da arch / x86 / boot / header.S'den ba�lar. �nceden birka� talimat oldu�undan ilk bak��ta biraz tuhaf. 

Uzun zaman �nce, Linux �ekirde�i kendi �ny�kleyicisini kullan�yordu. Ancak �imdi, �u komutu �al��t�r�rsan�z;

qemu-system-x86_64 vmlinuz-3.18-generic


�unu g�receksiniz:


resim


Asl�nda header.S, MZ'den ba�lar (yukar�daki resme bak�n), PE hata mesaj�n� yazd�ran ve a�a��daki PE header'�:

#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0


UEFI ile bir i�letim sistemini y�klemek i�in buna ihtiya� duyuyor. �u anda bunun i� i�leyi�ine bakmayaca��z ve ilerleyen b�l�mlerde de bunu ele alaca��z.

Ger�ek �ekirdek kurulum giri� noktas� ��yledir:

// header.S line 292
.globl _start
_start:


�ny�kleyici(grub2 ve di�erleri), bu noktay� (MZ'den 0x200 ofset) biliyor ve header.S'nin bir hata mesaj� yazd�ran .bstext b�l�m�nden ba�lamas�na ra�men, do�rudan ona bir s��rama yap�yor:

//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }


�ekirdek kurulum giri� noktas� ��yledir:

 .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //

Burada start_of_setup-1f noktas�na atlayan bir jmp komut opcode (0xeb) g�rebilirsiniz. Nf g�steriminde 2f, a�a��daki yerel 2: etiketini belirtir; Bizim durumumuzda, atlama sonras�nda bulunan etiket 1 ve kurulum ba�l���n�n geri kalan�n� i�eriyor. Kurulum ba�l���n�n hemen sonras�nda, .entrytext b�l�m�n� g�r�yoruz; bu b�l�m, start_of_setup etiketinden ba�l�yor.

Asl�nda �al��an ilk kod budur (elbette �nceki atlama talimatlar� hari�). �ekirdek kurulumu bootloader'dan kontrol ald�ktan sonra, ilk jmp komutu, �ekirdek ger�ek Real Mode'unun ba�lang�c�ndan itibaren 0x200 ofset'inde, yani ilk 512 bayttan sonra yer al�r. Bu, hem Linux �ekirde�i �ny�kleme protokol�n� okuyabilir hem de grub2 kaynak kodunda g�rebiliriz:

segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;

Bu, �ekirdek kurulumu ba�lad�ktan sonra segment kay�tlar�n�n a�a��daki de�erleri i�ermesi anlam�na gelir:

gs = fs = es = ds = ss = 0x1000
cs = 0x1020

Benim durumumda, �ekirdek 0x10000'a y�klendi.

Start_of_setup atlamas�n�n ard�ndan, �ekirde�in a�a��dakileri yapmas� gerekir:

- T�m segment kay�t de�erlerinin e�it oldu�undan emin ol
- Gerekirse do�ru bir y���n� ayarla.
- Bss'yi ayarla
- Main.c dosyas�ndaki C koduna atla

�imdi uygulamaya bakal�m.


Segment registers align


Her �eyden �nce, �ekirdek ds ve es segment kay�tlar�n�n ayn� adrese i�aret etmesini sa�lar. Sonra, cld y�nergesi kullanarak y�n bayra��n� temizler:

 movw    %ds, %ax
    movw    %ax, %es
    cld

Daha �nce yazm�� oldu�um gibi, grub2, �ekirde�i kurulum kodunu 0x10000 adresinde ve cs'yi 0x1020'de y�kler; ��nk� �al��t�rma dosyan�n ba��ndan ba�lamaz;

_start:
    .byte 0xeb
    .byte start_of_setup-1f

jump, which is at a 512 byte offset from 4d 5a. It also needs to align cs from 0x10200 to 0x10000, as well as all other segment registers. After that, we set up the stack:

pushw   %ds
    pushw   $6f
    lretw

Bu, ds de�erini 6 etiketinin adresiyle y���na iter ve lretw komutunu �al��t�r�r. lretw komutu �a��r�ld���nda, etiket 6'n�n adresini komut g�sterici kayd�na y�kler ve cs'yi ds de�eriyle y�kler. Daha sonra, ds ve cs ayn� de�erleri alacakt�r.

Stack Kurulumu

Kurulum kodunun neredeyse tamam� ger�ek modda C dil ortam�na haz�rlan�yor. Bir sonraki ad�m ss kay�t de�erini kontrol etmek ve e�er yanl��sa d�zeltmektir:

 movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f

Bu, 3 farkl� senaryonun ortaya ��kmas�na neden olabilir:
- Ss'in ge�erli de�eri 0x10000't�r (cs'in yan�ndaki t�m di�er b�l�m kay�tlar� gibi)
- Ss ge�ersiz ve CAN_USE_HEAP bayra�� ayarlanm�� (a�a��ya bak�n)
- Ss ge�ersiz ve CAN_USE_HEAP bayra�� ayarlanmam�� (a�a��ya bak�n)

Bu �� senaryonun hepsine s�rayla bakal�m:

- Ss'nin do�ru adrese sahip. (0x10000). Bu durumda, etiket 2'ye gidiyoruz:

2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti

Burada, dx (bootloader taraf�ndan verilen sp i�eriyor) 4 bayt'a ve s�f�r olup olmad���na ili�kin bir hizaya geldiklerini g�rebiliriz. S�f�r ise, dx'e 0xfffc (64 KB'lik maksimum segment boyutundan �nce 4 bayt hizal� adres) koyar�z. S�f�r de�ilse, �ny�kleyici (benim durumumda 0xf7f4) taraf�ndan verilen sp'yi kullanmaya devam ederiz. Bundan sonra, ax de�erini 0x10000'l�k do�ru segment adresini ss i�ine yerle�tirdik ve do�ru bir sp ayarlad�. Art�k do�ru bir y���n�m�z var:

resim

- �kinci senaryoda, (ss! = Ds). �nce, _end'in de�erini (kurulum kodunun sonundaki adres) dx'e koyar ve y���n� kullan�p kullanamayaca��m�z� test etmek i�in testb komutunu kullanarak loadflags ba�l�k alan�n� kontrol ederiz. Loadflags, a�a��daki gibi tan�mlanan bir bit maskesi header'� d�r:

#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)

Ve �ny�kleme protokol�n� okuyabildi�imiz i�in,

Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.


CAN_USE_HEAP biti ayarlanm��sa, heap_end_ptr'� dx'e (_end'i i�aret eder) yerle�tirir ve ona STACK_SIZE (minimum y���n boyutu, 512 bayt) ekleriz. Bundan sonra dx ta��nmazsa (ta��nmayacak, dx = _end + 512), etiket 2'ye atlan�r (�nceki durumda oldu�u gibi) ve do�ru bir y���n olu�ur.

resim


CAN_USE_HEAP ayarlanmad���nda _end ile _end + STACK_SIZE aras�nda en az bir y���n kullan�r�z:

resim

BSS Kurulumu


Ana C koduna atlayabilmemiz i�in ger�ekle�mesi gereken son iki ad�m BSS alan�n� kuruyor ve "sihirli" imzay� kontrol ediyor. �lk olarak imza kontrol�:

cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad

Bu, setup_sig'yi sihirli say� 0x5a5aaa55 ile kar��la�t�r�r. E�it de�illerse �l�mc�l bir hata bildirilir. Sihirli say� e�le�irse, bir dizi do�ru b�l�m kayd� ve bir y���m�z oldu�unu bilerek, yaln�zca C koduna atlamadan �nce BSS b�l�m�n� kurmam�z gerekir.

BSS b�l�m� statik olarak ayr�lm��, ba�lat�lmam�� verileri depolamak i�in kullan�l�r. Linux dikkatli bir �ekilde bu bellek alan�n� a�a��daki kod kullan�larak s�f�rlanmas�n� sa�lar:

 movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl

�lk olarak, __bss_start adresi di'ye ta��n�r. Daha sonra, _end + 3 adresi (+3 - 4 bayta hizalan�r) cx'e ta��n�r. Eax kayd� silinir (bir xor komutu kullan�larak) ve bss b�l�m boyutu (cx-di) hesaplan�r ve cx'e yerle�tirilir. Daha sonra, cx d�rde b�l�n�r (bir 'word' boyutu) ve stosl talimat� eax (s�f�r) de�erini di'nin g�sterdi�i adrese depolayarak di'yi d�rt artt�rarak tekrar cx'e kadar tekrarlar S�f�ra ula��r). Bu kodun net etkisi, s�f�rlar�n __bss_start'dan _end'e bellekteki t�m kelimeleri kullanarak yaz�ld���d�r:

resim

Main'e Atlamak 

��te hepsi bu - y���n ve BSS'ye sahibiz, bu y�zden main() C metoduna atlayabiliriz:

 calll main

Main() metodu arch/ x86/ boot/main.c dosyas�nda bulunur. Bir sonraki b�l�mde bunun ne yapt���n� okuyabilirsiniz.

Sonu�

Linux-insides hakk�ndaki ilk yaz�n�n sonuna geldik. Sorular�n�z veya �nerileriniz varsa, @0xAX twitter hesab�mdan, mail yoluyla veya GitHub'da isuue a�arak bana ula�abilirsiniz. Sonraki b�l�mde Linux �ekirde�i setup'�ndaki , memset, memcpy, initialprintk, konsol uygulamas� ve ba�lat�lmas� gibi bellek rutinlerini uygulayan ilk C kodunu ve �ok daha fazlas�n� g�rece�iz. �ngilizce ana dilim de�il ve bu durum i�in �z�r dilerim. Herhangi bir hata bulursan�z l�tfen linux-inside'a PR g�nderin.

Linkler

- Intel 80386 programmer's reference manual 1986
- Minimal Boot Loader for Intel� Architecture
- 8086
- 80386
- Reset vector
- Real mode
- Linux kernel boot protocol
- CoreBoot developer manual
- Ralf Brown's Interrupt List
- Power supply
- Power good signal

