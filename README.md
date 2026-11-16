# OpenDBO Monero (XMR) — Pack de Instalación y Actualización

Paquete Plug & Play con todos los componentes necesarios para desplegar la economía **Monero (XMR)**, minería CPU RandomX (`xmrig`), sistema de comisiones (`5%`), panel de reclamo in-game y gestión dinámica de precios en **OpenDBO**.

---

## 1. Estructura del paquete

```text
OpenDBO_XMR_UpdatePack/
├── 1_Database/
│   └── crypto_mining_schema.sql          # Tablas de minería, tesorería y retiros XMR
├── 2_Server/
│   ├── config/                           # item_prices.ini, price_catalog.tsv y configs INI
│   ├── AuthServer.exe
│   ├── CharServer.exe
│   ├── ChatServer.exe
│   ├── GameServer.exe
│   ├── MasterServer.exe
│   └── QueryServer.exe
├── 3_CryptoPoolBridge/
│   ├── config.json                       # Configuración de pool, wallet, MySQL y RPC
│   ├── cpu_worker_ipc.js                 # Puente Named Pipe <-> xmrig.exe (cálculo PPS por diff)
│   ├── pool_server.js                    # Pool Stratum v1 local (puerto 3333)
│   ├── payout_worker.js                  # Procesador de retiros XMR y cola WAITING_ADMIN
│   ├── nicehash_client.js                # Integración NiceHash API
│   ├── treasury_cli.js                   # CLI de auditoría de tesorería y fees
│   ├── PriceDashboard.py                 # Dashboard visual de precios (Cash Shop / Items-NPC)
│   ├── xmrig.exe / WinRing0x64.sys
│   └── package.json / node_modules/
├── 4_Client_Patch/
│   └── Client.exe                        # Cliente con MiningRigGui (F9), reclamo XMR y Store 2s
├── 5_Codigo_Fuente/
│   ├── DboClient/                        # Código fuente modificado del cliente (sin compilar)
│   ├── DboServer/                        # Código fuente modificado del servidor (sin compilar)
│   ├── DboShared/                        # Opcodes y estructuras compartidas (sin compilar)
│   ├── economy_xmr_changes.patch         # Parche git unificado con todos los cambios
│   └── LISTA_ARCHIVOS.txt                # Inventario de los archivos fuente incluidos
├── INSTALAR.bat                          # Lanzador del instalador automático
├── instalar_actualizacion.ps1            # Script PowerShell de copia/instalación y SQL
├── update_config.json                    # Rutas destino y credenciales de instalación
├── Arrancar_Todo.bat                     # Arranque completo de Bridge + Servidores OpenDBO
├── Abrir_Dashboard_Precios.bat           # Lanzador de PriceDashboard.py
├── CATALOGO_IDS_DBO.md                   # Catálogo de IDs de ítems y tiendas
├── DOCUMENTACION_ECONOMIA_XMR.pdf / .tex # Documentación técnica y económica completa
├── GUIA_CONFIGURACION_MINERIA.pdf / .tex # Guía de configuración de minería y NiceHash
└── LEEME_INSTALACION.txt                 # Resumen rápido de instalación
```

---

## 2. Requisitos previos

- **Windows** (64-bit) con **MySQL / MariaDB** en ejecución (`127.0.0.1:3306` por defecto).
- **Node.js** (para `pool_server.js`, `cpu_worker_ipc.js`, `payout_worker.js` y `treasury_cli.js`).
- **Python 3** (opcional, necesario para ejecutar `PriceDashboard.py`).
- **Monero Wallet RPC (`monero-wallet-rpc`)** en `127.0.0.1:18082/json_rpc` si deseas procesar pagos automáticos on-chain desde `payout_worker.js`.

---

## 3. Instalación rápida

1. **Detén todos los servidores** de OpenDBO y `CryptoPoolBridge` si están en ejecución.
2. Revisa **`update_config.json`** si tus rutas de instalación o credenciales MySQL son distintas a las predeterminadas:
   - `SERVER_EXECUTION_ENV_DIR`: carpeta `ExecutionEnv` del servidor.
   - `CRYPTO_BRIDGE_INSTALL_DIR`: carpeta donde se instalará `CryptoPoolBridge`.
   - `CLIENT_DIR`: carpeta del cliente (`DragonBall`) donde va `Client.exe`.
   - `MYSQL_BIN_DIR`, `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, `MYSQL_PASSWORD`, `DB_ACC`, `DB_CHAR`.
3. Haz doble clic en **`INSTALAR.bat`** (o ejecuta `instalar_actualizacion.ps1` en PowerShell). El instalador:
   - Aplica `1_Database/crypto_mining_schema.sql` sobre MySQL.
   - Copia los binarios y configuraciones de `2_Server/` al destino configurado.
   - Copia `3_CryptoPoolBridge/` y genera `config.json`.
   - Copia `4_Client_Patch/Client.exe` al directorio del cliente.

---

## 4. Configuración de Wallets y Economía XMR

Edita `config.json` en la carpeta instalada de `CryptoPoolBridge`:

- **`FEE_MINING_PCT`**: porcentaje de comisión de minería (`5` por defecto).
- **`FEE_TREASURY_WALLET`**: wallet destinada a recibir las comisiones (`5%`).
- **`DEVELOPER_TREASURY_WALLET` / `OWNER_TREASURY_WALLET`**: wallet del pool donde mina `xmrig.exe` (p. ej. dirección Monero con nombre de worker `/juego`).
- **`EXTERNAL_POOL_URL`**: pool externo RandomX (por defecto `xmr-eu.kryptex.network:7029`).
- **`MONERO_RPC`**: URL del RPC de la wallet del servidor (`http://127.0.0.1:18082/json_rpc`), mínimo de retiro (`min_withdraw_units`: `1000000` = `0.0001000000 XMR`), intervalo de sondeo (`10000 ms`) y `auto_transfer_enabled`.

### Reglas del sistema implementado
- **Escala interna:** `1 unidad = 10^-10 XMR = 100 piconeros` (`1 XMR = 10,000,000,000` unidades).
- **Recompensa por *share* (PPS):** `cpu_worker_ipc.js` lee la dificultad real (`diff`) de cada *share* aceptado por `xmrig.exe` y calcula el pago proporcional respecto a la dificultad y recompensa de bloque de la red Monero, descontando el `5%` a `dbo_char.treasury_ledger` y acreditando el `95%` a `dbo_char.mining_rewards_queue`.
- **Reclamos in-game (`F9` → XMR Claim):** mínimo `0.0001000000 XMR`, máximo del `50%` del saldo cada 7 días. Si la wallet del servidor no tiene suficiente saldo desbloqueado, la solicitud queda en cola `WAITING_ADMIN` hasta confirmación del administrador.
- **Comisión en compras (5%):** aplicada en compras del servidor de juego y registrada en tesorería.
- **Refresco de precios:** `GameServer` y `Client.exe` actualizan los overrides de `item_prices.ini` en vivo cada `2` segundos.

---

## 5. Arranque y herramientas

- **Arrancar todo el entorno:** doble clic en **`Arrancar_Todo.bat`** (inicia `pool_server.js`, `cpu_worker_ipc.js`, `payout_worker.js`, `MasterServer`, `QueryServer`, `AuthServer`, `CharServer`, `ChatServer` y `GameServer`).
- **Editor visual de precios:** doble clic en **`Abrir_Dashboard_Precios.bat`** para abrir `PriceDashboard.py` (Cash Shop e Items/NPC).
- **Auditoría de comisiones (`treasury_cli.js`):**
  ```powershell
  node treasury_cli.js summary
  ```

---

## 6. Código fuente sin compilar (`5_Codigo_Fuente/`)

Si deseas compilar tú mismo o integrar los cambios sobre otro repositorio de `OpenDBO-Core`:
- Copia los archivos de `5_Codigo_Fuente/DboClient`, `DboServer` y `DboShared` sobre tu árbol fuente, o aplica `5_Codigo_Fuente/economy_xmr_changes.patch`.
- Consulta `5_Codigo_Fuente/LISTA_ARCHIVOS.txt` para ver la lista exacta de archivos modificados y añadidos.

---

## 7. Documentación adicional

- **`DOCUMENTACION_ECONOMIA_XMR.pdf`**: arquitectura técnica, flujos de red, opcodes, esquema SQL y modelo económico.
- **`GUIA_CONFIGURACION_MINERIA.pdf`**: guía paso a paso de configuración de minería CPU, Kryptex/pool y NiceHash.
- **`CATALOGO_IDS_DBO.md`**: referencia de IDs de ítems y catálogo en inglés.
