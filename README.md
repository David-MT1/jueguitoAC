# "laberinto"
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

    ; Ocultar cursor del sistema
    mov ah, 01h
    mov cx, 2607h
    int 10h

    call draw_map

game_loop:
    ; Dibujar jugador: cuadradito amarillo
    mov dl, p_x
    mov dh, p_y
    call move_cursor
    mov al, 0FEh         ; Carácter de bloque/cuadradito
    mov ah, 09h
    mov bl, 0Eh          ; Amarillo brillante
    mov bh, 0
    mov cx, 1
    int 10h

    ; Leer teclado
    mov ah, 00h
    int 16h
    mov bl, al           ; AL = Código ASCII
    mov bh, ah           ; AH = Scan Code

    mov cl, p_x
    mov ch, p_y

    ; Tecla ESC para salir
    cmp bl, 27
    je exit

    ; Controles Arriba
    cmp bl, 'w'
    je move_up
    cmp bl, 'W'
    je move_up
    cmp bh, 48h
    je move_up

    ; Controles Abajo
    cmp bl, 's'
    je move_down
    cmp bl, 'S'
    je move_down
    cmp bh, 50h
    je move_down

    ; Controles Izquierda
    cmp bl, 'a'
    je move_left
    cmp bl, 'A'
    je move_left
    cmp bh, 4Bh
    je move_left

    ; Controles Derecha
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
    ; Offset en memoria = Y * 20 + X
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

    ; Borrar posición anterior dibujando un espacio
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
    mov bl, 0Ah          ; Verde brillante
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

; --- RUTINAS DE VIDEO ---
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
    mov bl, 09h          ; Pared azul claro
    jmp print_map_char

check_exit_color:
    cmp al, 'E'
    jne print_map_char
    mov bl, 0Ah          ; Salida verde
    mov al, 0FEh         ; Cuadradito para la salida

print_map_char:
    mov ah, 09h
    mov bh, 0
    mov cx, 1
    int 10h

    inc si
    jmp draw_loop
draw_done:
    ret

; --- RUTINAS DE SONIDO ---
sound_step:
    ret

sound_wall:
    cmp sound_on, 0
    je snd_end2
    call beep
snd_end2:
    ret

sound_win:
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
    mov ax, 0E07h        ; AH=0Eh teletipo, AL=07h carácter de campana (ASCII 7)
    mov bh, 0
    int 10h
    pop bx
    pop ax
    ret
Desarrollado en Ensamblador 8086.
