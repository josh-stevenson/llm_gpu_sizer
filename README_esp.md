

# Dimensionador y Perfilador de Clústeres de Kubernetes para vLLM ☸️

## Descripción General

El Dimensionador y Perfilador de Clústeres de K8s para vLLM es una herramienta de planificación de capacidad diseñada para Arquitectos de Plataformas e Ingenieros de IA/ML que implementan Modelos de Lenguaje Grande (LLMs) en Kubernetes.

Mientras que las calculadoras básicas de VRAM solo se fijan en los pesos del modelo, la implementación de LLMs en producción requiere navegar por una compleja red de topologías de hardware, crecimiento dinámico de memoria y restricciones de programación (*scheduling*) de Kubernetes. Esta herramienta cierra esta brecha al simular cómo el NVIDIA GPU Operator asigna el hardware (Passthrough, MIG, vGPU) y validar si tu carga de trabajo multimodelo encajará físicamente y funcionará de manera eficiente en tu clúster.

Se ejecuta completamente en tu navegador utilizando WebAssembly (stlite), por lo que no requiere servidores backend ni entornos de Python para alojarse.

---

## Características

* **Soporte Multimodelo:** Planifica la capacidad para hasta 4 *endpoints* concurrentes que ejecutan diferentes modelos.
* **Diagnóstico de Programación de K8s:** Valida la afinidad de nodos, los límites de Paralelismo Tensorial (TP) y los límites de interconexión de hardware.
* **Simulación de GPU Operator:** Modela con precisión los límites de VRAM y las capacidades de GPU Passthrough, Multi-Instance GPU (MIG) y vGPU (*Time-Slicing*).
* **Perfilado de Latencia:** Estima el Tiempo Hasta el Primer Token (TTFT), la Latencia Entre Tokens (ITL) y el Tiempo de Respuesta de Extremo a Extremo (E2E) basándose en el ancho de banda de memoria máximo de tu hardware.

---

## Cómo Usar la Herramienta

### 1. Topología del Clúster (Configuración Global)
Define la realidad física de tu clúster de Kubernetes y cómo la expone el GPU Operator:

* **Nodos de Trabajo y GPUs por Nodo:** Define la disposición física de tu hardware. Esto es crítico porque el grupo de Paralelismo Tensorial de un modelo debe caber en un solo nodo para utilizar enlaces NVLink de alta velocidad.
* **Arquitectura de GPU:** Selecciona tu hardware específico (por ejemplo, H100, RTX Pro 6000) para establecer los límites base de VRAM y ancho de banda de la memoria.
* **Estrategia de Asignación de GPU:**
  * **GPU Passthrough:** Los pods obtienen acceso exclusivo a GPUs físicas completas.
  * **MIG (Multi-Instance GPU):** Particiona físicamente las GPUs en fragmentos de hardware aislados. *(Nota: Los fragmentos MIG carecen de NVLink, por lo que no se admite un Paralelismo Tensorial > 1).*
  * **vGPU (Time-Slicing):** Utiliza segmentación de tiempo a nivel de software para compartir una GPU. Tú defines el número de "Réplicas", lo cual multiplica tus GPUs programables pero divide proporcionalmente el límite de VRAM por cada GPU virtual.

### 2. Configuraciones de Modelos (Pestañas)
Para cada modelo activo que tengas la intención de alojar, configura su perfil específico:

* **Especificaciones del Modelo:** Ingresa los parámetros (ej. 8B, 70B) y la precisión de los pesos (FP16, FP8, INT4).
* **Carga de Trabajo y E/S:** * Define tu promedio de Tokens de Entrada (*Prompt*) y Tokens de Salida (Generación).
  * Establece la Concurrencia (pico de usuarios simultáneos).
  * Ajusta el MBU Asumido (Utilización del Ancho de Banda del Modelo) para estimar qué tan eficientemente el tamaño de tu lote (*batch size*) utiliza el ancho de banda de memoria de la GPU.
* **Programación de K8s:** Asigna un grado específico de Paralelismo Tensorial (el número de GPUs en las que se fragmentará este modelo).

### 3. Veredicto y Diagnósticos
En la parte inferior de la aplicación, el motor agrega tu configuración y ejecuta una simulación de ubicación:

* 🟢 **Listo para Producción:** Todos los modelos encajan dentro de los límites de VRAM, los límites de TP y la topología del clúster.
* 🟡 **Implementación Riesgosa:** Los modelos se programarán, pero están peligrosamente cerca de los límites del hardware (ej. uso de VRAM > 90%), con riesgo de caídas por Falta de Memoria (OOM) bajo carga, o utilizando topologías de software altamente ineficientes.
* 🔴 **Implementación Fallida:** El programador de Kubernetes no logrará ubicar los pods. El motor listará las restricciones específicas violadas (ej. Fallas de Anti-Afinidad, Conflictos MIG o límites estrictos de VRAM).

---

## Fundamentos Matemáticos

Esta herramienta utiliza fórmulas de ingeniería de rendimiento estándar de la industria para calcular la huella de memoria y la latencia:

### Cálculos de Memoria (VRAM)
* **Pesos Estáticos:** Calculados en función del recuento de parámetros y el nivel de cuantización. 
  * **Fórmula:** `Parámetros × Bytes por Parámetro`
* **Caché KV Dinámica:** La memoria necesaria para almacenar los tensores clave (*key*) y valor (*value*) de tokens pasados para evitar cálculos redundantes. 
  * **Fórmula:** $$M_{KV} = 2 \times C_{users} \times T_{seq} \times L_{layers} \times N_{attn\_heads} \times D_{head} \times Z_{bytes}$$
* **Sobrecarga del Motor:** Se reserva una base conservadora de ~4 GB por GPU para el contexto CUDA, el tiempo de ejecución de vLLM y los búferes de activación de PyTorch.

### Perfilado de Latencia
* **Latencia Entre Tokens (ITL):** Debido a que la fase de decodificación está fuertemente limitada por el ancho de banda de la memoria, la velocidad a la que se generan los tokens posteriores depende del peso total y el tamaño de la caché dividido por la Utilización del Ancho de Banda del Modelo (MBU) efectiva.
* **Tiempo Hasta el Primer Token (TTFT):** La fase de llenado previo (*prefill*) está altamente paralelizada y limitada por la capacidad de cómputo. La herramienta utiliza un modelo de regresión lineal empírico para estimar la latencia del *prefill* basándose en el tamaño total del prompt de entrada.
* **Latencia de Extremo a Extremo (E2E):** El recorrido total, calculado como: 
  $$\text{TTFT} + (\text{ITL} \times (\text{Tokens de Salida} - 1))$$
