# Unidad 5
## Bitácora de proceso de aprendizaje


### Ventajas y desventajas del formato binario
**Ventajas**

**1. Menor tamaño de datos**

El protocolo ASCII puede ocupar muchos bytes dependiendo del número.
Por ejemplo:

"500,524,True,False\n" ≈ 19 bytes

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

1. No es legible para humanos

Mientras que ASCII se puede leer fácilmente:

500,524,True,False

El formato binario se ve como bytes incomprensibles.

2. Más difícil de depurar

Para entender los datos se necesitan herramientas como:

- analizadores hexadecimales

- scripts de decodificación

3. Dependencia del formato

Si el receptor interpreta mal el formato '(>2h2B)', los datos se leerán incorrectamente.

## Bitácora de aplicación 


## Bitácora de reflexión
