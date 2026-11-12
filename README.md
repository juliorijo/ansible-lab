# Laboratorio: Docker y Ansible

## Objetivo

Crear dos servidores Linux con Ubuntu 24.04 en contenedores Docker y
administrarlos con Ansible desde Linux o Ubuntu en WSL. El laboratorio muestra
el nombre y sistema operativo de cada servidor, crea `/tmp/ansible-demo`,
escribe `info.txt`, crea `logs`, `backup` y `config` mediante un loop y muestra
un mensaje condicionado a que el sistema operativo sea Ubuntu.

Los contenedores representan servidores de laboratorio: no son maquinas
virtuales, comparten el kernel del motor Docker y no necesitan SSH. Ansible
usa la conexion Docker local.

## Archivos

| Archivo | Funcion |
| --- | --- |
| `docker-compose.yml` | Define `server1` y `server2` con `ubuntu:24.04` y `sleep infinity`. |
| `inventory.ini` | Agrupa ambos servidores en `[docker]` con `ansible_connection=docker`. |
| `playbook.yml` | Recopila facts y configura ambos servidores. |
| `README.md` | Explica la preparacion, ejecucion y verificacion. |

## Requisitos

- Docker Engine con Docker Compose v2.
- En Windows: Docker Desktop iniciado con contenedores Linux, Ubuntu en WSL
  y la integracion de Docker Desktop habilitada para esa distribucion.
- Acceso a Internet para descargar Ubuntu e instalar Python y Ansible.

Los siguientes comandos se ejecutan en una terminal **Linux o Ubuntu en WSL**,
no en PowerShell. En Windows, abrir Ubuntu desde el menu Inicio.

### 1. Preparar Ansible en el equipo de control

```bash
sudo apt-get update
sudo apt-get install -y git python3 python3-venv
python3 -m venv ~/.venvs/ansible-lab
source ~/.venvs/ansible-lab/bin/activate
python -m pip install ansible-core
ansible-galaxy collection install community.docker

git clone https://github.com/juliorijo/ansible-lab.git
cd ansible-lab
docker info
docker compose version
```

Si ya se clono el repositorio, entrar en esa carpeta en lugar de clonarlo otra
vez. En una terminal nueva, reactivar el entorno con
`source ~/.venvs/ansible-lab/bin/activate`.

Si `docker info` falla, iniciar Docker Desktop y revisar su integracion con
WSL antes de continuar.

### 2. Revisar conflictos e iniciar los contenedores

```bash
docker ps -a --filter 'name=^/server1$' --filter 'name=^/server2$' \
  --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Si aparecen contenedores de otro proyecto llamados `server1` o `server2`,
detenerse: no eliminarlos ni modificarlos. Los nombres requeridos deben estar
libres, o corresponder a una ejecucion anterior de este mismo laboratorio.

```bash
docker compose up -d
docker compose ps
```

### 3. Instalar Python en cada contenedor

La imagen base no incluye el interprete que necesitan los modulos de Ansible.
Los contenedores ejecutan como root, por lo que estos comandos no usan sudo.

```bash
docker exec -e DEBIAN_FRONTEND=noninteractive server1 bash -c 'apt-get update && apt-get install -y python3'
docker exec -e DEBIAN_FRONTEND=noninteractive server2 bash -c 'apt-get update && apt-get install -y python3'
```

### 4. Comprobar la conexion y ejecutar el playbook

```bash
ansible-inventory -i inventory.ini --graph
ansible -i inventory.ini docker -m ansible.builtin.ping
ansible-playbook -i inventory.ini playbook.yml --syntax-check
ansible-playbook -i inventory.ini playbook.yml
```

El modulo `ping` comprueba la conexion y Python, no utiliza ICMP. Ambos
servidores deben devolver `pong`. En el resumen del playbook, cada servidor
debe mostrar `unreachable=0` y `failed=0`.

### 5. Verificar los resultados y la idempotencia

```bash
docker exec server1 cat /tmp/ansible-demo/info.txt
docker exec server2 cat /tmp/ansible-demo/info.txt
docker exec server1 ls -l /tmp/ansible-demo
docker exec server2 ls -l /tmp/ansible-demo
ansible-playbook -i inventory.ini playbook.yml
```

En cada contenedor deben existir `info.txt`, `logs`, `backup` y `config`.
Una segunda ejecucion sin cambios externos debe mostrar `changed=0`,
`unreachable=0` y `failed=0` para ambos servidores.

### 6. Detener el laboratorio

Desde esta misma carpeta:

```bash
docker compose down
```

Este comando elimina los contenedores de este laboratorio y sus archivos
internos, incluidos los resultados en `/tmp/ansible-demo`. No usar comandos
globales como `docker system prune`.

## Evidencia de ejecucion

Ejecucion real verificada el **12 de noviembre de 2026**, desde Ubuntu en WSL,
con Docker Engine 29.8.2, ansible-core 2.22.0 y community.docker 5.4.0.
El inventario detecto ambos servidores, `ping` devolvio `pong` para ambos y
la comprobacion de sintaxis termino sin errores.

Resumen real de la primera ejecucion:

```text
server1 : ok=6 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
server2 : ok=6 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Resumen real de la segunda ejecucion, que confirma idempotencia:

```text
server1 : ok=6 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
server2 : ok=6 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Tambien se comprobo directamente en ambos contenedores que `info.txt` contiene
su nombre, hostname y `Ubuntu 24.04`, y que existen los tres subdirectorios.
Ansible emitio solamente avisos informativos de descubrimiento automatico
del interprete Python (`/usr/bin/python3.12`), sin fallos de ejecucion.

**Captura de pantalla pendiente:** la politica de seguridad de las herramientas
de interfaz bloqueo el acceso a Terminal y Copilot para obtener la imagen.
Los resumenes anteriores son resultados reales, pero no sustituyen la captura
solicitada. No se ha fabricado ni simulado una imagen.

Para completar esa parte de la entrega, ejecutar
`ansible-playbook -i inventory.ini playbook.yml` en una terminal, tomar una
captura autentica que muestre las tareas y el `PLAY RECAP` de ambos servidores,
agregarla al repositorio y enlazarla en esta seccion.
