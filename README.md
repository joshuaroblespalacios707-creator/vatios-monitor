# ⚡ vatios.sh — Monitor de Consumo Eléctrico y Costo en Tiempo Real (Linux)

Herramienta de terminal en **Bash** para Linux que muestra, casi en vivo, cuántos vatios consume tu PC y cuánto cuesta por hora en pesos mexicanos (MXN).

> **Importante:** el resultado es una **estimación**, no una medición de pared. La GPU y el CPU se leen de los sensores del hardware, pero el resto del equipo (placa, RAM, SSDs, refrigeración) se aproxima con un valor fijo. Ver [Limitaciones](#️-limitaciones-y-notas).

---

## 📌 Descripción

`vatios.sh` lee el consumo eléctrico de la **GPU (NVIDIA)** y del **CPU** directamente desde sus interfaces de telemetría, le suma una estimación fija para los componentes restantes, y convierte el total (W) a costo por hora usando una tarifa configurable de la CFE.

**Características:**
- Refresco continuo en la terminal, sin parpadeo
- Desglose por componente: GPU, CPU y base del sistema
- Costo instantáneo por hora en MXN
- Salida limpia con `Ctrl + C` (restaura el cursor)
- Funciones separadas por fuente de datos, fáciles de adaptar

---

## 🛠️ Arquitectura y Fuentes de Datos

```
+-----------------------------------------------------------------------+
|                         CAPA DE APLICACIÓN                            |
|                Monitor con bucle `while` + ANSI (`tput`)              |
+-----------------------------------+-----------------------------------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
+-----------v-----------+                       +-----------v-----------+
|      GPU (NVIDIA)     |                       |          CPU          |
|   `nvidia-smi` CLI    |                       |   RAPL (kernel Linux) |
|     (power.draw)      |                       | /sys/class/powercap/  |
+-----------+-----------+                       +-----------+-----------+
            |                                               |
            +-----------------------+-----------------------+
                                    |
+-----------------------------------v-----------------------------------+
|                    BASE DEL SISTEMA (45 W, estimado)                  |
|   Placa base (chipset/VRM) + 2x SSD NVMe + RAM + bomba/ventiladores   |
+-----------------------------------------------------------------------+
```

| Fuente | Cómo se obtiene | Tipo de dato |
|---|---|---|
| **GPU (NVIDIA RTX 3060)** | `nvidia-smi --query-gpu=power.draw`, que devuelve el consumo en **vatios** | Reportado por el driver |
| **CPU** | Interfaz RAPL del kernel: se lee `/sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj` dos veces con 0.2 s de diferencia, y se divide la energía consumida (µJ) entre el tiempo transcurrido | Reportado por el hardware |
| **Base del sistema** | Constante de 45 W (placa, RAM, 2 SSDs NVMe, bomba y ventiladores de la AIO) | **Estimado** |
| **Valores de respaldo** | 15 W si no hay `nvidia-smi`, 25 W si no existe la interfaz RAPL | Valores fijos, **no son mediciones** |

---

## 📋 Requisitos

- Linux con `bash`, `awk` y `tput` (paquete `ncurses`)
- **GPU NVIDIA** con el driver propietario instalado (opcional; sin él se usa el valor de respaldo)
- **CPU con soporte RAPL** (Intel; en AMD depende del kernel y del hardware)
- Permisos de lectura sobre `energy_uj`: en kernels recientes este archivo solo es legible por `root`. Si el CPU aparece en `0 W`, ejecuta el script con `sudo`.

---

## 💻 Código Fuente (`vatios.sh`)

Ubicación sugerida: `~/laboratorio/vatios.sh`

```bash
#!/bin/bash

# ==============================================================================
# CONFIGURACIÓN GENERAL
# ==============================================================================
TARIFA_CFE=1.80  # Precio por kWh en Pesos Mexicanos (MXN)
BASE_SISTEMA=45  # Consumo base estimado en Watts (Motherboard + RAM + 2x SSDs + AIO)

# ==============================================================================
# FUNCIONES DE CAPTURA DE TELEMETRÍA
# ==============================================================================

# Captura de consumo de la GPU mediante nvidia-smi
obtener_watts_gpu() {
    if command -v nvidia-smi &> /dev/null; then
        nvidia-smi --query-gpu=power.draw --format=csv,noheader,nounits | awk '{print int($1)}'
    else
        echo 15
    fi
}

# Captura de consumo de la CPU leyendo directamente del Kernel (RAPL Interface)
obtener_watts_cpu() {
    local rapl_path="/sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj"
    if [ -f "$rapl_path" ]; then
        local e1=$(cat "$rapl_path")
        sleep 0.2
        local e2=$(cat "$rapl_path")
        echo $(( (e2 - e1) / 200000 ))
    else
        echo 25
    fi
}

# ==============================================================================
# INICIALIZACIÓN DE LA TERMINAL Y TRAP DE SEÑALES
# ==============================================================================
clear
tput civis  # Oculta el cursor para evitar parpadeo visual

# Captura Ctrl+C (SIGINT) para restaurar la terminal de forma limpia al salir
trap "tput cnorm; echo -e '\nSaliendo del monitor de energía...'; exit 0" INT

# ==============================================================================
# BUCLE PRINCIPAL DE MONITORIZACIÓN (0.8s)
# ==============================================================================
while true; do
    w_gpu=$(obtener_watts_gpu)
    w_cpu=$(obtener_watts_cpu)
    w_total=$((w_gpu + w_cpu + BASE_SISTEMA))

    # Cálculo de costo instantáneo por hora en Pesos Mexicanos (MXN)
    costo_hora=$(awk "BEGIN {printf \"%.2f\", ($w_total / 1000) * $TARIFA_CFE}")

    # Posiciona el cursor en la fila 0, columna 0 sin borrar la pantalla completa
    tput cup 0 0

    echo "=========================================="
    echo "   MONITOR DE TELEMETRÍA DE ENERGÍA PC    "
    echo "=========================================="
    printf "  GPU (NVIDIA RTX)  : %3d W               \n" "$w_gpu"
    printf "  CPU (Procesador)  : %3d W               \n" "$w_cpu"
    printf "  Base (Placa/AIO)  : %3d W               \n" "$BASE_SISTEMA"
    echo "------------------------------------------"
    printf "  CONSUMO EN VIVO   : %3d W               \n" "$w_total"
    printf "  COSTO POR HORA    : \$%s MXN/h          \n" "$costo_hora"
    echo "=========================================="
    echo " Presiona [Ctrl + C] para detener el monitor"

    sleep 0.8
done
```

---

## 🚀 Instalación y Uso

```bash
cd ~/laboratorio
nano vatios.sh        # pega el código
chmod +x vatios.sh    # permisos de ejecución
sudo ./vatios.sh      # sudo es necesario para leer RAPL en kernels recientes
```

Para detenerlo, presiona `Ctrl + C`.

El ciclo completo de actualización toma cerca de 1 segundo (0.2 s de la muestra de RAPL + 0.8 s de espera + el tiempo de `nvidia-smi`).

---

## 🧮 Cálculo del Costo

```
Costo por hora (MXN) = (Watts totales / 1000) × Tarifa (MXN por kWh)
```

Con la tarifa de ejemplo (`$1.80/kWh`) y un consumo de 250 W:

```
(250 / 1000) × 1.80 = $0.45 MXN por hora
```

Para estimar el gasto mensual:

```
kWh al mes = (Watts / 1000) × horas por día × 30
Costo al mes = kWh al mes × Tarifa
```

---

## 📊 Estimación de Consumo Mensual

Calculada con la fórmula anterior y una tarifa de **$1.80/kWh**.

| Escenario | Carga de trabajo | Consumo promedio | Horas / día | kWh / mes | Costo mensual estimado |
|---|---|---|---|---|---|
| Reposo / tareas ligeras | Navegación, música, terminal | ~85 W | 6 a 8 | 15.3 – 20.4 | $28 – $37 MXN |
| Uso mixto | Trabajo + 2 hrs de gaming | ~140 W | 6 | 25.2 | ~$45 MXN |
| Gamer intensivo | Simuladores / gaming con GPU al 100% | ~250 – 300 W | 4 a 6 | 30 – 54 | $54 – $97 MXN |
| Encendida 24/7 | En reposo, sin apagar | ~85 W | 24 | 61.2 | ~$110 MXN |

---

## ⚠️ Limitaciones y Notas

- **La tarifa es un supuesto.** La CFE cobra por tramos (básico, intermedio y excedente) y varía según la región y la temporada. Ajusta `TARIFA_CFE` según tu recibo para que el costo sea más realista.
- **La base de 45 W es una estimación.** No incluye las pérdidas de la fuente de poder (que dependen de su eficiencia), el monitor ni los periféricos. El consumo real en la pared suele ser **mayor** que el que muestra el script.
- **Los valores de respaldo no son mediciones.** Si no hay `nvidia-smi` o RAPL, el script muestra 15 W o 25 W fijos sin avisar.
- **Permisos de RAPL.** Sin `sudo` (o sin ajustar permisos), el CPU puede mostrarse en `0 W`.
- **Solo el paquete del CPU.** `intel-rapl:0` reporta el consumo del paquete del procesador, no de todo el sistema. El contador también puede reiniciarse al llegar a su máximo, lo que ocasionalmente produce una lectura negativa.
- **Una sola GPU.** Con varias GPUs, `nvidia-smi` devuelve varias líneas y el cálculo falla.
- **Cómo validar.** Para saber qué tan cerca está de la realidad, compara contra un medidor de consumo de pared (o un enchufe inteligente con medición) durante unas horas.

---

## 💡 Conceptos Clave Utilizados

- **Manejo de señales (`trap`):** restaura el cursor (`tput cnorm`) si el usuario interrumpe el script con `Ctrl + C`.
- **Refresco sin parpadeo (`tput cup 0 0`):** en lugar de borrar la pantalla en cada ciclo con `clear`, coloca el cursor en (0,0) y sobrescribe las líneas.
- **Lectura de contadores de energía (RAPL):** el consumo se calcula como la diferencia de energía entre dos lecturas, dividida entre el tiempo transcurrido.
- **Modularidad:** funciones independientes para GPU y CPU, que permiten cambiar o añadir fuentes de datos sin tocar la interfaz.

---

## 🔮 Mejoras Futuras

- Acumular **kWh y costo de la sesión**, para saber cuánto se gastó realmente en lugar de solo el costo instantáneo por hora
- Guardar las lecturas en un archivo CSV para analizarlas después
- Recibir la tarifa y la base del sistema como argumentos de línea de comandos
- Soporte para varias GPUs y para AMD
- Restaurar el cursor también con `trap ... EXIT`, no solo con `INT`
- Aviso en pantalla cuando se usen valores de respaldo
