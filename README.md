Ejemplos Podman
====

## Consideraciones:

- OS: RedHat Enterprise Linux 8.x  / Centos 8.x
- FileSystem: Ext4
- Mem: 8Gb (recomendable)
- CPU: 2vcpu (recomendable)
- HD: 50 Gb (recomendable)
- Selinux: ENFORCING

Este repositorio aloja ejemplos de podman usando rootless.

Vamos a configurar lo siguiente:

> dnf install slirp4netns podman podman-plugins podman-docker -y 

**slirp4netns** se conectara a  network namespace y podremos exponer puertos fuera de la red completamente en rootless (sin privilegios)

Caso contrario podemos ejecutar esto:

> net.ipv4.ip_unprivileged_port_start=0 lo agregamos en */etc/sysctl.conf*. 

Ejecutamos:

> sysctl -p && systcl -a

Vamos a aumentar el numero de namespace:

> echo "user.max_user_namespaces=28633" > /etc/sysctl.d/userns.conf 	 

> sysctl -p /etc/sysctl.d/userns.conf

Esta accion habilita el uso de espacios de nombres/namespace de red sin root.

Ahora si, creamos un usuario:

> useradd -c "podman admin" podman 

> passwd podman

Ejecutamos:

> loginctl enable-linger podman

> export XDG_RUNTIME_DIR=/run/user/$(id -u)


Iniciamos de la siguiente forma el servicio:

> systemctl enable --now podman.socket

Listamos el socket:

> /var/run/podman/podman.sock

Ejecutamos:

> export DOCKER_HOST=unix:///var/run/podman/podman.sock


## Una Recomendacion 

Bajo este entorno estamos usando podman con rootless, por ende deberiamos "aislar" el socket para el dueño del servicio sea el owner,
por ello podemos realizar esta configuracion (nos permitira hacer un artificio al famoso *docker.sock*):

> su - podman 

Ejecutamos:

> loginctl enable-linger podman

> export XDG_RUNTIME_DIR=/run/user/$(id -u)


```
vim .config/systemd/user/podman.socket

[Unit]
Description=Podman API Socket
Documentation=man:podman-api(1)
 
[Socket]
ListenStream=/home/podman/docker.sock
SocketMode=0660
 
[Install]
WantedBy=sockets.target
```

Creamos el servicio rootless para podman:

```
vim .config/systemd/user/podman.service

[Unit]
Description=Podman API Service
Requires=podman.socket
After=podman.socket
Documentation=man:podman-api(1)
StartLimitIntervalSec=0
 
[Service]
Type=oneshot
Environment=REGISTRIES_CONFIG_PATH=/etc/containers/registries.conf
ExecStart=/usr/bin/podman system service unix:///home/podman/docker.sock
TimeoutStopSec=30
KillMode=process
 
[Install]
WantedBy=multi-user.target
Also=podman.socket
```

Reiniciamos y recargamos:

> systemctl --user daemon-reload

> systemctl --user enable --now podman.socket

Configuramos el socket via rootless:

> export DOCKER_HOST=unix:///home/podman/docker.sock

## Instalamos podman-compose

**Metodo 1:**
```
pip3 install https://github.com/containers/podman-compose/archive/devel.tar.gz
```

**Metodo 2:**
```
curl -o /usr/local/bin/podman-compose https://raw.githubusercontent.com/containers/podman-compose/devel/podman_compose.py
chmod +x /usr/local/bin/podman-compose
```
