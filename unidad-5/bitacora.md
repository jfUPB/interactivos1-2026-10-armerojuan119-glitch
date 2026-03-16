# Unidad 5
## Bitácora de proceso de aprendizaje


### Ventajas y desventajas del formato binario
**Ventajas**

**1. Menor tamaño de datos**

El protocolo ASCII puede ocupar muchos bytes dependiendo del número.
Por ejemplo:
```
"500,524,True,False\n" ≈ 19 bytes
```
Mientras que el protocolo binario siempre ocupa 6 bytes.

Esto reduce:

- uso de ancho de banda

- tiempo de transmisión

- carga de procesamiento

**2. Tamaño fijo del paquete**

Cada paquete siempre tiene el mismo tamaño, lo que facilita:

- leer datos del puerto serial

- interpretar correctamente los paquetes

**3. Mayor eficiencia**

Los sistemas embebidos como micro:bit tienen recursos limitados.
Enviar datos binarios evita conversiones innecesarias de texto.

**Desventajas**

**1. No es legible para humanos**

Mientras que ASCII se puede leer fácilmente:
```
500,524,True,False
```
El formato binario se ve como bytes incomprensibles.

**2. Más difícil de depurar**

Para entender los datos se necesitan herramientas como:

- analizadores hexadecimales

- scripts de decodificación

**3. Dependencia del formato**

Si el receptor interpreta mal el formato `(>2h2B)`, los datos se leerán incorrectamente.

---

### Representación del paquete en hexadecimal

Valores dados:
```
xValue = 500
yValue = 524
aState = True
bState = False
```
Formato:
```
>2h2B
```

---

**Convertir cada valor**

xValue = 500

En hexadecimal:
```
500 = 01F4
```
Como `h` usa 2 bytes:
```
01 F4
```

---

yValue = 524

En hexadecimal:
```
524 = 020C
```
En 2 bytes:
```
02 0C
```
---

aState = True

Se convierte a entero:
```
True → 1
```

Como `B` usa 1 byte:

```
01
```
---

bState = False
```
False → 0
```
En 1 byte:
```
00
```

---

**Hexadecimal Final**

Orden según el formato:

`xValue | yValue | aState | bState`

Resultado:
```
01 F4 02 0C 01 00
```
## Bitácora de aplicación 


## Bitácora de reflexión
