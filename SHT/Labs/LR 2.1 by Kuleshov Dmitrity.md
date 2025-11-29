# Задание 1

```asm
; Короче сначала напишу код на C, который будет лучше помогать разобраться в коде после сна

; int a[10];
; int ecx = 0;
; while (ecx < 10) {
;     a[ecx] = 0;
;     ecx += 1;
; }
; ecx = 0;
; while (ecx < 10) {
;     eax = ecx;
;     eax *= eax;
;     a[ebx] = eax;
;     ecx += 1;
; }
; ecx = 0;
; eax = 0;
; return 0;


global _start

section .text

_start:
    sub  esp, 40      ; выделяем 40 байт для локальных переменных (массив на 10 dword по 4 байта), i будем хранить в регистрах-счетчиках

    mov ecx, 0x0  ; обнуляем счётчик
    jmp .first_while_head ; Идём в условие первого вайла


; первый цикл while (i < 10) {body1}
.first_while_head:
    cmp ecx, 0xa
    jl .first_while_body ; Если сравнение выдало less (<), то продолжаем цикл и прыгаем в тело цикла
    ; Если оказались здесь => не перепрыгнули в тело цикла => он закончился. Обнуляем счетчик для следующего цикла и переходим в его голову

    mov ecx, 0x0
    jmp .second_while_head

.first_while_body:
    mov dword [esp+ecx*4], 0x0
    add ecx, 0x1
    ; Отправляемся в проверку условия выхода - в голову цикла
    jmp .first_while_head

; второй цикл while (i < 10) {body2}
.second_while_head:
    cmp ecx, 10
    jl .second_while_body ; Если сравнение выдало less (<), то продолжаем цикл и прыгаем в тело цикла
    ; Если оказались здесь => не перепрыгнули в тело цикла => он закончился. Обнуляем регистры и выходим из программы

    mov eax, 1 ; sys_exit
    mov ecx, 0
    mov ebx, 0 ; return code = 0
    int 0x80 ; kernel call

.second_while_body:
    mov eax, ecx
    imul eax, eax

    mov dword [esp+ecx*4], eax
    add ecx, 0x1
    ; Отправляемся в проверку условия выхода - в голову цикла
    jmp .second_while_head
```

Компиляция:
```bash
nasm -g -f elf32 task_1.asm -o task_1.o
gcc -m32 -nostartfiles -nostdlib -g task_1.o -o program
gdb ./program
```

Отладка:

В самом начале первого цикла:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764337716833.jpeg)

В середине первого цикла

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764337997948.jpeg)

Видно, что первые 5 элементов (как раз пятый имеет индекс 4, это число и записано в ecx) обнулились

После первого цикла (оно же начало второго цикла)

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764338124883.jpeg)

Весь массив обнулился

Середина второго цикла 

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764338646538.jpeg)

Конец программы:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764338921891.jpeg)

Все правильно и до конца заполнилось квадратами!

# Задание 2

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

extern int calculate_ans(uint32_t *arr, uint32_t n, uint32_t m);

void print_arr(uint32_t *arr, uint32_t n, uint32_t m) {
    for (uint32_t i = 0; i < n; i++) {
        for (uint32_t j = 0; j < m; j++) {
            printf("%d\t", arr[i * m + j]);
        }
        printf("\n");
    }
}

int main() {
    uint32_t n, m;
    scanf("%d%d", &n, &m);

    uint32_t array[n][m];

    for (uint32_t i = 0; i < n; i++) {
        for (uint32_t j = 0; j < m; j++) {
            array[i][j] = rand();
        }
    }

    print_arr(&array[0][0], n, m);

//    uint32_t result = 0;
//
//    for (uint32_t i = 0; i < n; i++) {
//        for (uint32_t j = 0; j < m; j++) {
//            uint32_t modulo_value = array[i][j] % (i + 1);
//            result += modulo_value * modulo_value;
//        }
//    }

    uint32_t result = calculate_ans(&array[0][0], n, m);

    printf("Sum returned: %d\n", result);
    return 0;
}
```

```nasm
global calculate_ans

section .text

; uint32_t calculate_ans(uint32_t *arr, uint32_t n, uint32_t m)
; rdi = arr
; rsi = n
; rdx = m (далее - r9d)
calculate_ans:
    mov r8d, 0 ; r8d = result = 0
    mov r9, rdx ; потому что rdx будет использоваться в делении
    mov ecx, 0 ; ecx = i = 0

for_i:
    cmp ecx, esi ; ecx = i, esi = n
    jge end
    mov ebx, ecx ; копируем i в ebx (в делитель)
    add ebx, 1
    mov r10d, 0 ; r10d = j = 0

for_j:
    cmp r10d, r9d
    jge for_j_end

    mov r11d, ecx ; r11d = i
    imul r11d, r9d ; r11d = i * m
    add r11d, r10d ; r11d = i * m + j

    mov eax, [rdi + r11 * 4] ; eax = arr[i * n + j]
    mov edx, 0 ; для div: EDX|EAX
    div ebx ; eax = arr[i][j] / (i+1) = edx|eax / ebx, частное в eax, остаток в edx

    mov eax, edx
    imul eax, eax
    add r8d, eax

    inc r10d ; j++
    jmp for_j

for_j_end:
    inc ecx ; i++
    jmp for_i

end:
    mov eax, r8d ; eax -> return result
    ret
```

Компиляция:
```bash
nasm -g -f elf64 module.asm -o module.o
gcc -g -c main.c -o main.o
gcc -g main.o module.o -o program
gdb ./program
```

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764333409562.jpeg)

Отладка

До цикла с заполнением рандомными числами:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764399285113.jpeg)

В середине первого цикла

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764399388485.jpeg)

В конце цикла:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764399473202.jpeg)

Вывод массива и переход а asm:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764399578606.jpeg)

Перед циклом регистры вот такие:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764399939471.jpeg)

Назначения регистров в моей программе:

| rcx | i                                                                    |
| --- | -------------------------------------------------------------------- |
| rsi | n (параметр)                                                         |
| r10 | j                                                                    |
| rdx | m (сначала в качестве параметра функции), далее - остаток от деления |
| r9  | сохраненный m                                                        |
| r11 | для индексации по двумерному массиву (r11 = i\*n+m)                  |
| r8  | результат                                                            |
| rdi | адрес начала массива (параметр)                                      |

После первых двух итераций (попал в  i=1, j=2):

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764400150999.jpeg)

r8 такой маленький, потому что он не успел увеличиться, первая строка вообще не дает прибавки к ответу, вторая - может дать +1 в 50% случаев

на i=3,j=8:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764400328275.jpeg)

Как мы видим, r8 потихоньку увеличивается и будет увеличиваться дальше ещё быстрее, т.к. i станет больше и остатки тоже

В конце работы asm модуля:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764400605012.jpeg)

ответ в r8 - 149

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764400658584.jpeg)

# Дополнительное задание

## Задание 1

```asm
global _start
global buffer

section .data
buffer db 12 dup(0) ; статичный указатель на начало буфера (char[])

section .text

_start:
    push ebp
    mov  ebp, esp
    sub  esp, 40      ; выделяем 40 байт для локальных переменных (массив на 10 dword по 4 байта), i будем хранить в регистрах-счетчиках

    mov ecx, 0x0  ; обнуляем счётчик
    jmp .first_while_head ; Идём в условие первого вайла


; первый цикл while (i < 10) {body1}
.first_while_head:
    cmp ecx, 0xa
    jl .first_while_body ; Если сравнение выдало less (<), то продолжаем цикл и прыгаем в тело цикла
    ; Если оказались здесь => не перепрыгнули в тело цикла => он закончился. Обнуляем счетчик для следующего цикла и переходим в его голову

    mov ecx, 0x0
    jmp .second_while_head

.first_while_body:
    mov dword [esp+ecx*4], 0x0
    add ecx, 0x1
    ; Отправляемся в проверку условия выхода - в голову цикла
    jmp .first_while_head

; второй цикл while (i < 10) {body2}
.second_while_head:
    cmp ecx, 0xa
    jl .second_while_body ; Если сравнение выдало less (<), то продолжаем цикл и прыгаем в тело цикла
    ; Если оказались здесь => не перепрыгнули в тело цикла => он закончился. Обнуляем регистры и выходим из программы

    mov eax, 0
    mov ecx, 0
    jmp .print_loop

.second_while_body:
    mov eax, ecx
    imul eax, eax

    mov dword [esp+ecx*4], eax
    add ecx, 1
    ; Отправляемся в проверку условия выхода - в голову цикла
    jmp .second_while_head


.print_loop:
    cmp ecx, 10
    jge .end_program

    mov eax, dword [esp + ecx*4]

    ; конвертация числа в строку
    mov edi, buffer

    push ecx
    call int_to_str ; функция int_to_str берёт число из eax и переводит его в строчный вид, помещая в edi
    pop ecx

    ; вывести buffer через write(stdout=1, buffer, eax)
    push ecx
    mov ebx, 1 ; stdout
    mov ecx, buffer ; указатель на буфер
    mov edx, eax ; длина вывода
    mov eax, 4 ; syscall write
    int 0x80
    pop ecx

    inc ecx
    jmp .print_loop

.end_program:
    mov eax, 1 ; sys_exit
    mov ecx, 0
    mov ebx, 0 ; return code = 0
    int 0x80 ; kernel call
    ret


; в eax - число
; в edi - адрес буфера
int_to_str:
    ; оч важный момент, нам уже не хватает регистров,
    ; а мы на 32, да и в целом будет классно,
    ; если в каждой функции мы будем сохранять старые значения регистров,
    ; чтобы за собой возвращать на место
    push ebx
    push esi
    ; ecx будем использовать как индекс для записи цифр (в обратном порядке)
    mov ecx, 0

.convert:
    mov edx, 0
    mov ebx, 10 ; делитель
    div ebx ; частное пойдёт обратно в eax, остаток - в edx, он же dl
    add dl, '0' ; узнаем номер символа искомой цифры
    mov byte [edi + ecx], dl ; в С было бы: char *edi = buffer; buffer[ecx] = dl;
    inc ecx
    test eax, eax ; то же самое, что-> cmp eax, 0
    jnz .convert ;                     jne .convert, но быстрее

    ; reverse, т.к. у нас сча обратный порядок символов числа из-за специфики алгоритма (1337 -> '7','3','3','1')
    mov esi, 0 ; будет вторым индексом, который будет идти на встречу ebx
    mov ebx, ecx
    ; развернём строку: i = 0; j = ecx-1; while i < j swap
    dec ebx
.rev:
    cmp esi, ebx
    jge .after_rev ; если они пересеклись - то пора остановиться

    mov al, byte [edi + esi] ; в al записываем символ из переднего индекса
    mov ah, byte [edi + ebx] ; в ah записываем символ из заднего индекса (симметричного относительно середины)

    xchg al, ah ; свапаем извлечённые символы строки

    mov byte [edi + esi], al ; возвращаем символ переднего индекса на место
    mov byte [edi + ebx], ah ; возвращаем символ заднего индекса на место

    inc esi
    dec ebx
    jmp .rev
.after_rev:

    mov byte [edi + ecx], 10 ; '\n'

    ; возвращаем длину для WriteConsole: ecx + 1 (digits + newline)
    mov eax, ecx
    inc eax

    pop esi
    pop ebx

    ret
```

Как видим, вывод в консоль работает

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764400756071.jpeg)

## Задание 2

```asm
.globl _main

.section .text

_main:
    pushl %ebp
    movl %esp, %ebp
    subl $40, %esp

    movl $0, %ecx 
    jmp first_while_head

first_while_head:
    cmpl $10, %ecx
    jl first_while_body

    movl $0, %ecx
    jmp second_while_head


first_while_body:
    movl $0, (%esp,%ecx,4)
    incl %ecx
    jmp first_while_head

second_while_head:
    cmpl $10, %ecx
    jl second_while_body

    movl $0, %eax
    movl $0, %ecx
    leave
    ret

second_while_body:
    movl %ecx, %eax
    imull %eax, %eax

    movl %eax, (%esp,%ecx,4)
    incl %ecx
    jmp second_while_head
```

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764401051224.jpeg)
