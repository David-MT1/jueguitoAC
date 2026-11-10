# Juego de Laberinto en Ensamblador (8086 Assembly)

Juego interactivo de laberinto desarrollado en lenguaje ensamblador **x8086** para entornos MS-DOS / 16-bits. El juego se ejecuta en modo texto y hace uso directo de las interrupciones del BIOS (`INT 10h` para renderizado e impresiones en pantalla y `INT 16h` para la lectura de teclado).

---

## 🎮 Características

* **Mapa Matricial 20x10:** Definido directamente en memoria con colisiones de paredes y punto de victoria.
* **Gráficos en Modo Texto BIOS:**
  * **Jugador:** Cuadradito amarillo (`0FEh` / `0Eh`).
  * **Paredes:** Azul claro (`09h`).
  * **Meta (E):** Cuadradito verde (`0Ah`).
* **Soporte de Controles Completo:** Permite movimiento con las teclas `W`, `A`, `S`, `D` (mayúsculas o minúsculas) y las **flechas direccionales**.
* **Efectos de Sonido (Beep del altavoz):**
  * 1 *beep* al colisionar contra una pared.
  * 3 *beeps* consecutivos al alcanzar la salida.

---

## ⌨️ Tabla de Controles

| Acción | Tecla ASCII | Flechas / Scan Code |
| :--- | :--- | :--- |
| **Arriba** | `W` / `w` | Flecha Arriba (`48h`) |
| **Abajo** | `S` / `s` | Flecha Abajo (`50h`) |
| **Izquierda** | `A` / `a` | Flecha Izquierda (`4Bh`) |
| **Derecha** | `D` / `d` | Flecha Derecha (`4Dh`) |
| **Salir** | `ESC` (`27`) | — |

---

## 🚀 Requisitos y Ejecución

### Opción 1: EMU8086 (Recomendado)
1. Abre el programa **EMU8086**.
2. Copia el código fuente del archivo `laberinto.asm` en el editor.
3. Presiona el botón **Compile & Emulate**.
4. En la ventana del emulador, presiona **Run**.

### Opción 2: DOSBox + TASM / MASM
1. Ensambla el código como ejecutable de tipo `.COM` (debido al encabezado `ORG 100h`).
2. Ejcuta el binario `.COM` generado dentro de DOSBox.

---

## 📜 Código Fuente (`laberinto.asm`)

```assembly
name "laberinto"
org 100h

jmp start

; --- MAPA DEL LABERINTO (20x10) ---
map db "####################"
    db "#   #               #"
    db "# # # #### ####### #"
    db "# #        #       #"
    db "# ######## # ##### #"
    db "#        # #     # #"
    db "###### ### ##### # #"
    db "#      #       #   #"
    db "# ###### #########E#"
    db "####################"

p_x db 1
p_y db 1
sound_on db 1            ; 1 = sonido en pared y al ganar, 0 = sin sonido

win_msg db "Felicidades! Escapaste del laberinto.$"

start:
    ; Configurar modo de video 80x25 texto
    mov ax, 0003h
    int 10h

    ; Ocultar cursor
    mov ah, 01h
    mov cx, 2607h
    int 10h

    call draw_map

game_loop:
    ; Dibujar jugador: cuadradito amarillo
    mov dl, p_x
    mov dh, p_y
    call move_cursor
    mov al, 0FEh         ; cuadradito
    mov ah, 09h
    mov bl, 0Eh          ; amarillo
    mov bh, 0
    mov cx, 1
    int 10h

    ; Leer teclado
    mov ah, 00h
    int 16h
    mov bl, al
    mov bh, ah

    mov cl, p_x
    mov ch, p_y

    cmp bl, 27
    je exit

    cmp bl, 'w'
    je move_up
    cmp bl, 'W'
    je move_up
    cmp bh, 48h
    je move_up

    cmp bl, 's'
    je move_down
    cmp bl, 'S'
    je move_down
    cmp bh, 50h
    je move_down

    cmp bl, 'a'
    je move_left
    cmp bl, 'A'
    je move_left
    cmp bh, 4Bh
    je move_left

    cmp bl, 'd'
    je move_right
    cmp bl, 'D'
    je move_right
    cmp bh, 4Dh
    je move_right

    jmp game_loop

move_up:
    dec ch
    jmp check_col
move_down:
    inc ch
    jmp check_col
move_left:
    dec cl
    jmp check_col
move_right:
    inc cl

check_col:
    ; Offset = Y*20 + X
    mov al, ch
    mov dl, 20
    mul dl
    mov bx, ax
    mov dl, cl
    mov dh, 0
    add bx, dx

    mov al, map[bx]

    cmp al, '#'
    je wall_hit
    cmp al, 'E'
    je win

    ; Borrar posición anterior
    mov dl, p_x
    mov dh, p_y
    call move_cursor
    push cx
    mov al, ' '
    mov ah, 09h
    mov bl, 07h
    mov bh, 0
    mov cx, 1
    int 10h
    pop cx

    ; Guardar nueva posición
    mov p_x, cl
    mov p_y, ch

    call sound_step
    jmp game_loop

wall_hit:
    call sound_wall
    jmp game_loop

win:
    call sound_win

    mov dl, 0
    mov dh, 11
    call move_cursor

    mov si, offset win_msg
print_win_loop:
    mov al, [si]
    cmp al, '$'
    je exit
    mov ah, 09h
    mov bl, 0Ah          ; verde
    mov bh, 0
    mov cx, 1
    int 10h
    inc si
    inc dl
    call move_cursor
    jmp print_win_loop

exit:
    mov ax, 4C00h
    int 21h

; --- VIDEO ---
move_cursor:
    push ax
    push bx
    mov ah, 02h
    mov bh, 00h
    int 10h
    pop bx
    pop ax
    ret

draw_map:
    mov si, 0
draw_loop:
    cmp si, 200
    je draw_done

    mov ax, si
    mov cl, 20
    div cl
    mov dh, al
    mov dl, ah
    call move_cursor

    mov al, map[si]
    mov bl, 07h
    cmp al, '#'
    jne check_exit_color
    mov bl, 09h          ; pared azul claro
    jmp print_map_char
check_exit_color:
    cmp al, 'E'
    jne print_map_char
    mov bl, 0Ah          ; salida verde
    mov al, 0FEh         ; cuadradito en vez de 'E'

print_map_char:
    mov ah, 09h
    mov bh, 0
    mov cx, 1
    int 10h

    inc si
    jmp draw_loop
draw_done:
    ret

; --- SONIDO ---
sound_step:
    ret

sound_wall:              ; 1 beep al chocar
    cmp sound_on, 0
    je snd_end2
    call beep
snd_end2:
    ret

sound_win:               ; 3 beeps al ganar
    cmp sound_on, 0
    je snd_end3
    call beep
    call beep
    call beep
snd_end3:
    ret

beep:
    push ax
    push bx
    mov ax, 0E07h        ; AH=0Eh teletipo, AL=07h campana
    mov bh, 0
    int 10h
    pop bx
    pop ax
    ret
```

---

## 🛠️ Explicación de la Lógica

1. **Cálculo de Colisiones (`check_col`):** La posición actual del jugador $(X, Y)$ se traduce a un offset lineal dentro del arreglo del mapa mediante la fórmula $\text{Offset} = (Y \times 20) + X$.
2. **Impresión de Pantalla:** Se utiliza la interrupción `INT 10h, AH=09h` para escribir caracteres junto a sus atributos de color específicos (atributos de texto de 4 bits para color de fuente y fondo).
3. **Manejo de Teclas:** `INT 16h, AH=00h` devuelve el código ASCII en `AL` y el Scan Code del teclado en `AH`, permitiendo soportar tanto letras como las flechas direccionales.
