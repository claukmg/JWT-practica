# API con JWT en FastAPI

Claudia Medina 28182853
Repositorio: https://github.com/claukmg/JWT-practica.git

## Ejecución

```bash
pip install fastapi uvicorn "python-jose[cryptography]" "passlib[bcrypt]" "bcrypt<4.1" python-multipart python-dotenv
python -m uvicorn main:app --reload
```

Swagger UI en http://127.0.0.1:8000/docs

Usuarios de prueba: `maria@ujap.edu.ve` / `profesor123` (rol profesor) y `estudiante@ujap.edu.ve` / `estudiante123` (rol estudiante).

---

# Parte A — Conceptual

## P1. ¿Qué significa JWT y cuáles son sus tres partes?

**JWT** significa *JSON Web Token*. Es un estándar para transmitir información entre cliente y servidor como un token firmado. El servidor lo emite al hacer login y el cliente lo envía en cada petición (`Authorization: Bearer <token>`). Como el servidor solo verifica la firma, no necesita guardar sesiones (es *stateless*).

Tiene tres partes separadas por puntos (`header.payload.signature`):

1. **Header:** indica el algoritmo de firma (`alg`, por ejemplo HS256) y el tipo de token (`typ: JWT`).
2. **Payload:** contiene los datos (*claims*): `sub` (quién es el usuario), `role` (permisos) y `exp` (fecha de expiración).
3. **Signature:** es `HMACSHA256(base64(header) + "." + base64(payload), SECRET_KEY)`. Garantiza que nadie modificó el token.

## P2. ¿Por qué el payload NO es seguro para guardar contraseñas?

Porque el payload solo está codificado en **Base64**, no cifrado. Base64 es reversible y cualquiera puede decodificarlo sin ninguna clave. La firma protege la *integridad* del token (que no fue alterado), pero no su *confidencialidad*. Por eso cualquiera que intercepte o copie el token puede leer todo el payload. Nunca deben ir contraseñas, CVV, datos médicos ni otros datos sensibles; solo información mínima como `sub`, `role` y `exp`.

## P3. ¿Qué sucede si alguien modifica el payload sin conocer el SECRET_KEY?

La firma deja de coincidir. La firma se calculó sobre el header y el payload originales usando el SECRET_KEY. Si un atacante cambia el payload (por ejemplo `"role": "profesor"`), el servidor recalcula la firma con su SECRET_KEY y obtiene un valor distinto al que trae el token. Sin el SECRET_KEY el atacante no puede generar una firma válida para su payload alterado. `jwt.decode` lanza `JWTError` y la API responde **401 Unauthorized** ("Token expirado o inválido").

## P4. Diferencia entre 401 Unauthorized y 403 Forbidden. ¿Cuándo usa cada uno FastAPI?

- **401 Unauthorized:** el usuario **no está autenticado**: no envió token, el token es inválido, fue modificado o está expirado, o las credenciales del login son incorrectas. Significa "no sé quién eres".
- **403 Forbidden:** el usuario **sí está autenticado**, pero **no tiene permiso** para ese recurso. Significa "sé quién eres, pero no puedes entrar aquí".

En esta API: `POST /login` con credenciales malas devuelve 401. `GET /privado` sin token o con token inválido/expirado devuelve 401 (lo lanza `get_current_user`). `GET /admin` con un
