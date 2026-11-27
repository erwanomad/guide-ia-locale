# Guide IA Locale — PC GIGABYTE

Transformez le PC GIGABYTE en serveur d'IA gratuit pour **AssAI** (assai360.com) et l'audit de code, en remplacement des crédits API Claude.

| Machine | Rôle | Détails |
|---|---|---|
| **PC GIGABYTE** (DESKTOP-FK0CUBF) | Serveur IA 24h/24 | i9-10980HK — 64 Go RAM — RTX 3080 Laptop 8 Go — Ollama |
| **Zenbook ASUS** | Travail et code | Claude Code via claude-code-router + développement AssAI |
| **NAS DS920+** | Données et projets | Code source assai360.com + rapports d'audit |

📄 **Version PDF : [Guide_IA_Locale_PC_GIGABYTE.pdf](Guide_IA_Locale_PC_GIGABYTE.pdf)**

---

## 1. Architecture

```
┌─────────────────┐    projets (SMB/Git)   ┌──────────────┐
│   NAS DS920+    │◄──────────────────────►│   Zenbook    │
│  assai360.com   │                        │ dev + Claude │
└────────┬────────┘                        └──────┬───────┘
         │ rapports d'audit                       │ API
         │                                        ▼
         │                                ┌──────────────┐
         └───────────────────────────────►│ PC GIGABYTE  │
                                          │ Ollama + GPU │
                                          └──────────────┘
```

Le Zenbook lit/écrit les projets sur le NAS, envoie les questions à l'API du PC, et le PC renvoie les réponses générées par le GPU. Les rapports d'audit nocturnes sont écrits sur le NAS.

## 2. Installation du serveur IA (Ollama)

### 2.1 Installation de base

PowerShell **en administrateur** :

```powershell
winget install Ollama.Ollama
```

Ou téléchargement manuel : https://ollama.com/download/windows

### 2.2 Variables d'environnement (API accessible au réseau)

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0", "Machine")
[Environment]::SetEnvironmentVariable("OLLAMA_KEEP_ALIVE", "30m", "Machine")
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "16384", "Machine")
```

> ⚠️ **Un redémarrage du PC est nécessaire** pour que ces variables s'appliquent au service Ollama.

### 2.3 Pare-feu Windows

```powershell
New-NetFirewallRule -DisplayName "Ollama API" -Direction Inbound -Protocol TCP -LocalPort 11434 -Action Allow
```

### 2.4 Démarrage manuel pour les tests (fenêtre à laisser ouverte)

```powershell
Stop-Process -Name "ollama app" -Force -ErrorAction SilentlyContinue
Stop-Process -Name "ollama" -Force -ErrorAction SilentlyContinue

$env:OLLAMA_HOST = "0.0.0.0"
$env:OLLAMA_CONTEXT_LENGTH = "16384"
$env:OLLAMA_KEEP_ALIVE = "30m"

ollama serve
```

Résultat attendu : `Listening on [::]:11434 (version ...)`

### 2.5 Rendre permanent

Un simple redémarrage du PC suffit : l'application Ollama redémarre toute seule avec les variables machine.
**Ne jamais mettre le PC en veille** (Paramètres > Système > Alimentation).

## 3. Modèles recommandés

| Modèle | Taille | Usage | Sur ce PC |
|---|---|---|---|
| `ministral3` | ~3B | Assistant robot AssAI, très réactif | ✅ entièrement en VRAM |
| `mistral:7b` | 7B | Généraliste Mistral, léger | ✅ entièrement en VRAM |
| `mistral-nemo` | 12B | Excellent en français, contexte 128k | ✅ juste dans la VRAM |
| `qwen2.5-coder:7b` | 7B | Code rapide pour Claude Code | ✅ entièrement en VRAM |
| `devstral-small:24b` | 24B | Audit nocturne, qualité max | 🐢 lent (déborde en RAM) |

```powershell
ollama pull ministral3
ollama pull mistral-nemo
ollama pull qwen2.5-coder:7b
ollama pull devstral-small:24b

ollama list   # vérifier ce qui est installé
```

- Catalogue complet : https://ollama.com/library
- Modèles Mistral/Ministral : https://ollama.com/search?q=ministral
- **Pilote NVIDIA** (passe de Vulkan à CUDA, +10-20 % de vitesse) : https://www.nvidia.com/drivers

## 4. Tests de vérification

```powershell
# Sur le PC :
curl http://localhost:11434        # → Ollama is running
netstat -ano | findstr 11434       # → 0.0.0.0:11434 (PAS 127.0.0.1)
ipconfig                           # noter l'Adresse IPv4
```

Depuis le **Zenbook** : navigateur → `http://IP_DU_PC:11434` → « Ollama is running ».

> Réservez ensuite cette IP en **bail statique** dans votre box (DHCP) pour qu'elle ne change jamais.

Test complet de génération depuis le Zenbook :

```bash
curl http://IP_DU_PC:11434/v1/chat/completions -d '{"model":"qwen2.5-coder:7b","messages":[{"role":"user","content":"dit bonjour"}]}'
```

## 5. Claude Code sans crédits (claude-code-router)

Claude Code parle le protocole Anthropic ; le routeur le traduit vers Ollama.
Sur le **Zenbook** :

```bash
npm install -g @musistudio/claude-code-router
```

Documentation : https://github.com/musistudio/claude-code-router

`~/.claude-code-router/config.json` :

```json
{
  "LOG": true,
  "Providers": [
    {
      "name": "ollama",
      "api_base_url": "http://IP_DU_PC:11434/v1/chat/completions",
      "api_key": "ollama",
      "models": [
        "qwen2.5-coder:7b",
        "mistral-nemo:12b",
        "devstral-small:24b"
      ],
      "transformer": { "use": ["openai"] }
    }
  ],
  "Router": {
    "default": "ollama,qwen2.5-coder:7b",
    "background": "ollama,qwen2.5-coder:7b",
    "think": "ollama,devstral-small:24b",
    "longContext": "ollama,mistral-nemo:12b",
    "longContextThreshold": 20000
  }
}
```

Lancer **`ccr code`** au lieu de `claude`. Gardez la commande `claude` classique avec vos crédits pour les tâches très difficiles.

## 6. AssAI — connexion à l'API locale

Si AssAI utilise un client OpenAI :

```bash
OPENAI_API_BASE=http://IP_DU_PC:11434/v1
OPENAI_API_KEY=ollama
OPENAI_MODEL=mistral-nemo        # ou ministral3 pour le robot
```

Si AssAI utilise le SDK **Anthropic** : passer par claude-code-router (`ANTHROPIC_BASE_URL=http://127.0.0.1:3456` avec `ccr` lancé) ou LiteLLM sur le PC : https://docs.litellm.ai

**Avantages pour un robot** : gratuit, sans limite de crédits, fonctionne même si la connexion Internet tombe (tout reste local).

## 7. Audit de code nocturne

Fichier `C:\AuditIA\audit-nuit.ps1` sur le PC — parcourt le code du NAS et écrit un rapport chaque nuit :

```powershell
$NasPath   = "\\IP_NAS\projets\assai360"      # partage NAS - à adapter
$ReportDir = "\\IP_NAS\projets\audits"
$Model     = "devstral-small:24b"
$Date      = Get-Date -Format "yyyy-MM-dd"
$Report    = "$ReportDir\audit-$Date.md"

New-Item -ItemType Directory -Force -Path $ReportDir | Out-Null
"# Audit IA du $Date`n" | Out-File $Report -Encoding utf8

Get-ChildItem $NasPath -Recurse -Include *.py,*.js,*.ts,*.ino | ForEach-Object {
    $rel = $_.FullName.Replace($NasPath, "")
    $code = Get-Content $_.FullName -Raw
    if ($code.Length -gt 15000) { $code = $code.Substring(0, 15000) }

    $body = @{
        model = $Model
        stream = $false
        messages = @(
            @{ role = "system"; content = "Tu es un auditeur de code senior. Reponds en francais, de facon concise : bugs potentiels, failles de securite, mauvaises pratiques. Si le fichier est bon, dis-le en une ligne." }
            @{ role = "user"; content = "Fichier : $rel`n$code" }
        )
    } | ConvertTo-Json -Depth 5

    try {
        $resp = Invoke-RestMethod -Uri "http://localhost:11434/api/chat" -Method Post -Body $body -ContentType "application/json" -TimeoutSec 600
        "## $rel`n$($resp.message.content)`n" | Out-File $Report -Append -Encoding utf8
    } catch {
        "## $rel`nERREUR : $($_.Exception.Message)`n" | Out-File $Report -Append -Encoding utf8
    }
}
```

Planification à 23h chaque soir :

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\AuditIA\audit-nuit.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 23:00
Register-ScheduledTask -TaskName "AuditIA-nocturne" -Action $action -Trigger $trigger -RunLevel Highest
```

Chaque matin, le rapport est disponible dans `\\NAS\projets\audits\audit-AAAA-MM-JJ.md`.

## 8. Sécurité — règles à respecter

- L'API Ollama n'a **AUCUNE authentification** : acceptable sur un réseau domestique, mais **n'ouvrez JAMAIS le port 11434 sur votre box** (pas de redirection de port vers Internet).
- Ne jamais mettre le PC en veille quand il sert de serveur, mais verrouillez la session (Win+L).
- Les modèles restent sur le PC (`C:\Users\...\.ollama\models`) : aucune donnée ne sort de chez vous.
- Sauvegardez régulièrement le dossier `.ollama\models` sur le NAS pour ne pas re-télécharger ~20 Go après un crash.
- Pour AssAI connecté à un robot : même réseau local, sans exposition Internet directe.

## 9. Dépannage rapide

| Symptôme | Cause | Solution |
|---|---|---|
| La fenêtre PowerShell se ferme aussitôt | Script lancé par double-clic | Win+X → Terminal admin, commandes à la main |
| `bind: Only one usage of each socket address` | Port 11434 déjà pris | `Stop-Process -Name ollama -Force` puis relancer |
| Écoute sur 127.0.0.1 au lieu de 0.0.0.0 | Variables posées après le démarrage | Redémarrer le PC |
| Inaccessible depuis le Zenbook | Pare-feu ou IP erronée | Vérifier la règle « Ollama API » et l'IPv4 avec `ipconfig` |
| `ollama` commande introuvable | PATH pas encore à jour | Redémarrer le PC puis retenter |
| Réponses très lentes | Modèle plus grand que la VRAM | Petit modèle le jour, 24B la nuit |

## 10. Liens utiles

| Ressource | Lien |
|---|---|
| Ollama (Windows) | https://ollama.com/download/windows |
| Catalogue des modèles | https://ollama.com/library |
| Modèles Mistral / Ministral | https://ollama.com/search?q=ministral |
| claude-code-router | https://github.com/musistudio/claude-code-router |
| LiteLLM | https://docs.litellm.ai |
| Pilotes NVIDIA | https://www.nvidia.com/drivers |
| Documentation API Ollama | https://github.com/ollama/ollama/blob/main/docs/api.md |
| AssAI | https://www.assai360.com |
