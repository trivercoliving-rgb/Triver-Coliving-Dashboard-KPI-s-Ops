# Rutina — Publicar datos en el Visor TRIVER

**Quién:** Diego Medina · **Cuándo:** cada vez que cierres o actualices un mes · **Tiempo:** 3 minutos

## Pasos

1. Abre tu `index.html` local (modo editor) y verifica que los datos del mes estén completos.
2. Haz clic en **🔐 Publicar snapshot**, escribe la contraseña del equipo dos veces y se descargará `data.enc.json`.
3. En github.com, entra al repo → **Add file → Upload files** → arrastra `data.enc.json` → mensaje `Datos al [fecha]` → **Commit changes**.
4. Espera 1–2 minutos, abre el link del visor y confirma que el sello "Datos al:" muestra la fecha nueva.

## Cambiar la contraseña (cuando alguien sale del equipo)

1. Publica un snapshot nuevo con la **contraseña nueva** (pasos 2 y 3).
2. Comparte la contraseña nueva por un canal privado, nunca en grupos abiertos.
3. La contraseña anterior deja de funcionar en cuanto se sube el archivo nuevo.

## Reglas

- La contraseña debe tener **mínimo 12 caracteres**. Mejor una frase de 4 palabras.
- La contraseña **no** se guarda en el código ni en el repo. Si la olvidas, solo publicas con una nueva.
- Sigue exportando tu backup `.json` normal. El snapshot cifrado **no** sustituye tu respaldo.
