---

# Guia d’Integració Service Next → Salesly

Aquesta guia descriu el procés general per implementar l’autenticació OAuth de Salesly a la vostra aplicació, mitjançant la integració en forma d’**iframe** de Salesly.

## Prerequisits

Assegureu-vos de disposar dels valors següents, proporcionats per l’equip de Salesly:

* **CLIENT\_ID**: ` `
* **CLIENT\_SECRET**: ` `
* **REDIRECT\_URI**: `https://exampe.com/callback`

> *Cal enviar la URL utilitzada a [joan.guitart@salesly.app](mailto:joan.guitart@salesly.app).*

---

## Flux d’Autenticació Recomanat

1. L’usuari fa clic a “Inicia sessió amb Salesly”.
2. És redirigit a l’endpoint d’autorització de Salesly.
3. L’usuari concedeix els permisos sol·licitats.
4. Salesly redirigeix a la vostra aplicació amb un **codi d’autorització**.
5. El vostre backend intercanvia aquest codi per un **token d’accés**.
6. El token s’utilitza per autenticar l’usuari dins d’un **iframe** de Salesly.

---

## Resolució d’Incidències

* Comproveu que el `REDIRECT_URI` coincideixi exactament amb el registrat a Salesly.
* Verifiqueu que el `CLIENT_ID` i el `CLIENT_SECRET` siguin correctes.
* Assegureu-vos que s’hagin sol·licitat i concedit els **scopes** necessaris (actualment només `login`).

---

## Exemple d’Implementació

### 1. Redirigir l’Usuari a l’Endpoint d’Autorització

```javascript
const authEndpoint = 'https://api.salesly.app/oauth/authorize';
const clientId = CLIENT_ID;
const redirectUri = encodeURIComponent(REDIRECT_URI);
const scope = 'login';
const state = generarRandomString();

localStorage.setItem('oauth_state', state); // Protecció CSRF

const authUrl = `${authEndpoint}?response_type=code&client_id=${clientId}&redirect_uri=${redirectUri}&scope=${scope}&state=${state}`;
window.location.href = authUrl;
```

---

### 2. Gestionar el Callback OAuth

```javascript
const params = new URLSearchParams(window.location.search);
const code = params.get('code');
const state = params.get('state');

const savedState = sessionStorage.getItem('oauth_state');
if (state !== savedState) {
  // Error de verificació CSRF
  return;
}

if (code) {
  intercambiarCodigoPorToken(code);
}
```

---

### 3. Intercanviar el Codi per un Token (Backend amb Node.js/Express)

```javascript
app.post('/api/auth/token', async (req, res) => {
  try {
    const tokenEndpoint = 'https://api.salesly.app/oauth/token';
    const params = new URLSearchParams();
    params.append('grant_type', 'authorization_code');
    params.append('code', CODE);
    params.append('redirect_uri', REDIRECT_URI);
    params.append('client_id', CLIENT_ID);
    params.append('client_secret', CLIENT_SECRET);

    const response = await fetch(tokenEndpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: params,
    });

    const tokens = await response.json();
    console.log(tokens);

    // Desa els tokens associats a l’usuari
    res.json({ success: true });
  } catch (error) {
    console.error('Error en intercanviar el token:', error);
    res.status(500).json({ error: 'Autenticació fallida' });
  }
});
```

---

### 4. Iniciar Sessió amb el `access_token`

```javascript
const iframeSrc = `https://crm.salesly.app/oauth-login?access_token=${ACCESS_TOKEN}`;

<iframe
  src={iframeSrc}
  title="Salesly CRM"
  sandbox="allow-same-origin allow-scripts allow-popups allow-forms allow-storage-access-by-user-activation allow-top-navigation"
  referrerPolicy="no-referrer-when-downgrade"
  allowFullScreen
/>
```

---

### 5. Refrescar el Token d’Accés

```javascript
app.post('/api/auth/token', async (req, res) => {
  try {
    const tokenEndpoint = 'https://api.salesly.app/oauth/token';
    const params = new URLSearchParams();
    params.append('grant_type', 'refresh_token');
    params.append('refresh_token', REFRESH_TOKEN);
    params.append('client_id', CLIENT_ID);
    params.append('client_secret', CLIENT_SECRET);

    const response = await fetch(tokenEndpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: params,
    });

    const tokens = await response.json();
    console.log(tokens);

    // Desa els nous tokens
    res.json({ success: true });
  } catch (error) {
    console.error('Error en refrescar el token:', error);
    res.status(500).json({ error: 'Autenticació fallida' });
  }
});
```

---

## Resum del Flux Complet

**Frontend**

* Redirigeix l’usuari a l’endpoint d’autorització de Salesly.
* Gestiona el `callback` i recupera el codi d’autorització.
* Envia aquest codi al backend.

**Backend**

* Intercanvia el codi per un token d’accés i, si cal, un token de refresc.
* Desa els tokens de forma segura.
* Gestiona l’expiració i la renovació automàtica dels tokens.

---
