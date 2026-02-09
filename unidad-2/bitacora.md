# Unidad 2

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
Diagrama de PlantUML
```
@startuml
[*] --> Configuracion

Configuracion : entry/ count = inicia 20
Configuracion : A / incrementar count (max 25)
Configuracion : B / decrementar count (min 15)
Configuracion : S / armar temporizador
Configuracion --> Contando : S

Contando : iniciar timer (1000ms)= 1s
Contando : Timeout --, mostrar
Contando --> Alarma : Timeout [count == 0]
Contando --> Contando : Timeout [count > 0]

Alarma : entry/ mostrar calavera, sonar speaker
Alarma : A / reiniciar
Alarma --> Configuracion : A is pressed

@enduml
```


## Bitácora de reflexión

