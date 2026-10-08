# COMPAT — shamela sunucusunu diger uygulamalara baglama

Tek komut: `nodeC:\dev\mcp\shamela-mcp\dist\index.js`
Bu komut asagidaki uclude aynen boyle kullanilir.

## Claude Desktop
Konum: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "shamela": {
      "command": "node",
      "args": [
        "C:\\dev\\mcp\\shamela-mcp\\dist\\index.js"
      ],
      "env": {
        "SHAMELA_INSTALL_ROOT": "C:\\shamela4",
        "SHAMELA_JRE": "java"
      }
    }
  }
}
```

## Codex CLI
Konum: `%USERPROFILE%\.codex\config.toml` (satirlari ekleyin)

```toml
# Codex CLI: %USERPROFILE%\.codex\config.toml

[mcp_servers.shamela]
command = "node"
args = ["C:\dev\mcp\shamela-mcp\dist\index.js"]

[mcp_servers.shamela.env]
SHAMELA_INSTALL_ROOT = "C:\shamela4"
SHAMELA_JRE = "java"

```

## ChatGPT (masaustu/web)
Yerel stdio sunuculari baglanti olarak dogrudan takilmaz; relay/Developer Mode gerekir.
Surum: `cmdc-stack/mcp/hostconfigs/CHATGPT.md`

> Merkezi kurulum/senkron icin: `cmdc-stack` deposu (`scripts/install.ps1`) tumunu
> tek manifestten kaydeder (Hermes / CommandCode dahil).
