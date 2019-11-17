## Фрагментация Heap'а: Зёбра
Если в процессе существует как минимум два heap'а, оба они динамически расширяемые и при этом их рост происходит параллельно, то в виртуальной памяти процесса может нарисоваться зебра:
![зебра](data/heap-zebra+ny_pogodi.png)

Для решения этой проблемы нужно каким-то образом заставить heap расти большими блоками, чтобы не допустить конкурентного перекрытия памяти heap'ов.
При использовании базового API нам доступны следующие механизмы:
* Можно [зарезервировать размер под Default Process Heap](https://docs.microsoft.com/ru-ru/cpp/build/reference/heap-set-heap-size?view=vs-2019), который по совместительству является CrtHeap'ом, начиная с MSVC2012.
* Создавать дополнительные heap'ы с указанием максимального размера. Это приведет к резервированию памяти в размере указанного размера с выравниванием до размера страницы в большую сторону. Но есть и негативные стороны данного способа:
    * Heap не сможет иметь прослойку LFH.
    * В таком heap'е нельзя будет выделить память более ~512Кб для 32-разрядного процесса и более ~1Мб для 64-разрядного процесса.
* Создать heap с `initialSize`. Однако, это не лучший вариант, т.к. такая память будет `commited`, т.е. будет связана с физической памятью.

Альтернативный механизм - это использовать для создания heap'а Rtl-функцию - [`RtlCreateHeap`](https://docs.microsoft.com/ru-ru/windows-hardware/drivers/ddi/ntifs/nf-ntifs-rtlcreateheap).
Её сигнатура выглядит следующим образом:
```
NTSYSAPI PVOID RtlCreateHeap(
  ULONG                Flags,
  PVOID                HeapBase,
  SIZE_T               ReserveSize,
  SIZE_T               CommitSize,
  PVOID                Lock,
  PRTL_HEAP_PARAMETERS Parameters
);
```

Т.е. позволяет гибко управлять параметрами, создаваемого heap'а.
На самом деле функция `HeapCreate` является обёрткой для `RtlCreateHeap`. Вот как выглядит дизассемблер `HeapCreate`:
```
KERNELBASE!HeapCreate:
7678d900 8bff            mov     edi,edi
7678d902 55              push    ebp
7678d903 8bec            mov     ebp,esp
7678d905 8b4d08          mov     ecx,dword ptr [ebp+8]   // HeapCreate::flOptions
7678d908 33d2            xor     edx,edx
7678d90a 8b4510          mov     eax,dword ptr [ebp+10h] // HeapCreate::dwMaximumSize
7678d90d 81e105000400    and     ecx,40005h
7678d913 56              push    esi
7678d914 8b35a8478376    mov     esi,dword ptr [KERNELBASE!SysInfo+0x8 (768347a8)]
7678d91a 81c900100000    or      ecx,1000h              // Flags |= HEAP_CLASS_1 // private heap
7678d920 3bc6            cmp     eax,esi
7678d922 7336            jae     KERNELBASE!HeapCreate+0x5a (7678d95a)  Branch

KERNELBASE!HeapCreate+0x24:
7678d924 85c0            test    eax,eax
7678d926 752e            jne     KERNELBASE!HeapCreate+0x56 (7678d956)  Branch

KERNELBASE!HeapCreate+0x28:
7678d928 8bd6            mov     edx,esi
7678d92a c1e204          shl     edx,4
7678d92d 83c902          or      ecx,2

KERNELBASE!HeapCreate+0x30:
7678d930 85d2            test    edx,edx
7678d932 7426            je      KERNELBASE!HeapCreate+0x5a (7678d95a)  Branch

KERNELBASE!HeapCreate+0x34:
7678d934 6a00            push    0                      // Parameters  = nullptr
7678d936 6a00            push    0                      // Lock        = nullptr
7678d938 ff750c          push    dword ptr [ebp+0Ch]    // CommitSize  = HeapCreate::dwInitialSize
7678d93b 50              push    eax                    // ReserveSize
7678d93c 6a00            push    0                      // HeapBase    = nullptr
7678d93e 51              push    ecx                    // Flags
7678d93f ff151c7a8376    call    dword ptr [KERNELBASE!_imp__RtlCreateHeap (76837a1c)]
                                // NTSYSAPI PVOID RtlCreateHeap(
                                //   ULONG                Flags,
                                //   PVOID                HeapBase,
                                //   SIZE_T               ReserveSize,
                                //   SIZE_T               CommitSize,
                                //   PVOID                Lock,
                                //   PRTL_HEAP_PARAMETERS Parameters
                                // );
7678d945 8bf0            mov     esi,eax
7678d947 85f6            test    esi,esi
7678d949 0f84513b0200    je      KERNELBASE!HeapCreate+0x23ba0 (767b14a0)  Branch

KERNELBASE!HeapCreate+0x4f:
7678d94f 8bc6            mov     eax,esi
7678d951 5e              pop     esi
7678d952 5d              pop     ebp
7678d953 c20c00          ret     0Ch

KERNELBASE!HeapCreate+0x56:
7678d956 8bc6            mov     eax,esi
7678d958 ebd6            jmp     KERNELBASE!HeapCreate+0x30 (7678d930)  Branch

KERNELBASE!HeapCreate+0x5a:
7678d95a 39450c          cmp     dword ptr [ebp+0Ch],eax
7678d95d 76d5            jbe     KERNELBASE!HeapCreate+0x34 (7678d934)  Branch

KERNELBASE!HeapCreate+0x5f:
7678d95f 8b450c          mov     eax,dword ptr [ebp+0Ch]
7678d962 ebd0            jmp     KERNELBASE!HeapCreate+0x34 (7678d934)  Branch

KERNELBASE!HeapCreate+0x23ba0:
767b14a0 6a08            push    8
767b14a2 ff15c8708376    call    dword ptr [KERNELBASE!_imp__RtlSetLastWin32Error (768370c8)]
767b14a8 e9a2c4fdff      jmp     KERNELBASE!HeapCreate+0x4f (7678d94f)  Branch
```

Использование функции `RtlCreateHeap` позволит создавать heap с `reserved size` достаточно большим куском, что в свою очередь позволит избавиться от "зебристости" выделений, в том числе и при последующем удалении такого heap'а у нас освободится память, ранее зарезервированного размера одним куском!
Примерно так:
![](data/heap+reserved-zebra.png)

Как видно из картинки, аллокации свыше запрошенного резерва осуществляются по прежней схеме.

**PoC** - https://github.com/pkjq/PoC.HeapReserve