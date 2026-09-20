# Guía de Backup para tu Jetson (antes de actualizar JetPack)

## ¿Para qué sirve esta guía?

Cuando actualizas la versión de JetPack de una Jetson, normalmente el proceso borra todo el contenido del disco (interno, SD o NVMe, según el modelo). Esto significa que si no guardas tus archivos, programas y configuraciones en otro lugar antes de actualizar, los pierdes para siempre.

Esta guía explica, paso a paso y en lenguaje simple, cómo hacer una copia de seguridad completa de tu Jetson y sacarla del dispositivo, para que puedas actualizar tranquilo sin perder nada.

No necesitas ser experto en Linux para seguir esta guía. Cada paso explica qué hace el comando y por qué se hace, antes de mostrar el comando en sí.

**Requisito:** tener acceso a una terminal de tu Jetson (una ventana donde se escriben comandos) y una forma de sacar los archivos de ahí (un USB, un disco externo, u otra computadora en la misma red).

> Esta guía cubre solo la parte del backup. La actualización de JetPack en sí va a estar en otro documento aparte, más adelante.

---

## Paso 1: Abrir una terminal

Abre una ventana de terminal en tu Jetson. Vas a ver algo como esto, esperando que escribas un comando:

```bash
usuario@jetson:~$
```

El símbolo `$` indica que la terminal está lista para recibir un comando. Todos los comandos de esta guía se escriben ahí y se ejecutan presionando **Enter**.

## Paso 2: Crear una carpeta para guardar los backups

Vamos a crear una carpeta especial en el Escritorio, donde va a vivir todo lo que respaldemos, para tenerlo todo ordenado y fácil de encontrar.

```bash
mkdir -p ~/Desktop/backups
cd ~/Desktop/backups
```

`mkdir` significa "make directory" (crear carpeta). `~/Desktop/backups` le dice a Linux que cree una carpeta llamada `backups` dentro de tu Escritorio.

`cd` significa "change directory" (entrar a esa carpeta). Después de este comando, todo lo que hagas va a guardarse ahí dentro, salvo que se indique lo contrario.

Puedes confirmar que estás dentro escribiendo:
```bash
pwd
```
Esto imprime en pantalla la carpeta en la que estás parado, debería terminar en `Desktop/backups`.

## Paso 3: Guardar la lista de programas instalados

Antes de copiar archivos grandes, guardamos algo simple: la lista de todo lo que tienes instalado. Esto sirve para reinstalar rápido todo después, sin tener que acordarte de memoria qué programas usabas.

```bash
dpkg --get-selections > paquetes_apt.list
apt-mark showmanual > paquetes_manuales.list
pip3 freeze > paquetes_pip.txt
docker images --format "{{.Repository}}:{{.Tag}}" > docker_imagenes.txt 2>/dev/null
crontab -l > crontab_backup.txt 2>/dev/null
```

Qué hace cada línea (todas hacen lo mismo: guardar una lista en un archivo de texto):

- `paquetes_apt.list`: todos los programas instalados con el instalador del sistema (`apt`).
- `paquetes_manuales.list`: los que instalaste tú a propósito, sin contar los que vinieron con el sistema.
- `paquetes_pip.txt`: librerías de Python instaladas.
- `docker_imagenes.txt`: si usas Docker, qué imágenes tienes descargadas (ver qué es Docker más abajo, en el Paso 6).
- `crontab_backup.txt`: tareas programadas automáticas, si tienes alguna.

Si alguno de estos comandos no aplica a tu caso, por ejemplo no usas Docker, el archivo puede quedar vacío. No es un error, simplemente no tenías nada que guardar ahí.

## Paso 4: Hacer una copia de todos tus archivos personales

Este es el paso más importante: empaquetar toda tu carpeta personal (documentos, proyectos, código, fotos, todo) en un solo archivo, fácil de mover a otro lado.

```bash
sudo tar --acls --xattrs -cvpf /home/backup_$(date +%F).tar -C / home/$(whoami)
sudo chown $(whoami):$(whoami) /home/backup_*.tar
sudo mv /home/backup_*.tar ~/Desktop/backups/
```

En palabras simples: `tar` junta muchos archivos y carpetas en un solo archivo grande, como armar una maleta con todas tus cosas adentro, sin dejar nada afuera.

Se guarda primero en `/home/`, un nivel arriba de tu carpeta, y no directamente dentro de `~/Desktop/backups`, porque si no sería como intentar meter la maleta dentro de sí misma mientras la armas. Después lo movemos a `~/Desktop/backups` para tener todo junto.

`sudo` significa "hacé esto con permisos de administrador". Se necesita porque algunos archivos tuyos pueden tener permisos especiales que un usuario normal no puede leer sin ayuda.

Este paso puede tardar varios minutos si tienes muchos archivos. Es normal, espera a que la terminal vuelva a mostrar el `$` antes de seguir.

## Paso 5: Comprobar que la copia salió bien

Antes de confiar en el backup, lo revisamos. Comparamos cuántos archivos hay adentro del backup contra cuántos archivos hay realmente en tu carpeta:

```bash
tar -tvf ~/Desktop/backups/backup_*.tar | wc -l
find /home/$(whoami) | wc -l
```

Los dos números deberían ser casi idénticos (una diferencia de 1 o 2 es normal y no indica ningún problema). Si son muy distintos, algo salió mal y hay que repetir el Paso 4 antes de continuar.

## Paso 6: Si usas Docker, revisar y guardar tus imágenes

¿Qué es Docker, rápidamente? Es un programa que permite correr aplicaciones "empaquetadas" en cajas independientes llamadas contenedores, sin instalar todo directamente en tu sistema. Cada imagen es como el molde de esa caja, y cada vez que la corres, se crea un contenedor a partir de ese molde. Si nunca instalaste ni usaste Docker, puedes saltar este paso y el siguiente.

Revisa qué tienes:
```bash
cat ~/Desktop/backups/docker_imagenes.txt
docker images
```

Guarda cada imagen que quieras conservar en un archivo propio:
```bash
docker save -o ~/Desktop/backups/nombre_que_le_quieras_poner.tar nombre_de_la_imagen:version
```

## Paso 7: Si usas Docker, guardar los datos generados (volúmenes)

Esto es distinto a las imágenes: la imagen es el molde, pero los volúmenes son los datos que generaste usándolo, por ejemplo tus proyectos guardados dentro de una app. Esto sí se pierde para siempre si no lo respaldas, aunque vuelvas a descargar la imagen después.

Ver qué volúmenes existen:
```bash
docker volume ls
```

Guardar el contenido de un volumen específico:
```bash
docker run --rm -v NOMBRE_DEL_VOLUMEN:/data -v ~/Desktop/backups:/backup alpine \
  tar -czf /backup/NOMBRE_DEL_VOLUMEN.tar.gz -C /data .
sudo chown $(whoami):$(whoami) ~/Desktop/backups/NOMBRE_DEL_VOLUMEN.tar.gz
```
Reemplaza `NOMBRE_DEL_VOLUMEN` por el nombre real que viste en el paso anterior.

## Paso 8: Revisar que ningún archivo se haya dañado

Un archivo dañado (corrupto) puede fallar al querer abrirlo después, cuando ya sea demasiado tarde. Lo comprobamos ahora, mientras todavía se puede repetir el paso si algo falló:

```bash
cd ~/Desktop/backups
tar -tf backup_*.tar > /dev/null && echo "OK: backup principal"
```

Si ves `OK: backup principal`, ese archivo está sano. Repite lo mismo con cualquier otro `.tar` o `.tar.gz` que hayas creado, usando `tar -tzf` en vez de `tar -tf` para los que terminan en `.gz`.

Como verificación extra, se genera una huella digital única de cada archivo (checksum). Sirve para comprobar, después de mover los archivos a otro lugar, que llegaron exactamente iguales y no se dañaron en el camino:

```bash
sha256sum * > checksums.sha256
```

## Paso 9: Sacar el backup de la Jetson

Este es el paso que realmente protege tu información. Mientras el backup siga guardado dentro de la Jetson, se pierde igual que todo lo demás al actualizar. Hay dos opciones para sacarlo: copiarlo a un USB o disco externo, o enviarlo por red a otra computadora. Elige la que te resulte más práctica, o haz ambas para mayor seguridad.

### Opción A: Copiar a un USB o disco externo

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
```
Este comando muestra todos los discos conectados. Busca el que corresponde a tu USB, generalmente es fácil de identificar por el tamaño, y anota su nombre exacto. Asegúrate de no confundirlo con el disco interno de la Jetson.

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/NOMBRE_DE_TU_USB /mnt/usb
cp -r ~/Desktop/backups /mnt/usb/
sync
sudo umount /mnt/usb
```
Recién cuando termine el `umount` sin errores es seguro desconectar el USB físicamente.

### Opción B: Copiar por red a otra computadora (usando Tailscale)

Tailscale es un programa que conecta tus dispositivos entre sí de forma privada, como si estuvieran en la misma red local aunque estén en lugares distintos. Esto permite copiar archivos de una máquina a otra usando la dirección que Tailscale le asigna a cada equipo, sin exponerlo a internet público.

Primero confirma que la Jetson está conectada a tu red de Tailscale:
```bash
tailscale status
```
Ahí vas a ver la lista de tus dispositivos junto con su dirección de Tailscale (algo como `100.x.x.x` o un nombre terminado en `.ts.net`).

Luego copia el backup con `rsync`, reemplazando `usuario` por el usuario de la otra máquina, y `DIRECCION_TAILSCALE` por la dirección que viste en el comando anterior:
```bash
rsync -avP --info=progress2 ~/Desktop/backups/ usuario@DIRECCION_TAILSCALE:/ruta/destino/
```

`usuario@DIRECCION_TAILSCALE` es donde va la dirección de Tailscale del equipo destino. `/ruta/destino/` es la carpeta de esa otra máquina donde quieres que se guarde la copia. La opción `-P` permite retomar la transferencia si se corta a mitad de camino, en vez de tener que empezar de cero.

## Paso 10: Confirmar que todo llegó bien al destino final

Ya en el destino (USB o la otra computadora), verifica que los archivos coincidan exactamente con los originales usando la huella digital que generaste en el Paso 8:

```bash
sha256sum -c checksums.sha256
```

Si cada línea dice `OK`, el backup está completo, íntegro, y seguro fuera de la Jetson. Recién en este punto es seguro proceder a actualizar JetPack.

---

## Precauciones importantes

- Nunca uses comandos que borren o reparen discos (`dd`, `mkfs`, herramientas de "reparar partición") sin antes revisar con calma qué hay en ese disco. Algunos comandos son irreversibles.
- Antes de conectar o usar cualquier USB o disco externo, usa `lsblk` para identificar bien cuál es cuál, sobre todo si tu Jetson tiene más de un disco conectado.
- Si un disco externo muestra errores de "tabla de particiones corrupta" y tiene datos importantes, no lo formatees de inmediato. Existen herramientas de recuperación, como `testdisk`, pensadas justamente para estos casos.
- El backup no cuenta como seguro hasta que existe una copia fuera del dispositivo que vas a actualizar.

## Siguiente paso: guía de actualización (documento aparte)

Este documento termina acá: con el backup completo, verificado y ya sacado de la Jetson. La actualización de JetPack en sí no se explica en esta guía, va a ser un documento nuevo y separado, específico para ese proceso.

Una vez actualizado el sistema, estos son los archivos del backup que se van a necesitar:

- `paquetes_apt.list`, `paquetes_manuales.list`, `paquetes_pip.txt`: para reinstalar tus programas.
- `backup_*.tar`: tus archivos personales, se restauran con `tar -xpvf backup_*.tar -C /`.
- Los `.tar`/`.tar.gz` de Docker: se restauran con `docker load -i archivo.tar` (imágenes) o extrayendo el `.tar.gz` dentro del volumen correspondiente.
