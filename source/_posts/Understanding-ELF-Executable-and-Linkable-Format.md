---
title: Understanding ELF - Executable and Linkable Format
date: 2025-11-15 00:00:00
categories:
  - Programming
tags:
  - ELF
  - Linux
  - C
---

## Introduction

When you compile a C/C++ program on Linux, the resulting binary isn't just raw machine code—it's structured as an ELF (Executable and Linkable Format) file. Understanding ELF is essential for low-level programming, debugging, and binary analysis. This article explores what ELF is, its structure, and how to inspect it programmatically.

<!-- more -->

## What is ELF?

ELF (Executable and Linkable Format) is the standard binary format for executables, object files, shared libraries, and core dumps on Unix-like systems. It serves multiple purposes:

- **Compilation**: Compilers generate ELF object files (`.o`)
- **Linking**: Linkers combine object files into executables or shared libraries
- **Loading**: The OS uses ELF headers to load programs into memory
- **Dynamic Linking**: Runtime resolution of shared library dependencies

## ELF File Structure

An ELF file has four main components:

1. **ELF Header**: File metadata and offsets to other structures
2. **Program Headers**: Memory loading instructions (execution view)
3. **Section Headers**: Section layout information (linking view)
4. **Data**: The actual code, data, and metadata

### ELF Header

Located at the file's beginning, the ELF header contains essential metadata:

```c
#include <elf.h>

// For 64-bit systems
typedef struct {
    unsigned char e_ident[16];  // Magic number and other info
    uint16_t      e_type;        // Object file type (executable, shared lib, etc.)
    uint16_t      e_machine;     // Architecture
    uint32_t      e_version;     // Object file version
    uint64_t      e_entry;       // Entry point virtual address
    uint64_t      e_phoff;       // Program header table offset
    uint64_t      e_shoff;       // Section header table offset
    uint32_t      e_flags;       // Processor-specific flags
    uint16_t      e_ehsize;      // ELF header size
    uint16_t      e_phentsize;   // Program header entry size
    uint16_t      e_phnum;       // Number of program headers
    uint16_t      e_shentsize;   // Section header entry size
    uint16_t      e_shnum;       // Number of section headers
    uint16_t      e_shstrndx;    // Section header string table index
} Elf64_Ehdr;
```

### Program Headers (Segments)

Program headers tell the loader how to create the process image. Each header defines a loadable segment:

```c
typedef struct {
    uint32_t   p_type;    // Segment type (LOAD, DYNAMIC, INTERP, etc.)
    uint32_t   p_flags;   // Segment flags (read/write/execute)
    uint64_t   p_offset;  // Offset in file
    uint64_t   p_vaddr;   // Virtual address in memory
    uint64_t   p_paddr;   // Physical address (not used on most systems)
    uint64_t   p_filesz;  // Size in file
    uint64_t   p_memsz;   // Size in memory
    uint64_t   p_align;   // Alignment
} Elf64_Phdr;
```

### Section Headers

Sections organize code and data for linking. Common sections include:

- `.text`: Executable code
- `.data`: Initialized data
- `.bss`: Uninitialized data
- `.rodata`: Read-only data
- `.symtab`: Symbol table
- `.strtab`: String table
- `.dynsym`: Dynamic symbol table
- `.rel/.rela`: Relocation entries

```c
typedef struct {
    uint32_t   sh_name;      // Section name (index into string table)
    uint32_t   sh_type;      // Section type
    uint64_t   sh_flags;     // Section flags
    uint64_t   sh_addr;      // Address in memory
    uint64_t   sh_offset;    // Offset in file
    uint64_t   sh_size;      // Section size
    uint32_t   sh_link;      // Link to another section
    uint32_t   sh_info;      // Additional info
    uint64_t   sh_addralign; // Alignment
    uint64_t   sh_entsize;   // Entry size if section holds table
} Elf64_Shdr;
```

## Code Examples

### Example 1: Reading the ELF Header

`elf_header.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <elf.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

void print_elf_header(const char* filename) {
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return;
    }

    Elf64_Ehdr header;
    if (read(fd, &header, sizeof(header)) != sizeof(header)) {
        perror("read");
        close(fd);
        return;
    }

    // Check ELF magic number
    if (memcmp(header.e_ident, ELFMAG, SELFMAG) != 0) {
        printf("Not an ELF file\n");
        close(fd);
        return;
    }

    printf("ELF Header:\n");
    printf("  Magic:   ");
    for (int i = 0; i < 16; i++) {
        printf("%02x ", header.e_ident[i]);
    }
    printf("\n");

    printf("  Class:                             %s\n",
           header.e_ident[EI_CLASS] == ELFCLASS64 ? "ELF64" : "ELF32");
    printf("  Data:                              %s\n",
           header.e_ident[EI_DATA] == ELFDATA2LSB ? "Little endian" : "Big endian");
    printf("  Type:                              %d\n", header.e_type);
    printf("  Machine:                           %d\n", header.e_machine);
    printf("  Entry point address:               0x%lx\n", header.e_entry);
    printf("  Start of program headers:          %ld (bytes)\n", header.e_phoff);
    printf("  Start of section headers:          %ld (bytes)\n", header.e_shoff);
    printf("  Number of program headers:         %d\n", header.e_phnum);
    printf("  Number of section headers:         %d\n", header.e_shnum);

    close(fd);
}

int main(int argc, char* argv[]) {
    if (argc != 2) {
        printf("Usage: %s <elf_file>\n", argv[0]);
        return 1;
    }

    print_elf_header(argv[1]);
    return 0;
}
```

Execution:

```bash
./elf_header ./elf_header
```

Output:

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              Little endian
  Type:                              3
  Machine:                           62
  Entry point address:               0x1160
  Start of program headers:          64 (bytes)
  Start of section headers:          14368 (bytes)
  Number of program headers:         13
  Number of section headers:         31
```

### Example 2: Listing Sections

`list_sections.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <elf.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

void list_sections(const char* filename) {
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return;
    }

    Elf64_Ehdr header;
    if (read(fd, &header, sizeof(header)) != sizeof(header)) {
        perror("read");
        close(fd);
        return;
    }

    // Read section headers
    Elf64_Shdr* sections = malloc(header.e_shnum * sizeof(Elf64_Shdr));
    if (!sections) {
        perror("malloc");
        close(fd);
        return;
    }

    lseek(fd, header.e_shoff, SEEK_SET);
    read(fd, sections, header.e_shnum * sizeof(Elf64_Shdr));

    // Read section string table
    Elf64_Shdr shstrtab = sections[header.e_shstrndx];
    char* string_table = malloc(shstrtab.sh_size);
    if (!string_table) {
        perror("malloc");
        free(sections);
        close(fd);
        return;
    }

    lseek(fd, shstrtab.sh_offset, SEEK_SET);
    read(fd, string_table, shstrtab.sh_size);

    printf("Section Headers:\n");
    printf("  [Nr] %-20s %-15s Address          Offset   Size\n", "Name", "Type");

    for (int i = 0; i < header.e_shnum; i++) {
        char* name = string_table + sections[i].sh_name;
        printf("  [%2d] %-20s %-15d 0x%016lx %08lx %08lx\n",
               i, name, sections[i].sh_type,
               sections[i].sh_addr, sections[i].sh_offset, sections[i].sh_size);
    }

    free(string_table);
    free(sections);
    close(fd);
}int main(int argc, char* argv[]) {
    if (argc != 2) {
        printf("Usage: %s <elf_file>\n", argv[0]);
        return 1;
    }

    list_sections(argv[1]);
    return 0;
}
```

Execution:

```bash
./list_sections ./list_sections
```

Output:

```
Section Headers:
  [Nr] Name                 Type            Address          Offset   Size
  [ 0]                      0               0x0000000000000000 00000000 00000000
  [ 1] .interp              1               0x0000000000000318 00000318 0000001c
  [ 2] .note.gnu.property   7               0x0000000000000338 00000338 00000030
  [ 3] .note.gnu.build-id   7               0x0000000000000368 00000368 00000024
  [ 4] .note.ABI-tag        7               0x000000000000038c 0000038c 00000020
  [ 5] .gnu.hash            1879048182      0x00000000000003b0 000003b0 00000024
  [ 6] .dynsym              11              0x00000000000003d8 000003d8 00000180
  [ 7] .dynstr              3               0x0000000000000558 00000558 000000d8
  [ 8] .gnu.version         1879048191      0x0000000000000630 00000630 00000020
  [ 9] .gnu.version_r       1879048190      0x0000000000000650 00000650 00000040
  [10] .rela.dyn            4               0x0000000000000690 00000690 000000c0
  [11] .rela.plt            4               0x0000000000000750 00000750 000000f0
  [12] .init                1               0x0000000000001000 00001000 0000001b
  [13] .plt                 1               0x0000000000001020 00001020 000000b0
  [14] .plt.got             1               0x00000000000010d0 000010d0 00000010
  [15] .plt.sec             1               0x00000000000010e0 000010e0 000000a0
  [16] .text                1               0x0000000000001180 00001180 000004cb
  [17] .fini                1               0x000000000000164c 0000164c 0000000d
  [18] .rodata              1               0x0000000000002000 00002000 000000b0
  [19] .eh_frame_hdr        1               0x00000000000020b0 000020b0 0000003c
  [20] .eh_frame            1               0x00000000000020f0 000020f0 000000d0
  [21] .init_array          14              0x0000000000003d70 00002d70 00000008
  [22] .fini_array          15              0x0000000000003d78 00002d78 00000008
  [23] .dynamic             6               0x0000000000003d80 00002d80 000001f0
  [24] .got                 1               0x0000000000003f70 00002f70 00000090
  [25] .data                1               0x0000000000004000 00003000 00000010
  [26] .bss                 8               0x0000000000004010 00003010 00000008
  [27] .comment             1               0x0000000000000000 00003010 0000002b
  [28] .symtab              2               0x0000000000000000 00003040 00000450
  [29] .strtab              3               0x0000000000000000 00003490 0000029c
  [30] .shstrtab            3               0x0000000000000000 0000372c 0000011a
```

### Example 3: Reading the Symbol Table

`print_symbols.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <elf.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

void print_symbols(const char* filename) {
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return;
    }

    Elf64_Ehdr header;
    if (read(fd, &header, sizeof(header)) != sizeof(header)) {
        perror("read");
        close(fd);
        return;
    }

    // Read section headers
    Elf64_Shdr* sections = malloc(header.e_shnum * sizeof(Elf64_Shdr));
    if (!sections) {
        perror("malloc");
        close(fd);
        return;
    }

    lseek(fd, header.e_shoff, SEEK_SET);
    read(fd, sections, header.e_shnum * sizeof(Elf64_Shdr));

    // Find symbol table and string table
    Elf64_Shdr* symtab = NULL;
    Elf64_Shdr* strtab = NULL;

    for (int i = 0; i < header.e_shnum; i++) {
        if (sections[i].sh_type == SHT_SYMTAB) {
            symtab = &sections[i];
            strtab = &sections[sections[i].sh_link];
            break;
        }
    }

    if (!symtab) {
        printf("No symbol table found\n");
        free(sections);
        close(fd);
        return;
    }

    // Read symbol table
    int num_symbols = symtab->sh_size / sizeof(Elf64_Sym);
    Elf64_Sym* symbols = malloc(symtab->sh_size);
    if (!symbols) {
        perror("malloc");
        free(sections);
        close(fd);
        return;
    }

    lseek(fd, symtab->sh_offset, SEEK_SET);
    read(fd, symbols, symtab->sh_size);

    // Read string table
    char* str_table = malloc(strtab->sh_size);
    if (!str_table) {
        perror("malloc");
        free(symbols);
        free(sections);
        close(fd);
        return;
    }

    lseek(fd, strtab->sh_offset, SEEK_SET);
    read(fd, str_table, strtab->sh_size);

    printf("Symbol Table:\n");
    printf("  Num:    Value          Size Type    Bind   Vis      Name\n");

    for (int i = 0; i < num_symbols; i++) {
        char* name = str_table + symbols[i].st_name;
        printf("  %3d: %016lx %5ld %-7d %-6d %-8d %s\n",
               i, symbols[i].st_value, symbols[i].st_size,
               ELF64_ST_TYPE(symbols[i].st_info),
               ELF64_ST_BIND(symbols[i].st_info),
               ELF64_ST_VISIBILITY(symbols[i].st_other),
               name);
    }

    free(str_table);
    free(symbols);
    free(sections);
    close(fd);
}

int main(int argc, char* argv[]) {
    if (argc != 2) {
        printf("Usage: %s <elf_file>\n", argv[0]);
        return 1;
    }

    print_symbols(argv[1]);
    return 0;
}
```

Execution:

```bash
./print_symbols ./print_symbols
```

Output:

```
Symbol Table:
  Num:    Value          Size Type    Bind   Vis      Name
    0: 0000000000000000     0 0       0      0
    1: 0000000000000000     0 4       0      0        Scrt1.o
    2: 000000000000038c    32 1       0      0        __abi_tag
    3: 0000000000000000     0 4       0      0        crtstuff.c
    4: 00000000000011b0     0 2       0      0        deregister_tm_clones
    5: 00000000000011e0     0 2       0      0        register_tm_clones
    6: 0000000000001220     0 2       0      0        __do_global_dtors_aux
    7: 0000000000004010     1 1       0      0        completed.0
    8: 0000000000003d78     0 1       0      0        __do_global_dtors_aux_fini_array_entry
    9: 0000000000001260     0 2       0      0        frame_dummy
   10: 0000000000003d70     0 1       0      0        __frame_dummy_init_array_entry
   11: 0000000000000000     0 4       0      0        print_symbols.c
   12: 0000000000000000     0 4       0      0        crtstuff.c
   13: 00000000000021c0     0 1       0      0        __FRAME_END__
   14: 0000000000000000     0 4       0      0
   15: 0000000000003d80     0 1       0      0        _DYNAMIC
   16: 00000000000020bc     0 0       0      0        __GNU_EH_FRAME_HDR
   17: 0000000000003f70     0 1       0      0        _GLOBAL_OFFSET_TABLE_
   18: 0000000000000000     0 2       1      0        free@GLIBC_2.2.5
   19: 0000000000000000     0 2       1      0        __libc_start_main@GLIBC_2.34
   20: 0000000000000000     0 0       2      0        _ITM_deregisterTMCloneTable
   21: 0000000000004000     0 0       2      0        data_start
   22: 0000000000000000     0 2       1      0        puts@GLIBC_2.2.5
   23: 0000000000004010     0 0       1      0        _edata
   24: 0000000000001790     0 2       1      2        _fini
   25: 0000000000000000     0 2       1      0        __stack_chk_fail@GLIBC_2.4
   26: 0000000000000000     0 2       1      0        printf@GLIBC_2.2.5
   27: 0000000000001269  1231 2       1      0        print_symbols
   28: 0000000000000000     0 2       1      0        lseek@GLIBC_2.2.5
   29: 0000000000000000     0 2       1      0        close@GLIBC_2.2.5
   30: 0000000000000000     0 2       1      0        read@GLIBC_2.2.5
   31: 0000000000004000     0 0       1      0        __data_start
   32: 0000000000000000     0 0       2      0        __gmon_start__
   33: 0000000000004008     0 1       1      2        __dso_handle
   34: 0000000000002000     4 1       1      0        _IO_stdin_used
   35: 0000000000000000     0 2       1      0        malloc@GLIBC_2.2.5
   36: 0000000000004018     0 0       1      0        _end
   37: 0000000000001180    38 2       1      0        _start
   38: 0000000000004010     0 0       1      0        __bss_start
   39: 0000000000001738    88 2       1      0        main
   40: 0000000000000000     0 2       1      0        open@GLIBC_2.2.5
   41: 0000000000000000     0 2       1      0        perror@GLIBC_2.2.5
   42: 0000000000004010     0 1       1      2        __TMC_END__
   43: 0000000000000000     0 0       2      0        _ITM_registerTMCloneTable
   44: 0000000000000000     0 2       2      0        __cxa_finalize@GLIBC_2.2.5
   45: 0000000000001000     0 2       1      2        _init
```

## Standard ELF Tools

Linux provides several tools for examining ELF files:

### readelf

```bash
# Display ELF header
readelf -h program

# Display program headers
readelf -l program

# Display section headers
readelf -S program

# Display symbol table
readelf -s program

# Display all information
readelf -a program
```

### objdump

```bash
# Disassemble executable sections
objdump -d program

# Display all headers
objdump -x program

# Display section contents
objdump -s program

# Display dynamic symbol table
objdump -T program
```

### nm

```bash
# List all symbols
nm program

# List only defined symbols
nm -g program

# Display symbol sizes
nm -S program
```

### file

```bash
# Check if file is ELF and get basic info
file program
```

## How ELF Works

### Two Views of ELF

ELF provides two complementary perspectives:

- **Linking View (Sections)**: Used by the linker to combine object files. Organizes related data (code, symbols, relocations)
- **Execution View (Segments)**: Used by the loader. Groups sections by runtime properties (permissions, loading address)

### Program Loading Process

When executing an ELF binary:

1. Kernel validates the ELF header (magic number, architecture)
2. Reads program headers to determine memory layout
3. Maps segments to virtual memory at specified addresses
4. Sets memory permissions (read/write/execute) per segment
5. Loads the dynamic linker if needed (`PT_INTERP` segment)
6. Jumps to entry point (`e_entry`)

## Conclusion

ELF is a well-designed format that elegantly handles compilation, linking, and execution on Unix-like systems. Key takeaways:

- **Dual-purpose structure**: Sections for linking, segments for execution
- **Rich metadata**: Contains symbols, relocations, and debug information
- **Tool support**: `readelf`, `objdump`, and `nm` enable deep inspection
