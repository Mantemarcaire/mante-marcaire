# Netlify CMS para mantenimiento.marcaire.com

Este paquete agrega un panel de edición web en **/admin** para que puedas cambiar ajustes del formulario (correo remitente, botones de GPS, firma con dedo, adjuntos PDF/PNG, etc.) y los textos del formulario.

## 1) Archivos a subir (mantener estructura)
```
/admin/index.html
/admin/config.yml
/data/settings.json
/data/form_texts.json
```

## 2) Conectar el sitio a un repositorio (requerido por Netlify CMS)
- En Netlify, conecta el sitio a **GitHub / GitLab / Bitbucket** (Deploys → Connect to Git).
- Commit/push estos archivos al repositorio y desplegá de nuevo.

## 3) Activar autenticación
- En Netlify: **Site settings → Identity → Enable Identity**.
- En Identity → Settings, dejá activado **Git Gateway**.
- Invitá tu correo como usuario (Identity → Invite users).

## 4) Entrar al panel
- Abrí: `https://mantenimiento.marcaire.com/admin`
- Iniciá sesión con Netlify Identity.

## 5) Cómo se usan los ajustes en el sitio
- `data/settings.json` y `data/form_texts.json` se deben **leer desde tu `index.html`**.
- Agregá un script a tu HTML que cargue esos JSON y aplique las opciones (ejemplo abajo).
- Con esto, cada cambio desde el panel actualiza los JSON en el repo y **el sitio se república automáticamente**.

### Ejemplo de carga (inserta en tu HTML antes de tu script principal)
```html
<script>
async function cargarAjustes() {
  const [settingsRes, textsRes] = await Promise.all([
    fetch('/data/settings.json', {cache:'no-store'}),
    fetch('/data/form_texts.json', {cache:'no-store'}),
  ]);
  const settings = await settingsRes.json();
  const texts = await textsRes.json();

  // 1) Correo remitente
  window.MARCAIRE = window.MARCAIRE || {};
  window.MARCAIRE.emailFrom = settings.from_email || 'info@marcaire.com';
  window.MARCAIRE.emailToDefault = settings.to_email_default || '';

  // 2) Hora editable / firma
  window.MARCAIRE.allowTimeEdit = !!settings.allow_time_edit;
  window.MARCAIRE.signatureEnabled = !!settings.signature_enabled;

  // 3) Modo GPS (auto/google/apple/both)
  window.MARCAIRE.gpsMode = settings.gps_button_mode || 'auto';

  // 4) Adjuntos PDF/PNG
  window.MARCAIRE.attachPDF = !!(settings.attachments && settings.attachments.pdf);
  window.MARCAIRE.attachPNG = !!(settings.attachments && settings.attachments.png);

  // 5) Textos UI
  document.querySelector('[data-i18n="title"]')?.replaceChildren(document.createTextNode(texts.title || ''));
  document.querySelector('[data-i18n="description"]')?.replaceChildren(document.createTextNode(texts.description || ''));
  document.querySelector('[data-i18n="footer"]')?.replaceChildren(document.createTextNode(texts.footer || ''));

  // 6) Etiquetas de botones (si existen)
  const gpsBtn = document.querySelector('[data-role="gps"]');
  if (gpsBtn && settings.ui_texts?.gps_label) gpsBtn.textContent = settings.ui_texts.gps_label;

  const signBtn = document.querySelector('[data-role="sign"]');
  if (signBtn && settings.ui_texts?.sign_label) signBtn.textContent = settings.ui_texts.sign_label;

  const sendBtn = document.querySelector('[data-role="send"]');
  if (sendBtn && settings.ui_texts?.send_label) sendBtn.textContent = settings.ui_texts.send_label;

  // 7) Tema UI
  document.documentElement.dataset.theme = settings.advanced?.theme || 'light';
}
cargarAjustes().catch(console.error);
</script>
```

> Nota: El envío **desde `info@marcaire.com`** depende del servicio de correo (Netlify Functions + proveedor SMTP/SendGrid/Postmark). Este CMS te permite cambiar el remitente en `data/settings.json`. Tu función de envío debe leer ese valor y usarlo como `from`.

## 6) Botones de GPS compatibles iOS/Android (lógica sugerida)
- Auto-detectar plataforma y abrir Apple Maps en iOS o Google Maps en Android.
- Si `gpsMode` es "both", mostrar dos botones.

Ejemplo breve:
```js
function abrirMaps(lat, lng){
  const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);
  const apple = `http://maps.apple.com/?ll=${lat},${lng}`;
  const google = `https://www.google.com/maps?q=${lat},${lng}`;
  const mode = window.MARCAIRE?.gpsMode || 'auto';
  if (mode === 'apple') return window.location.href = apple;
  if (mode === 'google') return window.location.href = google;
  if (mode === 'both') { /* mostrar dos botones */ return; }
  // auto
  window.location.href = isIOS ? apple : google;
}
```

## 7) Firma con el dedo
- Usá una librería de canvas (por ej. `signature_pad`) y habilitala solo si `signatureEnabled` es verdadero.
- Guardá el PNG y/o incorpóralo al PDF si `attachPNG` o `attachPDF` está activo.

---

Cualquier duda, te ayudo a integrar el script con tu `index.html`.