# 🛠️ Troubleshooting — ESP8266 / WEMOS D1 Mini no Windows

Registro dos problemas encontrados durante o desenvolvimento com ESP8266 (WEMOS D1 Mini R2) no Windows, usando Arduino IDE e PlatformIO (VS Code).

---

## Problema 1: Erro ao fazer upload — `PermissionError(13)` na porta COM

### O que aconteceu

Ao tentar fazer upload do sketch ou apagar a flash via `esptool`, o seguinte erro aparecia:

```
A fatal error occurred: Could not open COM4, the port is busy or doesn't exist.
PermissionError(13, 'Um dispositivo conectado ao sistema não está funcionando.', None, 31)
```

### Causas identificadas

- Outro processo segurando a porta serial (Serial Monitor aberto, instância anterior do esptool travada, Arduino IDE em segundo plano)
- Driver CH340 corrompido ou desatualizado
- Porta COM trocando de número após reinicialização do sistema

### O que foi tentado

1. Fechar o Serial Monitor e outros programas que usam porta serial
2. Matar processos Python travados via terminal:
   ```bash
   taskkill /F /IM python.exe
   ```
3. Desabilitar e reabilitar o dispositivo no Gerenciador de Dispositivos
4. Verificar se a porta mudou de número (ex: COM4 → COM3) após reinicialização

### ✅ Solução que funcionou

O problema era o **driver CH340 corrompido**. A solução foi reinstalar o driver corretamente:

1. Abrir o **Gerenciador de Dispositivos**
2. Expandir **Portas (COM e LPT)**
3. Clicar com botão direito em **USB-SERIAL CH340** → **Desinstalar dispositivo**
4. Marcar a opção **"Excluir o software de driver deste dispositivo"**
5. Desconectar o ESP8266 do USB
6. Baixar e instalar o driver CH340 oficial como Administrador
7. Reconectar o ESP8266

**Referências que ajudaram a resolver:**
- 📺 [Vídeo tutorial — Instalação do driver CH340 (YouTube)](https://youtu.be/t-cLmtPBsD0?si=hWyLZWujGl8xZiBn)
- 📄 [Guia completo do driver CH340 (Sparks)](https://sparks-gogo-co-nz.translate.goog/ch340.html?_x_tr_sl=en&_x_tr_tl=pt&_x_tr_hl=pt&_x_tr_pto=tc)

---

## Problema 2: ESP8266 em loop de reset — `wdt reset` no Serial Monitor

### O que aconteceu

Após conectar o D1 Mini ao Serial Monitor, a seguinte mensagem aparecia repetidamente:

```
rst cause:4, boot mode:(3,6)
wdt reset
load 0x4010f000, len 3460...
```

### Causa identificada

Firmware antigo corrompido gravado na flash do ESP8266. O `rst cause:4` indica reset por **Watchdog Timer** — o chip travava ao tentar executar o firmware anterior e reiniciava em loop.

### ✅ Solução

Apagar completamente a flash do ESP8266 via `esptool`:

```bash
python -m esptool --port COM3 erase-flash
```

> ⚠️ Substituir `COM3` pela porta correta do seu dispositivo (verificar no Gerenciador de Dispositivos).

Se o comando travar em `Connecting...`, segurar o botão **FLASH/BOOT** do D1 Mini enquanto o comando roda e soltar ao aparecer a barra de progresso.

---

## Problema 3: Crash do ESP8266 — `Exception (9)` / `LoadStoreAlignmentCause`

### O que aconteceu

O ESP8266 entrava em pânico e despejava o seguinte no Serial Monitor:

```
Exception (9):
epc1=0x4023718e ...
LoadStoreAlignmentCause: Load or store to an unaligned address
_dtoa_r() at dtoa.c:397
timer1_isr_handler() at core_esp8266_timer.cpp:37
```

### Causa identificada

Uso de funções de formatação de ponto flutuante (`sprintf("%.2f", ...)`, `Serial.print(float)`, `String(float)`) **dentro de uma ISR (rotina de interrupção de timer)**. O Xtensa LX106 não suporta esse tipo de operação em contexto de interrupção.

### ✅ Solução

Mover toda formatação e impressão de dados para fora da ISR, usando uma flag de sinalização:

```cpp
volatile bool flagTimer = false;
volatile float valorLeitura = 0.0;

void ICACHE_RAM_ATTR timer_callback() {
    valorLeitura = leitura / 10.0; // cálculo simples: ok
    flagTimer = true;              // apenas sinaliza
}

void loop() {
    if (flagTimer) {
        flagTimer = false;
        Serial.print(valorLeitura); // formatação AQUI, fora da ISR
    }
}
```

---

## Ambiente utilizado

| Item | Versão / Modelo |
|---|---|
| Placa | WEMOS D1 Mini R2 (ESP8266) |
| Sistema Operacional | Windows |
| IDE | Arduino IDE + PlatformIO (VS Code) |
| Driver USB-Serial | CH340 |
| esptool | v5.2.0 |
| Framework | Arduino / ESP8266 Arduino Core 3.1.2 |

---

## SnapEDA:

https://www.snapeda.com/home/

## Dicas gerais

- Sempre verificar qual porta COM o dispositivo está usando no **Gerenciador de Dispositivos** antes de rodar comandos
- Após reiniciar o notebook, a porta COM pode mudar de número
- Usar `python -m esptool` ao invés de `esptool.py` diretamente no Windows
- Manter o driver CH340 atualizado para evitar problemas de conexão
