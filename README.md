# Análisis Forense de Tráfico de Red: Diagnóstico DNS e ICMP con Wireshark

## 1. Resumen Ejecutivo
Este proyecto documenta el análisis práctico de captura de tráfico de red a nivel de paquetes utilizando el archivo `dns+icmp.pcapng`. El objetivo principal fue auditar el flujo de resolución de nombres DNS mediante búsquedas inversas (PTR), evaluar el desempeño y conectividad a través del protocolo ICMP, e inspeccionar los datos transportados (*payloads*) para descartar vectores de ataque comunes como canales encubiertos (*tunneling*) o exfiltración de información.

---

## 2. Entorno y Herramientas
* **Analizador de Protocolos:** Wireshark
* **Archivo de Captura:** `dns+icmp.pcapng`
* **Dirección IP Cliente (Host):** `192.168.43.9`
* **Servidor DNS Local / Gateway:** `192.168.43.1`
* **Servidores Remotos Objetivos:**
  * `8.8.8.8` (Google Public DNS Primario)
  * `8.8.4.4` (Google Public DNS Secundario)
  * `4.2.2.2` (Level 3 / Lumen DNS)

---

## 3. Análisis Técnico Paso a Paso

### 3.1. Inspección de Protocolo DNS (Reverse Lookups)
Se aplicó el filtro `dns` para evaluar el proceso de traducción de direcciones IP a nombres de dominio cualificados (FQDN).

* **Solicitud PTR (Paquete #1):**
  * **Transacción ID:** `0x528e`
  * **Flags:** Bit *Recursion Desired* activo (`1`), solicitando resolución recursiva al DNS local.
  * **Query:** Consulta de tipo **PTR** (*Pointer Record*) para el recurso `8.8.8.8.in-addr.arpa`.
  * **Observación:** Se identificó la advertencia `DNS response missing`, indicando un reintento/retransmisión de solicitud en el paquete #2.

* **Respuesta PTR (Paquete #3):**
  * **Estado (*Reply Code*):** `No error (0)`, confirmando resolución limpia.
  * **Respuesta Obtención FQDN:** La IP `8.8.8.8` fue mapeada con éxito a `google-public-dns-a.google.com`.
  * **Rendimiento:** Latencia de respuesta (*Response Time*) del servidor local registrada en **5.78 ms**.

---

### 3.2. Diagnóstico de Conectividad e Inspección ICMP
Se aplicó el filtro `icmp` para evaluar el comportamiento de los paquetes *Echo Request* y *Echo Reply*.

* **Análisis de Petición (Paquete #4 - Echo Request):**
  * **Tipo / Código:** `Type 8`, `Code 0` (Solicitud Estándar).
  * **Integridad:** *Checksum* válido (`0xbbb3`).
  * **Identificador / Secuencia:** `ID: 0xd73b`, `Seq: 0`.

* **Análisis de Respuesta (Paquete #5 - Echo Reply):**
  * **Tipo / Código:** `Type 0`, `Code 0` (Respuesta de Eco).
  * **Latencia RTT:** Se midió un tiempo de viaje de ida y vuelta (*Round-Trip Time*) inicial de **492.2 ms** hacia la dirección `8.8.8.8`.

---

## 4. Hallazgos Clave y Análisis de Anomalías

1. **Pérdida de Paquetes en Servidor Secundario (`8.8.4.4`):**
   * Durante la prueba ICMP hacia `8.8.4.4` (paquetes 12 al 15), las solicitudes en los paquetes #12 y #15 marcaron la etiqueta `no response found!`.
   * **Diagnóstico:** Se determinó un **66% de pérdida de paquetes** hacia este host objetivo, lo que indica posible degradación de red intermedia, límites de tasa (*rate-limiting*) de ICMP o reglas de filtrado en firewall.

2. **Evaluación de Seguridad (Descarte de Amenazas):**
   * **Descarte de DNS Tunneling / DGA:** Las consultas mantuvieron una longitud y formato regular sin presencia de cadenas pseudoaleatorias o datos cifrados en la sección de nombres.
   * **Descarte de ICMP Tunneling:** La inspección del panel *Payload* (48 bytes) confirmó patrones por defecto del sistema operativo, descartando la presencia de comandos o exfiltración encubierta de datos.

---

## 5. Conclusiones
El tráfico analizado corresponde a una secuencia legítima de diagnóstico de red donde un host mapea la identidad de servidores DNS públicos e inicie pruebas de conectividad. Aunque la infraestructura principal respondió de forma adecuada, la captura permitió demostrar capacidades de detección de fallos de red (pérdida de paquetes en `8.8.4.4`) e inspección forense de la carga útil de los protocolos para garantizar la seguridad de la comunicación.
