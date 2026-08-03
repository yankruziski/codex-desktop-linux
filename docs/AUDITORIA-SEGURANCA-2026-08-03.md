# Auditoria de Segurança — codex-desktop-linux

- **Data:** 2026-08-03
- **Auditor:** Yan Kruziski (com Claude Code)
- **Upstream auditado:** `ilysenko/codex-desktop-linux`
- **Commit:** `ec38ca6` — tree `d1360367a727f959e875ad0e0fc84246a3479cde`
- **Fork:** `yankruziski/codex-desktop-linux`
- **Veredito:** ✅ **Sem malware, backdoor ou exfiltração de dados.** Aprovado para uso.

> A árvore Git auditada foi comparada byte a byte com a do fork clonado: hashes idênticos.
> Esta auditoria cobre exatamente o código que está em `~/github/codex-desktop-linux`.

---

## 1. Escopo

551 arquivos versionados. O projeto é bem maior do que o post original sugere ("hack de
20 minutos"): além do repack do DMG, inclui crates Rust para *computer-use*, gravação de
tela, TTS, extensão GNOME Shell, serviço systemd de update e empacotamento deb/rpm/pacman.

Vetores verificados: execução dinâmica, escalonamento de privilégio, origem dos binários,
exfiltração de credenciais, persistência, supply chain de CI e reputação do projeto.

## 2. Resultados por vetor

| # | Vetor | Resultado |
|---|---|---|
| 1 | Binários compilados commitados | ✅ **Nenhum.** Todos os 551 arquivos são texto auditável. |
| 2 | `curl \| bash` | ✅ Só `rustup.rs` (oficial), em `install-deps.sh` e docs. |
| 3 | `eval` / `new Function()` | ✅ Só em testes e no motor de patch (reescreve JS do app — é a função do projeto). |
| 4 | Ofuscação / base64 | ✅ Nenhum blob. O único `base64` descriptografa chave local via `safeStorage` do Electron. |
| 5 | Acesso a credenciais | ✅ Nenhuma leitura de `~/.ssh`, `~/.aws`, `auth.json` ou keyring. |
| 6 | Exfiltração / upload | ✅ **Nenhum POST/upload.** Única saída de rede não-download é um `HEAD` para checar versão. |
| 7 | Telemetria de terceiros | ✅ Sem PostHog/Sentry/Mixpanel/Amplitude. As flags `analytics` são do app da OpenAI, preservadas — não adicionadas. |
| 8 | Persistência oculta | ✅ Não toca `.bashrc`, `.profile`, cron nem autostart. Só um serviço systemd **de usuário** para updates, opcional. |
| 9 | Uso de `sudo` | ✅ Apenas gerenciadores de pacote (`apt`/`dnf`/`pacman`/`zypper`). **O `install.sh` não usa sudo.** |
| 10 | Domínios contactados | ✅ Todos legítimos (ver §3). |
| 11 | Reputação | ✅ 3.278 stars, 417 forks, MIT, conta de 2012, 30+ contribuidores reais, ativo. |

## 3. Origem dos binários — o ponto que mais importa

O app **não** vem de servidor do autor do repo. Vem direto da OpenAI:

```
https://persistent.oaistatic.com/codex-app-prod/ChatGPT.dmg   # CDN oficial OpenAI
https://chatgpt.com/codex/install.sh                          # Codex CLI oficial
https://github.com/electron/electron/releases/...             # Electron oficial
https://artifacts.electronjs.org/headers/dist                 # headers Electron
https://deb.nodesource.com, https://nodejs.org/dist           # Node oficial
https://sh.rustup.rs, https://static.crates.io                # Rust oficial
https://huggingface.co                                        # modelo TTS Kokoro (read-aloud, opcional)
```

Nenhum domínio de terceiro, encurtador, IP hardcoded ou host suspeito. O repo só fornece
**patches em texto** aplicados sobre o binário oficial — e todo patch é legível em
`scripts/patches/` e `linux-features/`.

## 4. Modelo de privilégio

- `install.sh` roda **sem root**, instalando em diretório local (`./codex-app`).
- Instalação é **transacional**: monta um candidato, valida (`validate-upstream-dmg.js`) e
  só promove se aprovado; senão descarta e mantém o app anterior.
- A política polkit (`com.github.ilysenko...update.policy`) usa `auth_admin` em **todas** as
  ações, inclusive `allow_active`. Ou seja: pede senha de admin sempre, sem exceção para
  sessão ativa. É a configuração conservadora — o oposto do que um backdoor faria.
- O serviço systemd de update é **de usuário** (`WantedBy=default.target`), não de sistema.

## 5. Supply chain de CI

- Único workflow com `pull_request_target` (`contributor-pr-limit.yml`) faz checkout do
  **branch default**, nunca do código do PR — padrão seguro contra o ataque clássico
  de `pull_request_target`.
- Actions majoritariamente pinadas por SHA completo.
- `webview-server.py` faz bind em `127.0.0.1` por padrão.

## 6. Observações (não são vulnerabilidades)

1. **Sem verificação de assinatura do DMG.** O SHA256 é usado para detectar mudança e
   controlar rollback, não para validar contra um valor esperado. A âncora de confiança é
   o TLS do CDN da OpenAI. Aceitável, mas significa que a segurança do download depende do
   HTTPS — não há defesa extra caso o CDN da OpenAI seja comprometido.
2. **`validate_dmg_url` exige HTTPS mas não restringe o host.** Só é explorável se alguém
   definir `CODEX_UPSTREAM_DMG_URL` apontando para outro lugar. Não defina essa variável.
3. **Superfície de features poderosa.** `computer-use`, `record-and-replay` e
   `global-dictation` capturam tela, teclado e microfone **por design** — é a função delas,
   e os dados ficam locais. Ainda assim, só habilite o que for usar.
4. **Testado pelo autor no Ubuntu 25.10**; esta máquina é Ubuntu 24.04. Diferença de versão
   pode gerar atrito de build, não risco de segurança.

## 7. Como reproduzir a auditoria

```bash
cd ~/github/codex-desktop-linux
git rev-parse HEAD^{tree}                                  # deve bater com o hash do topo

# 1. nenhum binário commitado
git ls-files | while read f; do file -b --mime "$f" \
  | grep -qE 'text/|image/|inode/' || echo "BINÁRIO: $f"; done

# 2. todos os domínios contactados
grep -rhoE 'https?://[a-zA-Z0-9._-]+' --exclude-dir=.git . | sort -u

# 3. pipe-to-shell
grep -rnE '(curl|wget)[^|;]*\|[[:space:]]*(sudo )?(ba)?sh' --exclude-dir=.git .

# 4. upload de dados
grep -rnE "curl.*(-d |--data|-F )|\.post\(" --include='*.sh' --include='*.py' --include='*.rs' .
```

## 8. Recomendações operacionais

- Não definir `CODEX_UPSTREAM_DMG_URL` (mantém o download preso ao CDN oficial).
- Fazer `git fetch upstream` e revisar o diff antes de atualizar — a auditoria vale para
  este commit, não para commits futuros.
- Habilitar apenas as `linux-features` que forem realmente usadas.
