# Tu MikroTik podría estar atacando a otros: caso de reflexión DNS

[English version](README.md)

## Resumen

Analizando datos de NetFlow en un proveedor de internet, encontré que cientos de routers de clientes se estaban usando como **reflectores DNS** en ataques de amplificación contra terceros. Yo encontré el problema y coordiné la respuesta.

| Métrica | Valor |
|---|---|
| Routers de clientes usados como reflectores | 232 |
| Porcentaje que eran MikroTik | **82 %** |
| Víctimas atacadas en 24 horas | **más de 1.400** (sobre todo servidores de juegos) |
| Amplificación | ~20x |
| Pico de tráfico de ataque | ~1 Gbps |

## Qué pasó

- Los routers de los clientes tenían **"allow remote requests" activado** en el DNS y el **puerto 53 (UDP/TCP) abierto en la WAN**, lo que los convertía en resolvers abiertos.
- Los atacantes enviaban consultas DNS pequeñas con la **IP de la víctima falsificada** como origen. Cada router le respondía a la víctima con un paquete unas 20 veces más grande.
- Unos 20 Mbps de consultas se convertían en cientos de Mbps de ataque, con picos cercanos a 1 Gbps.
- El tráfico era visible en NetFlow desde el primer día de recolección, dos semanas antes de detectarlo formalmente.
- **Ningún cliente había reportado un problema.** Su internet "funcionaba bien", mientras su canal de subida se usaba para atacar a otros.

## Cómo se encontró

1. NetFlow mostró tráfico UDP/53 saliente y constante desde direcciones de clientes.
2. Al correlacionar los flujos con el inventario (Grafana + NetBox), cada origen quedó ligado a un servicio y cada destino a una víctima y un horario.
3. Un script generó un informe por cada incidente.
4. Al agrupar por fabricante, el 82 % de los reflectores eran MikroTik con el mismo error de configuración.

## Por qué bloquear a la víctima no alcanza

Enviar el destino a un blackhole (RTBH) frena un ataque, pero el atacante pasa a otra víctima usando los mismos reflectores. La solución de fondo está en el origen: cerrar los resolvers abiertos.

**Lección para automatizar:** antes de automatizar RTBH, define una lista blanca y una expiración automática para cada bloqueo. Si no, terminas con una lista de prefijos bloqueados que nadie revisa.

## Cómo corregirlo (MikroTik)

Desactiva las consultas remotas si el router no necesita darle DNS a otros equipos:

```
/ip dns set allow-remote-requests=no
```

Si debe darle DNS a la LAN, bloquea el DNS desde la WAN:

```
/ip firewall filter
add chain=input protocol=udp dst-port=53 in-interface-list=WAN action=drop comment="Bloquear DNS desde WAN"
add chain=input protocol=tcp dst-port=53 in-interface-list=WAN action=drop comment="Bloquear DNS desde WAN"
```

Además, mantén RouterOS actualizado y no expongas Winbox, la API ni la interfaz web a internet. En 2021 la botnet **Meris** tomó el control de unos 250.000 MikroTik expuestos usando una falla antigua de Winbox.

## Desafío pendiente

La corrección técnica es simple. Lo difícil es la **comunicación con los clientes**: la mayoría no sabe que su router participa, y actuar sobre su servicio requiere una política clara.

## Notas

Todas las gráficas y los datos están anonimizados. No se incluyen direcciones IP, configuraciones de la red del proveedor, topología ni datos de clientes.
