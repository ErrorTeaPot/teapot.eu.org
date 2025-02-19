---
author: Gabin Le Saout
pubDatetime: 2024-10-07T15:44:00Z
title: Binary code injection
featured: true
draft: false
tags: 
  - WIP
  - school_projects
description: This article will describe the project done during ISOS at school
---
# Introduction
This project has been done during the ISOS lab lessons. The objective was to inject some code and execute it into a given ELF binary. It is not binary dependant, as long as it is an ELF one it should work. 

To help us conducting this, the steps has been divided into multiple challenges. I am following this structure for this article

## Table of contents

# Initializing things
## Arguments
First we need to create our C file and write the argument parsing part. We needed to do it using `argp()`, handling the necessary files and a `--help`.
This part has been done into a dedicated C file named `arg_parser.c`. 

There are some requirements to make it work :
- Variables to describe how we want arguments to be given
```c
static char doc[] = "ISOS project";

static char args_doc[] = "-e <elf_file> -c <code_section> -b <base_address> -m <modify-entry>";

/**
 * @brief Describes the options our program needs to understand
 *
 */
static struct argp_option options[] = {
    {"elf", 'e', "ELF_FILE", 0, "The ELF file to be analyzed", 0},
    {"code", 'c', "CODE_FILE", 0, "The binary file containing the machine code to be injected", 0},
    {"section", 's', "SECTION_NAME", 0, "The name of the newly created section", 0},
    {"base", 'b', "BASE_ADDRESS", 0, "The base address of the injected code", 0},
    {"modify-entry", 'm', 0, 0, "Modify the entry function or not", 0},
    {0}};
```
- A switch case to code what to do for each given parameter
```c
static error_t
parse_opt(int key, char *arg, struct argp_state *state)
{
    struct arguments *arguments = state->input;

    switch (key)
    {
    case 'e':
        //I use directly = since arguments are stored into the main stack frame
        arguments->elf_filename = arg;
        break;
    case 'c':
        arguments->injected_code_filename = arg;
        break;
    case 's':
        arguments->section_name = arg;
        break;
    case 'b':
        arguments->base_address = strtoul(arg, NULL, 16);
        break;
    case 'm':
        arguments->should_modify_entry_point = 1;
        break;
    default:
        return ARGP_ERR_UNKNOWN;
    }
    return EXIT_SUCCESS;
}
```
- The function that will start all this process
```c
void parse_arguments(struct arguments *arguments, int argc, char **argv)
{
    // Parse arguments
    error_t return_code = argp_parse(&argp, argc, argv, 0, 0, arguments);
    if (return_code != 0)
        errx(EXIT_FAILURE, "Bad return code for argp_parse()");
}
```
## Verifying the target
Our program is supposed to work on 64-bits executable ELF binaries. To perform this verification, we are using `libbfd`.
> BFD is a package which allows applications to use the same routines to operate on object files whatever the object file format. A new object file format can be supported simply by creating a new BFD back end and adding it to the library. 
```c
int is_exploitable(struct arguments *arguments)
{
    bfd_init();

    bfd *elf_bfd = bfd_openr(arguments->elf_filename, NULL);
    if (elf_bfd == NULL)
        errx(EXIT_FAILURE, "Failed to open elf file using bfd_open()");

    // Check if the file is an ELF binary
    int is_an_ELF_bin = bfd_check_format(elf_bfd, bfd_object);
    int is_64bits = bfd_get_arch_size(elf_bfd) == 64;
    int is_executable = bfd_get_file_flags(elf_bfd) & EXEC_P;

    bfd_close(elf_bfd);

    if (is_an_ELF_bin && is_64bits && is_executable)
        return 1;
    else
        return 0;
}
```
The 3 required checking are done into those 3 functions.

# Finding the PT_NOTE segment header

Second, we need to find if the binary have a PT_NOTE segment header that is safe to overwrite. This header is a type that is made for auxiliary information, using it to load and execute code is known as PT_NOTE attacks.

To find it, we will go through each program header, look at its type and if it corresponds to what we are looking for then we save its index. This part is done into the `int get_first_pt_note_header(struct arguments *arguments);` function, with the arguments given by the user to be able to use the binary name. 

There is the code of this function :
```c
int get_first_pt_note_header(struct arguments *arguments)
{
    [...]
    struct stat binary_info;
    if (fstat(int_fd, &binary_info) == -1)
    {
        perror("Unable to get binary stats");
        goto _cleanup_fd;
    }

    Elf64_Ehdr *mapped_elf_file = mmap(NULL, binary_info.st_size, PROT_READ, MAP_PRIVATE, int_fd, 0);
    if (mapped_elf_file == NULL)
    {
        perror("Unable to mmap the binary");
        goto _cleanup_fd;
    }

    // Get the program headers
    // We cast at first he mapped_elf_file ptr to char* so as to be able to add the offset without problems
    Elf64_Phdr *program_headers = (Elf64_Phdr *)((uintptr_t)mapped_elf_file + mapped_elf_file->e_phoff);
    for (int i = 0; i < mapped_elf_file->e_phnum; i++)
    {
        // Get the p_type field
        uint32_t p_type = program_headers->p_type;

        // Check if the p_type field is PT_NOTE
        if (p_type == PT_NOTE)
        {
            // Close the file descriptor
            if (fclose(fd) == EOF)
                perror("Failed to close the file descriptor");
            // Unmap the binary file from memory
            if (munmap(mapped_elf_file, binary_info.st_size) == -1)
                err(EXIT_FAILURE, "unmaping failed");
            return i;
        }
        // Move to the next program header
        program_headers++;
    }
    [...]
}
```
I have removed basic parts where I am opening the file and at the end where I am closing it and managing errors to make it easier to read. First, I `mmap()` the binary we are working on to work on it later. Then, I create a pointer to the beginning of the program headers by going to the first address (`mapped_elf_file`) + the offset to the program headers (`e_phoff`). Then I can go through each program header in the for loop, in which I check each program header type (`p_type`) and save its index if we are on the desired one. 

This function is then returning the index at which PT_NOTE is, or -1 if none has been found. 
