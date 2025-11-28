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
            printf("%d\t", arr[i * n + j]);
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
    imul r11d, edx ; r11d = i * m
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

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764343341614.jpeg)

В середине первого цикла

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764343821700.jpeg)

В конце цикла:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764344004970.jpeg)

после функции с выводом:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764344906897.jpeg)

вот так переходим в asm:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764345178543.jpeg)

Перед циклом регистры вот такие:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764345516712.jpeg)

Назначения регистров в моей программе:

| ecx  | i                                                            |
| ---- | ------------------------------------------------------------ |
| esi  | n (параметр)                                                 |
| r10d | j                                                            |
| edx  | m (в качестве параметра функции), далее - остаток от деления |
| r9d  | сохраненный m                                                |
| r11d | для индексации по двумерному массиву                         |
| r8d  | результат                                                    |
| rdi  | адрес начала массива (параметр)                              |

После первых двух итераций:
![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764347388036.jpeg)

на i=3,j=1:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764348126469.jpeg)

Как мы видим, r8 потихоньку увеличивается и будет увеличиваться дальше ещё быстрее, т.к. i станет больше и остатки тоже

В конце работы asm модуля:

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764348686221.jpeg)

ответ в r8 - 91

![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764348896969.jpeg)

# Дополнительное задание

## Задание 1


![](./Materials/LR%202.1%20by%20Kuleshov%20Dmitrity-1764355941904.jpeg)