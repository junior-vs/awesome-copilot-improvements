# Criando Links Simbólicos no Windows 11 com PowerShell

## O que é um Link Simbólico?

Um link simbólico (symlink) é um atalho que aponta para outro arquivo ou pasta. Diferente de um atalho `.lnk`, ele é transparente para o sistema operacional — programas o tratam como o arquivo/pasta original.

---

## Com Perfil de Administrador

### Arquivo → Arquivo

```powershell
# New-Item -ItemType SymbolicLink -Path <link> -Target <destino>
New-Item -ItemType SymbolicLink -Path "C:\links\meu-arquivo.txt" -Target "C:\original\arquivo.txt"
```

### Pasta → Pasta

```powershell
New-Item -ItemType SymbolicLink -Path "C:\links\minha-pasta" -Target "C:\original\pasta"
```

### Verificar se foi criado

```powershell
Get-Item "C:\links\meu-arquivo.txt" | Select-Object Name, LinkType, Target
```

---

## Sem Perfil de Administrador

Por padrão, criar symlinks no Windows requer privilégios de administrador. Há duas formas de contornar isso:

### Opção 1: Ativar o "Modo Desenvolvedor" (recomendado)

1. Abra **Configurações → Sistema → Para Desenvolvedores**
2. Ative **Modo Desenvolvedor**

Após ativar, o mesmo comando funciona sem elevação:

```powershell
New-Item -ItemType SymbolicLink -Path "$HOME\links\projeto" -Target "D:\repos\meu-projeto"
```

### Opção 2: Política de Grupo — "Create symbolic links"

1. Execute `gpedit.msc`
2. Vá em: **Configuração do Computador → Configurações do Windows → Configurações de Segurança → Políticas Locais → Atribuição de Direitos de Usuário**
3. Abra **"Criar links simbólicos"** e adicione seu usuário

> Requer reinicialização da sessão para ter efeito.

---

## Tipos de Links

| Tipo | Comando | Descrição |
|------|---------|-----------|
| **SymbolicLink** | `New-Item -ItemType SymbolicLink` | Aponta para arquivo ou pasta (caminho absoluto ou relativo) |
| **HardLink** | `New-Item -ItemType HardLink` | Somente para arquivos, mesmo volume, não requer admin |
| **Junction** | `New-Item -ItemType Junction` | Somente para pastas, mesmo volume, não requer admin |

### HardLink (sem admin, somente arquivos)

```powershell
New-Item -ItemType HardLink -Path "C:\links\copia-dura.txt" -Target "C:\original\arquivo.txt"
```

### Junction (sem admin, somente pastas locais)

```powershell
New-Item -ItemType Junction -Path "C:\links\pasta-junction" -Target "C:\original\pasta"
```

---

## Remover um Link

```powershell
# Não use Remove-Item -Recurse em links de pasta — apagaria o conteúdo original!
Remove-Item "C:\links\minha-pasta"        # pasta (link apenas)
Remove-Item "C:\links\meu-arquivo.txt"    # arquivo
```

---

## Resumo Rápido

| Tipo | Requer Admin | Escopo |
|------|-------------|--------|
| Symlink arquivo | Sim (ou Modo Dev) | Qualquer volume |
| Symlink pasta | Sim (ou Modo Dev) | Qualquer volume |
| HardLink | Não | Mesmo volume, somente arquivos |
| Junction | Não | Mesmo volume, somente pastas |

```powershell

New-Item -ItemType Junction -Path "skills"  -Target "D:\develop\repos\misc_repos\awesome-copilot-improvements\skills"

New-Item -ItemType Junction -Path "agents"  -Target "D:\develop\repos\misc_repos\awesome-copilot-improvements\agents"

```

Get-ChildItem "my-env\.github\instructions" | Select-Object -ExpandProperty Name | Out-File "my-env\lista-instruction.txt"
