#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

#define ADD 0
#define SUB 1
#define MUL 2
#define LDI 3
#define BEQZ 4
#define AND 5
#define OR 6
#define JR 7
#define SLC 8
#define SRC 9
#define LB 10
#define SB 11
#define END 15

struct register_file
{
    char reg[64]; // 8 bits per register (0-63)
    char serg;    // 000CVNSZ
    short pc;     // 16 bits -> 65536 addresses
};

struct decoded_instruction
{
    short instruction; // 16 bits
    char opcode;       // 4 bits
    char r1_address;   // 6 bits
    char r2_address;   // 6 bits
    char imm;          // 6 bits
    char r1_value;     // 8 bits
    char r2_value;     // 8 bits
    short pc_val_for_branch; // 16 bits
};

struct previous_clock_cycle
{
    short instruction_fetched;
    struct decoded_instruction instruction_decoded;
};

short instruction_memory[1024]; // 16 bits per word
char data_memory[2048];         // 8 bits per word
struct register_file register_file;
int clock_cycles = 1;
struct previous_clock_cycle previous_clock_cycle;
bool end = false;

void print_flags(char serg)
{
    bool C, V, N, S, Z;
    C = serg & 0b00010000;
    V = serg & 0b00001000;
    N = serg & 0b00000100;
    S = serg & 0b00000010;
    Z = serg & 0b00000001;
    printf("C=%d, V=%d, N=%d, S=%d, Z=%d\n", C, V, N, S, Z);
}

void print_instruction(short instruction)
{
    printf("opcode=%d, ", (instruction & 0xF000) >> 12);
    printf("r1=%d, ", (instruction & 0x0FC0) >> 6);
    printf("r2/imm=%d\n", instruction & 0x003F);
}

char str_opcode_to_char(char *op)
{
    if (strcmp(op, "ADD") == 0)
    {
        return ADD;
    }
    else if (strcmp(op, "SUB") == 0)
    {
        return SUB;
    }
    else if (strcmp(op, "MUL") == 0)
    {
        return MUL;
    }
    else if (strcmp(op, "LDI") == 0)
    {
        return LDI;
    }
    else if (strcmp(op, "BEQZ") == 0)
    {
        return BEQZ;
    }
    else if (strcmp(op, "AND") == 0)
    {
        return AND;
    }
    else if (strcmp(op, "OR") == 0)
    {
        return OR;
    }
    else if (strcmp(op, "JR") == 0)
    {
        return JR;
    }
    else if (strcmp(op, "SLC") == 0)
    {
        return SLC;
    }
    else if (strcmp(op, "SRC") == 0)
    {
        return SRC;
    }
    else if (strcmp(op, "LB") == 0)
    {
        return LB;
    }
    else if (strcmp(op, "SB") == 0)
    {
        return SB;
    }
    else if (strcmp(op, "END") == 0)
    {
        return END;
    }
    else
    {
        printf("str_opcode_to_char: Invalid opcode\n");
        return -1;
    }
}

int parse_program_file_to_inst_mem()
{
    char *filename = "program.txt";
    FILE *fp = fopen(filename, "r");
    short ctr = 0;
    if (fp == NULL)
    {
        printf("Error: could not open file %s", filename);
        return 1;
    }
    // reading line by line, max 256 bytes
    int MAX_LENGTH = 256;
    char buffer[MAX_LENGTH];

    while (fgets(buffer, MAX_LENGTH, fp))
    {
        short instruction;
        if (strcmp(buffer, "END") == 0)
        {
            instruction = 0xF000;
            instruction_memory[ctr] = instruction;
            return 0;
        }
        char *splitString = strtok(buffer, " ");
        char opcode = str_opcode_to_char(splitString);
        instruction = opcode << 12;
        int i = 0;
        while (splitString != NULL)
        {
            splitString = strtok(NULL, " ");
            if (i == 0)
            {
                instruction = instruction | (short)atoi(splitString) << 6;
            }
            else if (i == 1)
            {
                instruction = instruction | (short)atoi(splitString);
            }
            i++;
        }

        // print_instruction(instruction);
        instruction_memory[ctr] = instruction;
        ctr++;
    }

    fclose(fp);
    return 0;
}

short fetch()
{
    short instruction = instruction_memory[register_file.pc++];
    printf("Instruction fetched:\n");
    print_instruction(instruction);
    return instruction;
}

struct decoded_instruction decode(short instruction)
{
    struct decoded_instruction current_instruction;
    printf("Instruction decoded:\n");
    print_instruction(instruction);
    current_instruction.instruction = instruction;
    current_instruction.opcode = (instruction >> 12) & 0xF;
    if (current_instruction.opcode == 15)
        end = true;
    current_instruction.r1_address = (instruction >> 6) & 0x3F;
    current_instruction.r1_value = register_file.reg[current_instruction.r1_address];
    current_instruction.r2_address = (instruction) & 0x3F;
    current_instruction.r2_value = register_file.reg[current_instruction.r2_address];
    current_instruction.imm = (instruction) & 0x3F;
    current_instruction.pc_val_for_branch = register_file.pc;
    return current_instruction;
}

void add(char r1, char r2, char rd)
{
    short tmp = r1 + r2;
    // update carry flag 000C0000
    if ((tmp & 0b100000000) != 0)
        register_file.serg = register_file.serg | 0b00010000;
    else
        register_file.serg = register_file.serg & 0b11101111;

    // update overflow flag 0000V000
    bool sign1 = r1 & 0b10000000;
    bool sign2 = r2 & 0b10000000;
    bool sign_result = tmp & 0b10000000;
    if (sign1 == sign2 && sign_result != sign1)
        register_file.serg = register_file.serg | 0b00001000;
    else
        register_file.serg = register_file.serg & 0b11110111;

    // update negative flag 00000N00
    if ((tmp & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update sign flag S=V XOR N 000000S0
    bool sign_bit = ((register_file.serg & 0b00001000) >> 3) ^ ((register_file.serg & 0b00000100) >> 2);
    if (sign_bit == 1)
        register_file.serg = register_file.serg | 0b00000010;
    else
        register_file.serg = register_file.serg & 0b11111101;

    // update zero flag 0000000Z
    if (tmp == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

void sub(char r1, char r2, char rd)
{
    add(r1, -r2, rd);
}

void mul(char r1, char r2, char rd)
{
    char tmp = r1 * r2;

    register_file.reg[rd] = tmp;

    // update negative flag 00000N00
    if ((tmp & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update zero flag 0000000Z
    if (tmp == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

void and (char r1, char r2, char rd)
{
    char tmp = r1 & r2;

    // update negative flag 00000N00
    if ((tmp & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update zero flag 0000000Z
    if (tmp == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

void or (char r1, char r2, char rd)
{
    char tmp = r1 | r2;

    // update negative flag 00000N00
    if ((tmp & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update zero flag 0000000Z
    if (tmp == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

void slc(char r1, char imm, char rd)
{

    char tmp = r1 << imm;

    // update negative flag 00000N00
    if ((register_file.reg[r1] & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update zero flag 0000000Z
    if (register_file.reg[r1] == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

void src(char r1, char imm, char rd)
{
    char tmp = r1 >> imm;

    // update negative flag 00000N00
    if ((register_file.reg[r1] & 0b10000000) != 0)
        register_file.serg = register_file.serg | 0b00000100;
    else
        register_file.serg = register_file.serg & 0b11111011;

    // update zero flag 0000000Z
    if (register_file.reg[r1] == 0)
        register_file.serg = register_file.serg | 0b00000001;
    else
        register_file.serg = register_file.serg & 0b11111110;

    // write result to register
    register_file.reg[rd] = tmp;
}

short concatenate(char r1, char r2)
{
    return (r1 << 8) | r2;
}

int execute(struct decoded_instruction current_instruction)
{
    char opcode = current_instruction.opcode;
    char r1_val = current_instruction.r1_value;
    char r2_val = current_instruction.r2_value;
    char rd = current_instruction.r1_address;
    char imm = current_instruction.imm;
    printf("Instruction executed: \n");
    print_instruction(current_instruction.instruction);
    // ALU operations
    switch (opcode)
    {
    case 0:
        // R-TYPE - ADD
        add(r1_val, r2_val, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 1:
        // R-TYPE - SUB
        sub(r1_val, r2_val, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 2:
        // R-TYPE - MUL
        mul(r1_val, r2_val, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 3:
        // I-TYPE - LDI
        register_file.reg[rd] = imm;
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 4:
        // I-TYPE - BEQZ
        if (r1_val == 0)
            register_file.pc = current_instruction.pc_val_for_branch + imm;
        break;
    case 5:
        // R-TYPE - AND
        and(r1_val, r2_val, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 6:
        // R-TYPE - OR
        or (r1_val, r2_val, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 7:
        // R-TYPE - JR
        register_file.pc = concatenate(r1_val, r2_val);
        break;
    case 8:
        // I-TYPE - SLC
        slc(r1_val, imm, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 9:
        // I-TYPE - SRC
        src(r1_val, imm, rd);
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 10:
        // I-TYPE - LB
        register_file.reg[rd] = data_memory[imm];
        printf("R%d value changed to %d.\n", rd, register_file.reg[rd]);
        break;
    case 11:
        // I-TYPE - SB
        data_memory[imm] = r1_val;
        printf("mem[%d] value changed to %d.\n", imm, r1_val);
        break;
    case 15:
        // END
        break;
    default:
        printf("INVALID OPCODE\n");
        break;
    }

    return 0;
}

void next_cycle()
{
    clock_cycles++;
    printf("\n----------clock_cycle: [%d]----------\n", clock_cycles);
    if (clock_cycles > 2)
    {
        execute(previous_clock_cycle.instruction_decoded);
    }
    previous_clock_cycle.instruction_decoded = decode(previous_clock_cycle.instruction_fetched);
    if (!end)
    {
        previous_clock_cycle.instruction_fetched = fetch();
    }
}

int main()
{
    parse_program_file_to_inst_mem();
    register_file.pc = 0;
    printf("\n----------clock_cycle: [%d]----------\n", clock_cycles);
    previous_clock_cycle.instruction_fetched = fetch();
    while (!end)
    {
        next_cycle();
    }
    printf("\n>>>> TOTAL CLOCK CYCLES TO FINISH EXECUTION: %d\n", clock_cycles);
    printf(">>>> Registers:\n");
    for (int i = 0; i < 64; i++)
    {
        printf("R%d=%d ", i, register_file.reg[i]);
        if((i>0 && i%8==0) || i==63)
            printf("\n");
    }
    return 0;
}
