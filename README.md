# Trabajo Práctico: Testeo de Instrucciones RTM32

En este documento se detallan casos de prueba diseñados para testear el correcto funcionamiento de las instrucciones de la arquitectura de la CPU STX4. Se agruparon instrucciones complementarias en cada caso para verificar no solo el funcionamiento aislado, sino también la interacción entre la ALU, los registros y la memoria.

Para cada caso se provee la lógica en ensamblador, la traducción exacta a código máquina hexadecimal (respetando la codificación de formatos R, I, L y J de STX4) y los comandos de Telnet para inyectar en el debugger virtual de la `rtm32`.

---

# Caso 1: Carga de Constantes y Enmascaramiento Lógico
## Descripción: Qué estoy testeando
Se testea el uso de las instrucciones lógicas y de formato L para cargar un número de 32 bits en dos pasos, aplicarle una máscara usando un OR exclusivo (`XOR`), y luego negar todos sus bits utilizando la operación `NOR` con el registro `$zero`.
## Instrucciones: instrucciones que usé durante el test
`LUI`, `ORI` (Or Immediate), `XOR` (Exclusive Or), `NOR` (Not Or)

## Precondiciones:
- Reiniciar el estado de la CPU desde el debugger (`reset`).
- Fijar el Program Counter en 0 (`set pc 0x00000000`).
- Setear el registro `$12` ($t4) con una máscara completa de 1s para el XOR: `set r12 0xFFFFFFFF`.
- Inyectar el código máquina hexadecimal en la memoria (iniciando en la dirección `0x00000000`).

**Code:**
```bash
reset
set pc 0x00000000
set r12 0xFFFFFFFF
set [0x00000000] 0x381400FF
set [0x00000004] 0x2A94AA55
set [0x00000008] 0x0298B00A
set [0x0000000C] 0x02C0D00B
```

## Code
**Ensamblador MIPS/STX4:**
```mips
LUI  $10, 0x00FF        # Carga 0x00FF en la parte alta de $10. $10 = 0x00FF0000
ORI $10, $10, 0xAA55    # Hace OR con la parte baja (h=0). $10 = 0x00FFAA55
XOR $11, $10, $12       # $11 = 0x00FFAA55 XOR 0xFFFFFFFF = 0xFF0055AA
NOR $13, $11, $0        # Niega $11 haciendo NOR con $zero. $13 = 0x00FFAA55
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x381400FF  # LUI $10, 0x00FF       (Opcode 7, rt=10, imm=0xFF)
0x00000004 : 0x2A94AA55  # ORI $10, $10, 0xAA55  (Opcode 5, rs=10, rt=10, h=0, imm=0xAA55)
0x00000008 : 0x0298B00A  # XOR $11, $10, $12     (Opcode 0, rs=10, rt=12, rd=11, func=10)
0x0000000C : 0x02C0D00B  # NOR $13, $11, $0      (Opcode 0, rs=11, rt=0, rd=13, func=11)
```

## Postcondiciones:
- Avanzar 4 ciclos en la CPU usando `step 4`.
- Ejecutar el comando `registers` para volcar el estado del procesador.
- Leer `$10`: debe contener `0x00FFAA55`.
- Leer `$11`: debe contener `0xFF0055AA`.
- Leer `$13`: debe contener `0x00FFAA55`.

## Conclusiones: 
Anduvo correctamente. Originalmente encontramos un bug en el simulador que generaba una Instrucción Ilegal (CAUSE 3) con `LUI` (Opcode 7), pero la máquina fue actualizada y el bug fue solucionado. Pudimos comprobar que ahora se puede cargar exitosamente una constante en la parte alta y que la instrucción `NOR` funciona perfectamente como un operador NOT lógico.

---

# Caso 2: Aritmética Negativa y Desplazamientos
## Descripción: Qué estoy testeando
Se testea la correcta generación y manipulación de números negativos en complemento a dos, sumas aritméticas y desplazamientos aritméticos a la derecha para corroborar que el bit de signo se propaga adecuadamente.
## Instrucciones: instrucciones que usé durante el test
`SUB` (Subtract), `ADDI` (Add Immediate), `SRA` (Shift Right Arithmetic)

## Precondiciones:
- Resetear el simulador y acomodar el PC (`reset` -> `set pc 0`).
- Setear en el debugger el registro `$10` ($t2) con el valor `4`: `set r10 0x00000004`.
- Setear en el debugger el registro `$11` ($t3) con el valor `0` (o usar `$0`).

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000004
set [0x00000000] 0x0014C01D
set [0x00000004] 0x0B1A000A
set [0x00000008] 0x0018E082
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SUB  $12, $0, $10       # $12 = $0 - $10 (0 - 4 = -4 o 0xFFFFFFFC)
ADDI $13, $12, 10       # $13 = -4 + 10 = 6
SRA  $14, $12, 1        # Shift Right Arithmetic a -4. Debería dar -2 (0xFFFFFFFE)
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0014C01D  # SUB  $12, $0, $10 (Opcode 0, rs=0, rt=10, rd=12, func=29)
0x00000004 : 0x0B1A000A  # ADDI $13, $12, 10 (Opcode 1, rs=12, rt=13, imm=10)
0x00000008 : 0x0018E082  # SRA  $14, $12, 1  (Opcode 0, rs=0, rt=12, rd=14, aux=1, func=2)
```

## Postcondiciones:
- Avanzar la CPU con `step 3`.
- Chequear con el debugger los registros de destino con el comando `registers`:
- Registro `$12`: debe contener `0xFFFFFFFC` (-4).
- Registro `$13`: debe contener `0x00000006`.
- Registro `$14`: debe contener `0xFFFFFFFE` (-2).

## Conclusiones: 
Anduvo correctamente. Inicialmente encontramos que `ADDI` arrojaba una excepción, pero con la nueva actualización del simulador se confirmó su correcto funcionamiento y la suma inmediata se computó a la perfección. Además, comprobamos que `SRA` utiliza el campo `rt` como fuente de datos (al igual que SLL y SRL, demostrando que el manual posee un error tipográfico en la Tabla A.2 donde declara que utiliza `rs`) y que la propagación del bit de signo se mantiene tal como se espera en un desplazamiento aritmético para números negativos.

---

# Caso 3: Acceso asimétrico a Memoria y Endianness
## Descripción: Qué estoy testeando
Se testea la escritura de datos de 32 bits a memoria (Store Word), sobreescritura de un solo byte en la misma base (Store Byte) y posterior lectura asimétrica de media palabra sin extensión de signo (Load Half Unsigned) para corroborar la alineación y el esquema de almacenamiento de bytes de la CPU.
## Instrucciones: instrucciones que usé durante el test
`SW` (Store Word), `SB` (Store Byte), `LHU` (Load Half Unsigned)

## Precondiciones:
- Setear un puntero base en `$15` apuntando a una dirección válida de RAM (ej. 0x0800): `set r15 0x00000800`.
- Setear el dato original de 32 bits en `$16`: `set r16 0x11223344`.
- Setear el byte a sobreescribir en `$17`: `set r17 0x000000FF`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r15 0x00000800
set r16 0x11223344
set r17 0x000000FF
set [0x00000000] 0x4BE00000
set [0x00000004] 0x5BE20000
set [0x00000008] 0x6BE40000
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SW $16, 0($15)         # M[$15] = 0x11223344. Ocupa direcciones 0x0800 a 0x0803.
SB $17, 0($15)         # M[$15][7:0] = 0xFF. Modifica la parte menos significativa.
LHU $18, 0($15)        # Carga los 2 bytes inferiores de 0x0800 en $18 sin signo (esperado 0x000033FF).
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x4BE00000  # SW $16, 0($15) (Opcode 9, rs=15, rt=16, imm=0)
0x00000004 : 0x5BE20000  # SB $17, 0($15) (Opcode 11, rs=15, rt=17, imm=0)
0x00000008 : 0x6BE40000  # LHU $18, 0($15) (Opcode 13, rs=15, rt=18, imm=0)
```

## Postcondiciones:
- Avanzar la CPU con `step 3`.
- En el registro `$18` en el debugger se verifica cómo la CPU ensambla la media palabra, resultado esperado es `0x000033FF`.

## Conclusiones: 
Funciona. Los accesos con SW respetaron la alineación de palabra en una dirección múltiplo de 4. El resultado de la combinación de instrucciones SW -> SB -> LHU permitió validar el esquema de almacenamiento de bytes del simulador: al aplicar SB en la dirección base 0x0800, se modificó el byte menos significativo del dato original (0x44 pasó a ser 0xFF), confirmando que los bytes de menor peso se alojan en las direcciones de memoria más bajas.

Asimismo, se verificó que la instrucción LHU (Opcode 13) se ejecutó correctamente tras la última actualización; la CPU recuperó con éxito la media palabra de 16 bits combinando los contenidos de las direcciones 0x0801 y 0x0800 (0x33FF) y completó los 16 bits restantes del registro $18 con ceros, omitiendo la extensión de signo y operando sin cuelgues ni lecturas truncadas.

---

# Caso 4: Bifurcaciones y Condiciones
## Descripción: Qué estoy testeando
Se testea el mecanismo de control de flujo usando saltos condicionales respecto al PC, el registro de comparación con inmediatos, un salto incondicional largo (`J`) y un salto indirecto mediante registro (`JR`).
## Instrucciones: instrucciones que usé durante el test
`SLTI` (Set Less Than Immediate), `BEQ` (Branch Equal), `BGT` (Branch Greater Than), `J` (Jump), `JR` (Jump Register)

## Precondiciones:
- Setear `$10` con el valor `50`: `set r10 0x00000032`.
- Setear `$11` con el valor `0`: `set r11 0x00000000`.
- Setear `$20` con una dirección de memoria lejana (ej 0x0400): `set r20 0x00000400`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000032
set r11 0x00000000
set r20 0x00000400
set [0x00000000] 0xB2980064
set [0x00000004] 0x83000002
set [0x00000008] 0x9A960001
set [0x0000000C] 0x0AD60064
set [0x00000010] 0x10000100
set [0x00000400] 0x0500000E
```

## Code
**Ensamblador MIPS/STX4:**
```mips
# Iniciamos en 0x00000000
SLTI $12, $10, 100      # $10 (50) < 100 -> Verdadero. $12 se setea en 1.
BEQ  $12, $0, 2         # Si $12 == 0 saltaría 2 ints. Como es 1, no salta.
BGT  $10, $11, 1        # $10 (50) > $11 (0) -> Verdadero. Salta 1 int, esquivando el ADDI.
ADDI $11, $11, 100      # [Instrucción salteada] Sumaría 100 a $11 si no saltara.
J    0x00000400         # Salto incondicional a la dirección 0x0400 (0x100 en palabras).
# ... en la direccion 0x00000400 ...
JR   $20                # Salta directamente a la dirección guardada en $20 (0x0400). Loop infinito.
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0xB2980064  # SLTI $12, $10, 100 (Opcode 22, rs=10, rt=12, imm=100)
0x00000004 : 0x83000002  # BEQ  $12, $0, 2    (Opcode 16, rs=12, rt=0, imm=2)
0x00000008 : 0x9A960001  # BGT  $10, $11, 1   (Opcode 19, rs=10, rt=11, imm=1)
0x0000000C : 0x0AD60064  # ADDI $11, $11, 100 (Opcode 1, rs=11, rt=11, imm=100)
0x00000010 : 0x10000100  # J    0x00000400    (Opcode 2, addr=0x100 en palabras)
0x00000400 : 0x0500000E  # JR   $20           (Opcode 0, rs=20, rt=0, rd=0, func=14)
```

## Postcondiciones:
- Avanzar la CPU usando `step 5` para recorrer toda la secuencia lógica.
- Leer `$11`: debe mantenerse en `0` (lo cual comprueba que el branch `BGT` funcionó esquivando la suma).
- Leer el `$pc` (Program Counter) final: debe apuntar a `0x00000400` (comprobando que el salto incondicional y el salto por registro se completaron).

## Conclusiones: 
Anduvo sin problemas. La resolución de los saltos relativos en instrucciones tipo I (`BGT`, `BEQ`) funcionan correctamente como offsets de palabras y no de bytes. El salto `J` dividió bien la dirección destino para ajustarse al formato de palabra, y el salto mediante registro `JR` permitió transferir el control a un espacio arbitrario validando los modos de direccionamiento. Aquí integramos nuevamente el `ADDI` gracias a que el bug fue solucionado.

---

# Caso 5: Subrutinas y Saltos con Enlace (Function Calls)
## Descripción: Qué estoy testeando
Se testea el mecanismo de llamada a subrutinas y retorno. Al ejecutar un salto con enlace (`JAL`), la CPU debe guardar automáticamente la dirección de retorno (`PC + 4`) en el registro `$31` (`$ra`). Luego, la subrutina debe poder retornar el control al programa principal leyendo dicho registro.
## Instrucciones: instrucciones que usé durante el test
`JAL` (Jump And Link), `JALR` (Jump And Link Register), `J` (Jump)

## Precondiciones:
- Reiniciar el simulador y acomodar el PC en 0 (`reset` -> `set pc 0x00000000`).
- El código principal se cargará en `0x00000000` y la subrutina se ubicará lejos en memoria, en `0x00000800`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set [0x00000000] 0x18000200
set [0x00000004] 0x10000300
set [0x00000800] 0x07C0000F
```

## Code
**Ensamblador MIPS/STX4:**
```mips
# --- MAIN PROGRAM (Dirección 0x00000000) ---
JAL 0x00000800       # Salta a 0x0800 y guarda PC+4 (0x04) en $ra ($31)
J   0x00000C00       # [Retorno de subrutina] Salta al final del programa para terminar

# --- SUBRUTINA (Dirección 0x00000800) ---
JALR $31, $0         # Salta a la dirección en $31 (0x04), volviendo al MAIN. Guarda PC+4 en $0 (descartado).
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x18000200  # JAL  0x00000800 (Opcode 3, addr=0x200)
0x00000004 : 0x10000300  # J    0x00000C00 (Opcode 2, addr=0x300)
...
0x00000800 : 0x07C0000F  # JALR $31, $0    (Opcode 0, rs=31, rt=0, func=15)
```

## Postcondiciones:
- En lugar de correr paso a paso por cantidad, usaremos `step 3`.
- Esto ejecutará: 1) El JAL a la subrutina, 2) El JALR devolviendo el control al main, 3) El J final.
- Chequear el `$pc`: debería terminar en `0x00000C00`.
- Chequear el registro `$31`: debe valer `0x00000004` (la dirección de retorno guardada por el JAL).

## Conclusiones:
Funcionó excelente. El salto con enlace (`JAL`) guardó exitosamente la dirección de retorno (`PC + 4`) en `$31` y saltó a la subrutina. Luego, el retorno con `JALR` leyó correctamente ese registro y devolvió el control al programa principal sin problemas, demostrando que el hardware para el manejo de subrutinas está en orden.

---

# Caso 6: Multiplicación, División y Resto (ALU Avanzada)
## Descripción: Qué estoy testeando
Se testean las operaciones aritméticas complejas de la ALU.
## Instrucciones: instrucciones que usé durante el test
`MUL` (Multiply), `DIV` (Divide), `REST` (Remainder)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con el valor `20` (`0x00000014`): `set r10 0x00000014`.
- Setear `$11` con el valor `3` (`0x00000003`): `set r11 0x00000003`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000014
set r11 0x00000003
set [0x00000000] 0x0296C015
set [0x00000004] 0x0314D018
set [0x00000008] 0x0316E01A
```

## Code
**Ensamblador MIPS/STX4:**
```mips
MUL  $12, $10, $11   # $12 = 20 * 3 = 60 (0x3C)
DIV  $13, $12, $10   # $13 = 60 / 20 = 3 (0x03)
REST $14, $12, $11   # $14 = 60 % 3 = 0 (0x00)
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0296C015  # MUL  $12, $10, $11 (Opcode 0, rs=10, rt=11, rd=12, func=21)
0x00000004 : 0x0314D018  # DIV  $13, $12, $10 (Opcode 0, rs=12, rt=10, rd=13, func=24)
0x00000008 : 0x0316E01A  # REST $14, $12, $11 (Opcode 0, rs=12, rt=11, rd=14, func=26)
```

## Postcondiciones:
- Avanzar con `step 3`.
- Revisar registros:
  - `$12` debe valer `0x0000003C` (60)
  - `$13` debe valer `0x00000003` (3)
  - `$14` debe valer `0x00000000` (0)

## Conclusiones:
Todo perfecto. Las operaciones aritméticas complejas de la ALU demostraron funcionar correctamente. La multiplicación calculó el producto de forma precisa, y tanto la división como el resto aislaron sus respectivos resultados matemáticos sin sobreescribir ni arrastrar bits extraños entre los registros involucrados.

---

# Caso 7: Acceso a Memoria de Media Palabra (Half Words) y Shifts Lógicos
## Descripción: Qué estoy testeando
Se testea el almacenamiento y carga en memoria limitados a 16 bits (`Half Words`), y cómo la ALU maneja los desplazamientos de bits lógicos (sin respetar ni extender el signo).
## Instrucciones: instrucciones que usé durante el test
`SH` (Store Half), `LH` (Load Half), `LW` (Load Word), `SLL` (Shift Left Logical), `SRL` (Shift Right Logical)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Base de memoria `$15` en `0x00000800`.
- Dato de prueba `$16` con el valor `0x8899AABB`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r15 0x00000800
set r16 0x8899AABB
set [0x00000000] 0x53E00000
set [0x00000004] 0x63E20000
set [0x00000008] 0x43E40000
set [0x0000000C] 0x00233200
set [0x00000010] 0x00234201
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SH  $16, 0($15)      # M[$15][15:0] = 0xAABB. Solo guarda la media palabra inferior.
LH  $17, 0($15)      # Lee los 16 bits de 0x0800 y los extiende con signo (0xFFAABB).
LW  $18, 0($15)      # Lee la palabra entera (los 16 bits superiores deberían seguir en 0).
SLL $19, $17, 4      # $19 = $17 << 4. Llena con 0 por la derecha.
SRL $20, $17, 4      # $20 = $17 >> 4. Llena con 0 por la izquierda (desplazamiento lógico).
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x53E00000  # SH  $16, 0($15) (Opcode 10, rs=15, rt=16, imm=0)
0x00000004 : 0x63E20000  # LH  $17, 0($15) (Opcode 12, rs=15, rt=17, imm=0)
0x00000008 : 0x43E40000  # LW  $18, 0($15) (Opcode 8,  rs=15, rt=18, imm=0)
0x0000000C : 0x00233200  # SLL $19, $17, 4 (Opcode 0, rt=17, rd=19, aux=4, func=0)
0x00000010 : 0x00234201  # SRL $20, $17, 4 (Opcode 0, rt=17, rd=20, aux=4, func=1)
```

## Postcondiciones:
- Avanzar con `step 5`.
- Leer `$17`: debe valer `0xFFFFAABB` (ya que `AABB` tiene el MSB en 1, LH extiende el signo).
- Leer `$18`: debe valer `0x0000AABB` (si la RAM estaba en 0 antes del SH). *Nota: Si ejecutaste el Caso 5 previamente en la misma sesión, el registro te dará `0x07C0AABB`. Esto ocurre porque en el Caso 5 escribiste la instrucción JALR (`0x07C0000F`) en la dirección `0x0800`, y como `SH` solo sobreescribe los 16 bits inferiores (cambiando `000F` por `AABB`), los 16 bits superiores (`07C0`) se mantienen intactos. ¡Esto es una prueba excelente de que SH funciona perfecto!*
- Leer `$19`: debe valer `0xFFFAABB0` (se corrió 4 bits a la izquierda, perdiendo una F y sumando un 0).
- Leer `$20`: debe valer `0x0FFFFAAB` (se corrió 4 bits a la derecha, rellenando con un 0 a la izquierda por ser SRL).

---

# Caso 8: Operaciones Lógicas R-Type y SLT
## Descripción: Qué estoy testeando
Se testean comparaciones entre dos registros y la resolución de las operaciones lógicas booleanas AND y OR, fundamentales para cualquier branching avanzado o manipulación de bits.
## Instrucciones: instrucciones que usé durante el test
`SLT` (Set Less Than), `SLTU` (Set Less Than Unsigned), `AND`, `OR`

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con `-1` (`0xFFFFFFFF`): `set r10 0xFFFFFFFF`.
- Setear `$11` con `10` (`0x0000000A`): `set r11 0x0000000A`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0xFFFFFFFF
set r11 0x0000000A
set [0x00000000] 0x0296C00C
set [0x00000004] 0x0296D00D
set [0x00000008] 0x0296E008
set [0x0000000C] 0x0296F009
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SLT  $12, $10, $11   # ¿-1 < 10 (con signo)? Verdadero. $12 = 1.
SLTU $13, $10, $11   # ¿0xFFFFFFFF < 10 (sin signo)? Falso. $13 = 0.
AND  $14, $10, $11   # 0xFFFFFFFF AND 0x0A = 0x0A (10)
OR   $15, $10, $11   # 0xFFFFFFFF OR 0x0A = 0xFFFFFFFF (-1)
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0296C00C  # SLT  $12, $10, $11 (Opcode 0, rs=10, rt=11, rd=12, func=12)
0x00000004 : 0x0296D00D  # SLTU $13, $10, $11 (Opcode 0, rs=10, rt=11, rd=13, func=13)
0x00000008 : 0x0296E008  # AND  $14, $10, $11 (Opcode 0, rs=10, rt=11, rd=14, func=8)
0x0000000C : 0x0296F009  # OR   $15, $10, $11 (Opcode 0, rs=10, rt=11, rd=15, func=9)
```

## Postcondiciones:
- Avanzar con `step 4`.
- Revisar registros:
  - `$12` debe valer `1`
  - `$13` debe valer `0`
  - `$14` debe valer `0x0000000A`
  - `$15` debe valer `0xFFFFFFFF`

Conclusiones:Funciona. El test validó que la ALU del simulador responde de manera exacta a las instrucciones Tipo R (Opcode 0) usando el campo func.Por un lado, se comprobó el manejo de datos con y sin signo: SLT entendió que -1 (0xFFFFFFFF) es menor que 10, dando verdadero (12=1). En cambio, SLTU los comparó como valores sin signo (2^32−1>10), dando falso (13=0).Por el otro, las operaciones lógicas bit a bit corrieron impecable: AND aplicó la máscara de bits aislando el 10, y OR saturó el registro manteniendo el -1 (0xFFFFFFFF). Todo se consolidó en el banco de registros sin fallas en el procesador.

---

# Caso 9: Aritmética y Lógica con Inmediatos
## Descripción: Qué estoy testeando
Se testean las operaciones combinadas con inmediatos (tanto aritméticas como lógicas) para evaluar el formato I, el formato L y las distintas extensiones que realiza la CPU de forma simultánea. 
## Instrucciones: instrucciones que usé durante el test
`ADD` (Add), `ANDI` (And Immediate), `XORI` (Exclusive Or Immediate), `SLTIU` (Set Less Than Immediate Unsigned)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con `10` (`0x0000000A`): `set r10 0x0000000A`.
- Setear `$11` con `20` (`0x00000014`): `set r11 0x00000014`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x0000000A
set r11 0x00000014
set [0x00000000] 0x0296C01C
set [0x00000004] 0x231A000F
set [0x00000008] 0x335C00FF
set [0x0000000C] 0xBB9E0100
```

## Code
**Ensamblador MIPS/STX4:**
```mips
ADD   $12, $10, $11   # $12 = 10 + 20 = 30 (0x1E)
ANDI  $13, $12, 0x0F  # $13 = 30 & 15 = 14 (0x0E). ANDI usa extensión por ceros.
XORI  $14, $13, 0xFF  # $14 = 14 XOR 255 = 241 (0xF1). XORI usa extensión por ceros.
SLTIU $15, $14, 0x100 # ¿241 < 256 (sin signo)? Sí. $15 = 1.
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0296C01C  # ADD   $12, $10, $11  (Opcode 0, rs=10, rt=11, rd=12, func=28)
0x00000004 : 0x231A000F  # ANDI  $13, $12, 0x0F (Opcode 4, rs=12, rt=13, h=0, imm=0x0F)
0x00000008 : 0x335C00FF  # XORI  $14, $13, 0xFF (Opcode 6, rs=13, rt=14, h=0, imm=0xFF)
0x0000000C : 0xBB9E0100  # SLTIU $15, $14, 256  (Opcode 23, rs=14, rt=15, imm=0x100)
```

## Postcondiciones:
- Avanzar la CPU con `step 4`.
- Revisar registros:
  - `$12` debe valer `0x0000001E` (30)
  - `$13` debe valer `0x0000000E` (14)
  - `$14` debe valer `0x000000F1` (241)
  - `$15` debe valer `1`

## Conclusiones:
Fallo documentado. Tras la última actualización del simulador se comprobó que el bug oficial de `ANDI` fue solucionado (ahora enmascara correctamente retornando 0 al operar contra 0), pero insólitamente se rompió la instrucción `ADD`. Al realizar `ADD $12, $10, $11` (10 + 20), el registro `$12` devolvió `0x00` en lugar de `30`. Sospechamos que la ALU está aplicando internamente una compuerta `AND` (10 AND 20 = 0) o simplemente fallando al sumar. El resto de las instrucciones lógicas (`XORI`, `SLTIU`) funcionó a la perfección.

---

# Caso 10: Desplazamientos Dinámicos (Variable Shifts)
## Descripción: Qué estoy testeando
Se testean los desplazamientos a nivel de bit pero utilizando el valor dinámico proveniente de un registro fuente en lugar de un inmediato constante. 
## Instrucciones: instrucciones que usé durante el test
`SLLR` (Shift Left Logical Variable), `SRLR` (Shift Right Logical Variable), `SRAR` (Shift Right Arithmetic Variable)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear la cantidad de desplazamiento en `$10` con `4`: `set r10 0x00000004`.
- Setear el dato a desplazar en `$11` con `0x80000008`: `set r11 0x80000008`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000004
set r11 0x80000008
set [0x00000000] 0x0296C003
set [0x00000004] 0x0296D004
set [0x00000008] 0x0296E005
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SLLR $12, $10, $11   # $12 = $11 << $10 (0x80000008 << 4)
SRLR $13, $10, $11   # $13 = $11 >> $10 (0x80000008 >> 4, sin propagar signo)
SRAR $14, $10, $11   # $14 = $11 >> $10 (0x80000008 >> 4, propagando signo)
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0296C003  # SLLR $12, $10, $11 (Opcode 0, rs=10, rt=11, rd=12, func=3)
0x00000004 : 0x0296D004  # SRLR $13, $10, $11 (Opcode 0, rs=10, rt=11, rd=13, func=4)
0x00000008 : 0x0296E005  # SRAR $14, $10, $11 (Opcode 0, rs=10, rt=11, rd=14, func=5)
```

## Postcondiciones:
- Avanzar con `step 3`.
- Revisar registros:
  - `$12` debe valer `0x00000080`
  - `$13` debe valer `0x08000000` (se metió un cero a la izquierda)
  - `$14` debe valer `0xF8000000` (se propagó el 1 del signo)

## Conclusiones:
Anduvo perfecto. Las instrucciones SLLR, SRLR y SRAR responden correctamente capturando los 5 bits menos significativos del registro especificado y aplicando los corrimientos estipulados sin pérdida de información, probando el control de los shifters dinámicos.

---

# Caso 11: Multiplicación y División Unsigned / High
## Descripción: Qué estoy testeando
Se testea el acceso a la parte alta (`High`) del resultado de la multiplicación y la división aritmética considerando los operandos estrictamente sin signo.
## Instrucciones: instrucciones que usé durante el test
`MULHU` (Multiply High Unsigned), `DIVU` (Divide Unsigned), `RESTU` (Remainder Unsigned)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con `0xFFFFFFFF` (4294967295): `set r10 0xFFFFFFFF`.
- Setear `$11` con `2`: `set r11 0x00000002`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0xFFFFFFFF
set r11 0x00000002
set [0x00000000] 0x0296D017
set [0x00000004] 0x0296E019
set [0x00000008] 0x0296F01B
```

## Code
**Ensamblador MIPS/STX4:**
```mips
MULHU $13, $10, $11  # (0xFFFFFFFF * 2) = 0x1FFFFFFFE. $13 extrae la parte alta = 0x00000001
DIVU  $14, $10, $11  # 4294967295 / 2 = 2147483647 (0x7FFFFFFF)
RESTU $15, $10, $11  # 4294967295 % 2 = 1
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x0296D017  # MULHU $13, $10, $11 (Opcode 0, rs=10, rt=11, rd=13, func=23)
0x00000004 : 0x0296E019  # DIVU  $14, $10, $11 (Opcode 0, rs=10, rt=11, rd=14, func=25)
0x00000008 : 0x0296F01B  # RESTU $15, $10, $11 (Opcode 0, rs=10, rt=11, rd=15, func=27)
```

## Postcondiciones:
- Avanzar con `step 3`.
- Revisar registros:
  - `$13` debe valer `1` (el carry-over o high part de la multiplicación de 64 bits)
  - `$14` debe valer `0x7FFFFFFF`
  - `$15` debe valer `1`

## Conclusiones:
Fallo documentado. Tras la actualización del simulador se confirmó una advertencia del manual oficial: la instrucción `ANDI` está severamente bugueada. Al realizar `ANDI  $13, $12, 0x0F`, en lugar de aplicar la máscara lógica de bits (AND), el simulador parece ejecutar un `ORI` o cargar directamente el operando inmediato ignorando el registro fuente. Debido a este fallo encadenado, los valores matemáticos de la prueba en `$13` y `$14` no fueron los esperados. El resto de las instrucciones lógicas funcionó, pero `ANDI` se reporta como inoperativa para su propósito real.

---

# Caso 12: Condiciones de Salto Faltantes
## Descripción: Qué estoy testeando
Se testean todas las lógicas de bifurcación de formato I que no probamos anteriormente. Comprobaremos si saltan o no esquivando incrementos de registros.
## Instrucciones: instrucciones que usé durante el test
`BNE` (Branch Not Equal), `BLT` (Branch Less Than), `BLE` (Branch Less or Equal), `BGE` (Branch Greater or Equal)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con `5`: `set r10 0x00000005`.
- Setear `$11` con `10`: `set r11 0x0000000A`.
- Setear `$12` con `5`: `set r12 0x00000005`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r10 0x00000005
set r11 0x0000000A
set r12 0x00000005
set [0x00000000] 0x8A960001
set [0x00000004] 0x081A0001
set [0x00000008] 0x92D40001
set [0x0000000C] 0x081C0001
set [0x00000010] 0xA2980001
set [0x00000014] 0x081E0001
set [0x00000018] 0xAA960001
set [0x0000001C] 0x08200001
```

## Code
**Ensamblador MIPS/STX4:**
```mips
BNE $10, $11, 1      # ¿5 != 10? Sí. Toma el salto de 1 (esquiva el ADDI).
ADDI $13, $0, 1      # [Salteado]
BLT $11, $10, 1      # ¿10 < 5? No. No toma el salto y sigue.
ADDI $14, $0, 1      # Se ejecuta. $14 = 1.
BLE $10, $12, 1      # ¿5 <= 5? Sí. Toma el salto de 1.
ADDI $15, $0, 1      # [Salteado]
BGE $10, $11, 1      # ¿5 >= 10? No. Sigue de largo.
ADDI $16, $0, 1      # Se ejecuta. $16 = 1.
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x8A960001  # BNE  $10, $11, 1 (Opcode 17)
0x00000004 : 0x081A0001  # ADDI $13, $0, 1
0x00000008 : 0x92D40001  # BLT  $11, $10, 1 (Opcode 18)
0x0000000C : 0x081C0001  # ADDI $14, $0, 1 
0x00000010 : 0xA2980001  # BLE  $10, $12, 1 (Opcode 20)
0x00000014 : 0x081E0001  # ADDI $15, $0, 1 
0x00000018 : 0xAA960001  # BGE  $10, $11, 1 (Opcode 21)
0x0000001C : 0x08200001  # ADDI $16, $0, 1
```

## Postcondiciones:
- Avanzar con `step 8`.
- Revisar registros:
  - `$13` debe ser `0`
  - `$14` debe ser `1`
  - `$15` debe ser `0`
  - `$16` debe ser `1`

## Conclusiones:
Resultados positivos. La avalancha de saltos demostró que el comparador para las bifurcaciones BNE, BLT, BLE y BGE funciona con precisión al evaluar las distintas condiciones, ignorando la ejecución cuando la condición es falsa y tomando el branch mediante el cálculo PC=PC+4+offset cuando es verdadera.

---

# Caso 13: Cargas de Bytes y Direccionamiento Indexado
## Descripción: Qué estoy testeando
Se testean las cargas de bytes (signada vs no signada) y las instrucciones indexadas (terminadas en X) las cuales utilizan el contenido de dos registros que suman para conformar la dirección de memoria.
## Instrucciones: instrucciones que usé durante el test
`LB` (Load Byte), `LBU` (Load Byte Unsigned), `LBX` (Load Byte Indexed), `LWX` (Load Word Indexed)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Puntero base `$15` en `0x0800`: `set r15 0x00000800`.
- Dato de prueba `$10` con `0x8899AABB`: `set r10 0x8899AABB`.
- Indice de memoria `$14` con `2`: `set r14 0x00000002`.

**Comandos del Debugger (Telnet):**
```bash
reset
set pc 0x00000000
set r15 0x00000800
set r10 0x8899AABB
set r14 0x00000002
set [0x00000000] 0x4BD40000
set [0x00000004] 0x73D60000
set [0x00000008] 0x7BD80000
set [0x0000000C] 0x03DCD012
set [0x00000010] 0x03E00014
```

## Code
**Ensamblador MIPS/STX4:**
```mips
SW  $10, 0($15)      # Guarda 0x8899AABB en la dirección 0x0800
LB  $11, 0($15)      # Lee el byte inferior (0xBB). Como el MSB de 0xBB es 1, extiende signo: $11 = 0xFFFFFFBB
LBU $12, 0($15)      # Lee el mismo byte, pero rellena con 0s: $12 = 0x000000BB
LBX $13, $15, $14    # Lee byte de la direcc ($15 + $14) = 0x0802. Ahí debería estar el 0x99. Lo extiende: $13 = 0xFFFFFF99. (OJO: rs es base, rd es index, rt es dest).
LWX $0,  $15, $16    # Lee la palabra entera desde ($15 + $0) = 0x0800. $16 = 0x8899AABB
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0x00000000 : 0x4BD40000  # SW  $10, 0($15)   (Opcode 9)
0x00000004 : 0x73D60000  # LB  $11, 0($15)   (Opcode 14)
0x00000008 : 0x7BD80000  # LBU $12, 0($15)   (Opcode 15)
0x0000000C : 0x03DCD012  # LBX $15, $13, $14 (Opcode 0, rs=15, rt=13, rd=14, func=18)
0x00000010 : 0x03E00014  # LWX $15, $16, $0  (Opcode 0, rs=15, rt=16, rd=0, func=20)
```

## Postcondiciones:
- Avanzar con `step 5`.
- Revisar registros:
  - `$11` debe ser `0xFFFFFFBB`
  - `$12` debe ser `0x000000BB`
  - `$13` debe ser `0xFFFFFF99` (Asumiendo Little Endian)
  - `$16` debe ser `0x8899AABB`

## Conclusiones:
Anduvo impecable. Las lecturas de memoria a nivel de byte lograron procesarse correctamente con su respectiva extensión de signo (LB) y cero (LBU). Adicionalmente, las operaciones con modo de direccionamiento indexado (LBX, LWX) calcularon correctamente la dirección efectiva sumando ambos registros (base + offset dinámico) permitiendo abstraerse de los inmediatos estáticos.

---

# Caso 14: Registros Especiales (Bug Detectado)
## Descripción: Qué estoy testeando
Se testea el acceso de lectura y escritura a los registros especiales de la CPU, como el `$ecr` o `$psw`, utilizando las instrucciones dedicadas `CFS` y `CTS`.
## Instrucciones: instrucciones que usé durante el test
`CTS` (Copy To Special), `CFS` (Copy From Special)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).
- Setear `$10` con `0x12345678`: `set r10 0x12345678`.
- Setear `$11` con `0x00000000`: `set r11 0x00000000`.

**Code:**
```bash
reset
set pc 0x00000000
set r10 0x12345678
set r11 0x00000000
set [0x00000000] 0x02800087
set [0x00000004] 0x02C00086
```

## Postcondiciones:
- Avanzar con `step 2`.

## Conclusiones:
Fallo documentado (Persistente). A pesar de la actualización del simulador, se comprobó que el soporte para la copia desde y hacia registros especiales sigue inoperativo. Al ejecutar `CTS` y `CFS`, la máquina no levanta excepciones pero tampoco realiza la transferencia de datos (el registro destino `$11` mantiene su valor `0x00000000`), comportándose efectivamente como instrucciones vacías (NOPs). El error debe continuar siendo notificado al equipo de desarrollo para su futura revisión.

---

# Caso 15: Excepciones y Retornos (Bug Detectado)
## Descripción: Qué estoy testeando
Se testea la capacidad del procesador para levantar una excepción manual (`TRAP`), guardar el estado (`EPC = PC + 4`) y saltar a la tabla de vectores, así como retornar exitosamente al programa principal mediante la instrucción de retorno de excepción (`RFT`).
## Instrucciones: instrucciones que usé durante el test
`TRAP`, `RFT` (Return From Trap)

## Precondiciones:
- Reiniciar el simulador (`reset` -> `set pc 0`).

**Code:**
```bash
reset
set pc 0x00000000
set [0x00000000] 0x10000002
set [0x00000004] 0x00000800
set [0x00000008] 0x000000A0
set [0x00000800] 0x00000021
```

## Postcondiciones:
- Avanzar con `step 3`.
- Revisar el estado de la CPU. El `$pc` debería valer `0x0000000C`.

## Conclusiones:
Fallo documentado (Persistente). Al igual que con los registros especiales, se corroboró empíricamente que en la nueva versión el procesador sigue ignorando por completo la instrucción de excepción (TRAP). En lugar de resguardar el contexto (`EPC`) y saltar al vector de la dirección `0x0800`, el `PC` simplemente continuó de largo hacia las siguientes direcciones de memoria sin levantar errores ni excepciones, comportándose como un NOP e ignorando el llamado a interrupción. Se documenta la prueba para constancia y se excluye de las capacidades estables del microprocesador.

# Tarea 2: ROM de Booteo y Código Mínimo

## Descripción
En esta segunda parte del trabajo práctico, se diseña una pequeña ROM de booteo (Bootloader) para la CPU STX4. 
El simulador inicializa el `PC` (Program Counter) en `0xF0000000` tras un reinicio del sistema. Nuestro objetivo es inyectar un código de inicialización en esa dirección que prepare el entorno básico (puntero de pila y puntero global) antes de saltar al espacio de memoria del usuario (`0x00000000`) para ejecutar el código principal.

## Instrucciones de la ROM
- Inicializar Stack Pointer (`$sp` / `$29`) apuntando a la dirección `0x1000`.
- Inicializar Global Pointer (`$gp` / `$28`) apuntando a la dirección `0x0800`.
- Realizar un salto incondicional (`J`) a la dirección base `0x00000000`.

## Código de Usuario (Mínimo)
- Cargar un valor de prueba (ej. `42`) en un registro (`$10`) para comprobar que el salto fue exitoso.
- Entrar en un loop infinito (`J` a sí mismo) para evitar que la CPU lea basura en las siguientes direcciones de memoria.

## Precondiciones
- Reiniciar la CPU (`reset`).
- **Importante:** NO se debe alterar manualmente el PC. Dejar que arranque desde `0xF0000000`.

**Comandos del Debugger (Telnet):**
```bash
reset

# --- ROM BOOTLOADER (Memoria del Sistema) ---
set [0xF0000000] 0x083A1000
set [0xF0000004] 0x08380800
set [0xF0000008] 0x10000000

# --- CÓDIGO DE USUARIO (Memoria Base) ---
set [0x00000000] 0x0814002A
set [0x00000004] 0x10000001
```

## Code
**Ensamblador MIPS/STX4:**
```mips
# ROM EN 0xF0000000
ADDI $29, $0, 4096    # $sp = 4096 (0x1000)
ADDI $28, $0, 2048    # $gp = 2048 (0x0800)
J    0x00000000       # Salto incondicional al inicio del código de usuario

# USER CODE EN 0x00000000
ADDI $10, $0, 42      # $10 = 42 (0x2A). Operación de prueba.
J    0x00000004       # Loop infinito (salta a su propia dirección, offset 1 palabra).
```

**Tabla de Referencia (Código Máquina):**
```text
Dirección  : Código Hex  # Mnemónico Traducido
0xF0000000 : 0x083A1000  # ADDI $29, $0, 4096
0xF0000004 : 0x08380800  # ADDI $28, $0, 2048
0xF0000008 : 0x10000000  # J    0x00000000
...
0x00000000 : 0x0814002A  # ADDI $10, $0, 42
0x00000004 : 0x10000001  # J    0x00000004
```

## Postcondiciones
- Avanzar 5 ciclos con `step 5`. Esto ejecutará las 3 instrucciones de la ROM, el salto, la instrucción de usuario, y caerá en el loop infinito.
- Revisar registros:
  - `$29` (`$sp`) debe valer `0x00001000`.
  - `$28` (`$gp`) debe valer `0x00000800`.
  - `$10` debe valer `0x0000002A` (42).
  - El `$pc` debe quedarse bloqueado repitiéndose en `0x00000004`.
