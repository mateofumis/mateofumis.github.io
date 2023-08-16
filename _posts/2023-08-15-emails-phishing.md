---
title:  "Correos Phishing: ¿Cómo detectar los correos que son auténticos de los que no? Te explico brevemente en este artículo con un ejemplo."
date: 2023-08-15
mathjax: true
layout: post
categories: articulo
tags: red-teaming ingenieria-social phishing
---

![portada](https://i.ibb.co/J2CYrtS/portada.png)

En la actualidad todos nos comunicamos por correo electrónico (email), tanto en nuestro trabajo como para registrarnos en sitios web, para iniciar sesión en cuentas bancarias digitales y un sin fin de usos más. Pero la verdad es que, para los ciberdelincuentes, éste se convierte en un medio por el cual hacen uso de la Ingeniería Social con el fin de realizar estafas, robo de credenciales, tomar el control de nuestro ordenador o teléfono e incluso llegar a obtener acceso a la red corporativa entera de una empresa.*Y todo esto se ocasiona por tan sólo **una falla o accidente humano.***

### Recientemente he recibido un correo que me sirve de mucha utilidad para usarlo como ejemplo para explicar cómo podemos darnos cuenta de si estamos ante un intento de phishing bien planeado, o si bien es un correo auténtico.

![email-recibido](https://i.ibb.co/vcCgrWb/email-recibido.png)

*La imagen de arriba muestra el correo que recibí desde mi teléfono móvil.*

#### Este presunto correo aparenta ser enviado por la empresa **TechSmith**, la cual es conocida por su software para crear grabaciónes de pantalla: "Camtasia Studio".

![camtasia-techsmith](https://i.ibb.co/yqDtDpS/camtasia-techsmith.png)

### Ahora bien, lo primero que debemos hacer al recibir un correo es fijarnos en su origen (es decir, **el remitente**).

![remitente](https://i.ibb.co/qDwyzcy/remitente.png)

#### Como podemos observar en la imagen de arriba, el remitente es **`email@techsmith.messages4.com`**, y en este caso éste remitente utiliza el dominio `messages4.com`. Con una simple Búsqueda en Google o DuckDuckgo podemos saber cuál es el verdadero dominio de TechSmith.

![google-busqueda](https://i.ibb.co/1KsnM40/google-busqueda.png)

### Con esto en mente sabemos que, el creador de este correo está utilizando el **subdominio** `techsmith` para aparentar ser auténtico.

#### *Entonces para identificar un dominio legítimo, ¿cómo se separa el dominio principal de los subdominios que pueda tener?*

#### El dominio se establece por las primeras dos partes separadas por un punto desde la derecha hacia la izquierda. Es decir que en este caso, el dominio es `messages4.com` y `techshmith` es el primer subdominio.

```yaml
- Dirección: "techsmith.messages4.com"
  - Dominio: "messages4.com"
  - Subdominio: "techsmith"
```

#### Ejemplos: `subdomain1.subdomain2.subdomain3.example.com` donde `example.com` es el **dominio** y el resto son los **subdominios**. Cabe destacar que `com` en este caso es el TLD (top-level domain). Ejemplos más comunes de TLD son: `com`,`org`,`net`,`gov`, etc...

*Más información sobre los TLD: [https://www.cloudflare.com/learning/dns/top-level-domain/](https://www.cloudflare.com/learning/dns/top-level-domain/){:target="_blank"}*

### En este punto podemos identificar que el origen no es auténtico de la empresa TechSmith... pero entonces, ¿debemos fijarnos en el remitente de todos los correos que recibimos?🤔 La respuesta es: depende.

Por lo general, para identificar los correos de phishing éstos suelen tener fallas ortográficas, generar sensación de urgencia, hacerse pasar por una figura de autoridad como lo podría ser alguien de nuestra empresa, o incluso por personas que ya conocemos.

#### Lo que intentan es generar **"confianza, sensación de urgencia, intimidación, o interés por algo"**. 

![phishing](https://i.ibb.co/WsxzjTs/phishing.png)

#### Con eso se hace más facil creer que el correo es *auténtico*, cuando en realidad es un intento de phishing.

### Como conclusión, siempre tenemos que ser astutos y perceptivos frente a los correos que recibimos, especialmente cuando parecen ser *"demasiado buenos para ser verdad"*.

----

## Author: Mateo Fumis 
