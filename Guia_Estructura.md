🔐🔥 PROYECTO COMPLETO: “React + Node.js + Firebase Auth & Firestore”  
(Estructura real, código comentado línea a línea, paso a paso)

----------------------------------------
📂 ESTRUCTURA FINAL
----------------------------------------

mi-proyecto/
├─ cliente/               ← React (solo diseño y llamadas)
│  ├─ public/
│  │  └─ index.html
│  ├─ src/
│  │  ├─ index.js
│  │  ├─ App.js
│  │  ├─ firebase.js      ← conexión cliente (para auth)
│  │  ├─ api.js           ← llamadas al servidor
│  │  └─ styles.css
│  └─ .env.example
├─ servidor/              ← Node + Express + Firebase Admin
│  ├─ controllers/
│  │  └─ usuariosController.js
│  ├─ models/
│  │  └─ usuarioModel.js
│  ├─ routes/
│  │  └─ usuarios.js
│  ├─ serviceAccountKey.json  ← TU archivo JSON
│  ├─ index.js
│  └─ .env.example
└─ README.md

----------------------------------------
1️⃣ SERVIDOR (Node.js + Firebase Admin)
----------------------------------------
servidor/index.js
```js
/* =========================================
   1. SERVIDOR EXPRESS + FIREBASE ADMIN
   ========================================= */
require('dotenv').config();                 // variables de entorno
const express   = require('express');
const cors      = require('cors');
const admin     = require('firebase-admin');

// 1.1 Inicializar Admin SDK con el archivo JSON que nos da Firebase
const serviceAccount = require('./serviceAccountKey.json');
admin.initializeApp({
  credential: admin.credential.cert(serviceAccount)
});

// 1.2 Firestore ya está listo
const db = admin.firestore();

// 1.3 Crear servidor
const app = express();
app.use(cors());
app.use(express.json()); // parsear body

// 1.4 Rutas
app.use('/api/usuarios', require('./routes/usuarios'));

// 1.5 Puerto
const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Servidor corriendo en puerto ${PORT}`));

// 1.6 Exportar db por si lo necesitan otros archivos
module.exports = { db };
```

----------------------------------------
servidor/models/usuarioModel.js
```js
/* =========================================
   2. MODELO → interactúa directamente con Firestore
   ========================================= */
const { db } = require('../index.js'); // importamos db

// 2.1 Guardar usuario (luego del registro)
const crearUsuario = async (uid, { nombre, email }) => {
  await db.collection('usuarios').doc(uid).set({
    nombre,
    email,
    creadoEn: new Date()
  });
};

// 2.2 Obtener todos los usuarios
const obtenerUsuarios = async () => {
  const snapshot = await db.collection('usuarios').get();
  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
};

module.exports = { crearUsuario, obtenerUsuarios };
```

----------------------------------------
servidor/controllers/usuariosController.js
```js
/* =========================================
   3. CONTROLADOR → lógica de la ruta
   ========================================= */
const usuarioModel = require('../models/usuarioModel');

// 3.1 POST /api/usuarios  (body = {uid, nombre, email})
const guardarUsuario = async (req, res) => {
  try {
    const { uid, nombre, email } = req.body;
    await usuarioModel.crearUsuario(uid, { nombre, email });
    res.json({ msg: 'Usuario guardado en Firestore' });
  } catch (e) {
    res.status(500).json({ error: e.message });
  }
};

// 3.2 GET /api/usuarios
const listarUsuarios = async (req, res) => {
  try {
    const usuarios = await usuarioModel.obtenerUsuarios();
    res.json(usuarios);
  } catch (e) {
    res.status(500).json({ error: e.message });
  }
};

module.exports = { guardarUsuario, listarUsuarios };
```

----------------------------------------
servidor/routes/usuarios.js
```js
/* =========================================
   4. RUTAS → delegan en el controlador
   ========================================= */
const express   = require('express');
const router    = express.Router();
const ctrl      = require('../controllers/usuariosController');

// 4.1 End-points
router.post('/', ctrl.guardarUsuario); // POST → guardar
router.get('/',  ctrl.listarUsuarios); // GET  → listar

module.exports = router;
```

----------------------------------------
2️⃣ CLIENTE (React)
----------------------------------------
cliente/src/firebase.js  ← configuración cliente (auth)
```js
/* =========================================
   5. FIREBASE CLIENTE → solo para Authentication
   ========================================= */
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey:            process.env.REACT_APP_API_KEY,
  authDomain:        process.env.REACT_APP_AUTH_DOMAIN,
  projectId:         process.env.REACT_APP_PROJECT_ID,
  storageBucket:     process.env.REACT_APP_STORAGE_BUCKET,
  messagingSenderId: process.env.REACT_APP_MESSAGING_SENDER_ID,
  appId:             process.env.REACT_APP_APP_ID
};

// 5.1 Inicializar Firebase
const app  = initializeApp(firebaseConfig);
export const auth = getAuth(app);
```

----------------------------------------
cliente/src/api.js  ← llamadas al servidor
```js
/* =========================================
   6. LLAMADAS AL SERVIDOR (axios)
   ========================================= */
import axios from 'axios';

const API = axios.create({
  baseURL: process.env.REACT_APP_SERVER_URL || 'http://localhost:4000/api'
});

// 6.1 Guardar usuario en Firestore (después del registro)
export const guardarUsuario = (uid, nombre, email) =>
  API.post('/usuarios', { uid, nombre, email });

// 6.2 Obtener lista de usuarios
export const obtenerUsuarios = () =>
  API.get('/usuarios');
```

----------------------------------------
cliente/src/App.js  ← componente principal
```js
/* =========================================
   7. APP REACT → diseño super sencillo
   ========================================= */
import React, { useEffect, useState } from 'react';
import { auth } from './firebase';
import { createUserWithEmailAndPassword, signInWithEmailAndPassword, onAuthStateChanged, signOut } from 'firebase/auth';
import { guardarUsuario, obtenerUsuarios } from './api';
import './styles.css';

function App() {
  const [user, setUser]       = useState(null);
  const [nombre, setNombre]   = useState('');
  const [email, setEmail]     = useState('');
  const [password, setPassword] = useState('');
  const [lista, setLista]     = useState([]);

  // 7.1 Detectar si hay usuario logueado
  useEffect(() => {
    const unsub = onAuthStateChanged(auth, u => {
      setUser(u);
      if (u) cargarUsuarios();
    });
    return unsub;
  }, []);

  // 7.2 Registrar
  const registrar = async () => {
    try {
      const cred = await createUserWithEmailAndPassword(auth, email, password);
      await guardarUsuario(cred.user.uid, nombre, email);
      alert('Usuario registrado y guardado en Firestore');
    } catch (e) { alert(e.message); }
  };

  // 7.3 Login
  const login = async () => {
    try {
      await signInWithEmailAndPassword(auth, email, password);
    } catch (e) { alert(e.message); }
  };

  // 7.4 Logout
  const logout = () => signOut(auth);

  // 7.5 Cargar usuarios
  const cargarUsuarios = async () => {
    const { data } = await obtenerUsuarios();
    setLista(data);
  };

  return (
    <div className="container">
      <h1>React + Firebase Auth + Firestore</h1>

      {!user ? (
        <div className="form">
          <input placeholder="Nombre" value={nombre} onChange={e => setNombre(e.target.value)} />
          <input placeholder="Email" value={email} onChange={e => setEmail(e.target.value)} />
          <input placeholder="Contraseña" type="password" value={password} onChange={e => setPassword(e.target.value)} />
          <button onClick={registrar}>Registrarse</button>
          <button onClick={login}>Iniciar sesión</button>
        </div>
      ) : (
        <div>
          <p>Bienvenido {user.email}</p>
          <button onClick={logout}>Cerrar sesión</button>
          <h3>Usuarios en Firestore</h3>
          <ul>{lista.map(u => <li key={u.id}>{u.nombre} - {u.email}</li>)}</ul>
        </div>
      )}
    </div>
  );
}

export default App;
```

----------------------------------------
cliente/src/index.js
```js
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);
```

----------------------------------------
cliente/src/styles.css  (mínimo)
```css
.container { font-family: Arial; max-width: 400px; margin: auto; padding: 20px; }
input, button { display: block; width: 100%; margin: 5px 0; padding: 8px; }
```

----------------------------------------
3️⃣ ARCHIVOS DE ENTORNO
----------------------------------------
servidor/.env.example
```
PORT=4000
```

cliente/.env.example
```
REACT_APP_API_KEY=TU_API_KEY
REACT_APP_AUTH_DOMAIN=TU_AUTH_DOMAIN
REACT_APP_PROJECT_ID=pruebaapi-66320
REACT_APP_STORAGE_BUCKET=TU_STORAGE_BUCKET
REACT_APP_MESSAGING_SENDER_ID=TU_SENDER_ID
REACT_APP_APP_ID=TU_APP_ID
REACT_APP_SERVER_URL=http://localhost:4000/api
```

----------------------------------------
4️⃣ ARRANQUE PASO A PASO
----------------------------------------
1. Copia tu `serviceAccountKey.json` dentro de `servidor/`
2. Renombra `.env.example` → `.env` (ambas carpetas) y completa datos
3. Servidor:
```bash
cd servidor
npm init -y
npm install express cors dotenv firebase-admin
node index.js
```
4. Cliente:
```bash
cd cliente
npx create-react-app . --template blank
npm install axios firebase
npm start
```

----------------------------------------
¡Listo! Tienes **React** (solo vista) + **Node.js** (administra Firestore) + **Firebase Auth** (email y redes sociales) con **cada línea explicada**.
