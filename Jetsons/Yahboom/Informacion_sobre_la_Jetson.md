# Ficha Técnica — Jetson Yahboom
**Fecha:** 18 septiembre 2026

## Resumen
Jetson Orin NX (Engineering Reference Dev Kit "Super") de Yahboom, con Ubuntu/JetPack, usada como equipo de IA embebida (visión, LLMs locales, robótica ROS). Acceso remoto vía Tailscale y SSH. Almacenamiento en SSD NVMe de 256GB, con 84G libres actualmente.

---

## 1. Identidad del equipo
| Campo | Valor |
|---|---|
| Hostname | jetson@yahboom |
| Modelo | NVIDIA Jetson Orin NX Engineering Reference Developer Kit "Super" |
| CPU | ARM Cortex-A78AE, 8 núcleos (aarch64) |
| RAM | 15 GB (uso típico ~5GB, resto cache) |
| Swap | 15 GB (zram) |
| Almacenamiento | SSD NVMe 256GB, partición raíz ext4 237G |
| Uso de disco actual | 139G usados / 84G libres (63%) |

## 2. Sistema operativo y plataforma NVIDIA
| Campo | Valor |
|---|---|
| JetPack / L4T | R36.4.7 |
| CUDA | 12.6 |
| cuDNN | 9.20 (para CUDA 12.9) |
| TensorRT | 10.7.0 |
| Python | 3.10.12 |
| PyTorch | 2.5.0 (build NVIDIA), con soporte CUDA confirmado (`cuda disponible: True`) |
| Docker | 29.5.3 |

## 3. Red y acceso remoto
| Campo | Valor |
|---|---|
| IP local (WiFi) | 172.16.230.72 (DHCP, sin IP fija configurada) |
| Interfaz WiFi | wlP1p1s0 |
| Tailscale (VPN) | 100.93.48.128 |
| Docker bridge | 172.17.0.1 |
| CAN bus | can0 activo (uso probable: robótica/vehículo) |
| SSH | activo |
| OpenVPN | servicio habilitado |
| NoMachine (nxserver) | servicio habilitado — escritorio remoto |

## 4. Uso / propósito del equipo
Basado en los servicios y contenedores instalados, este equipo se usa para:
- **IA generativa local:** Ollama (LLMs locales) y Open WebUI como interfaz de chat
- **Etiquetado de datos:** Label Studio (contenedor Docker)
- **Robótica:** ROS Melodic con cámara USB (contenedor `yahboomtechnology/ros-melodic:usb_cam`), bus CAN activo
- **Visión/IA embebida NVIDIA:** DeepStream (usuario de sistema `deepstream` presente), Argus (cámaras CSI)
- **Monitorización:** Netdata, jtop/jetson_stats
- **Desarrollo:** JupyterLab activo como servicio
