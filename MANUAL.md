# ps-buffs — Manual

Rastreador de buffs temporários para QBCore: outros recursos aplicam um buff por `citizenid`, o tempo é decrementado no servidor e o ícone correspondente aparece no HUD do jogador.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Buffs e enhancements](#buffs-e-enhancements)
5. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
6. [Eventos enviados ao HUD](#eventos-enviados-ao-hud)
7. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core` | Sim | Declarado em `dependencies` no `fxmanifest.lua`. Usa callbacks e `GetPlayerByCitizenId` |
| `ps-hud` | Sim, na prática | O recurso não desenha nada. Ele dispara `hud:client:BuffEffect` e `hud:client:EnhancementEffect`, que só o `ps-hud` consome |

---

## Instalação

1. Copie a pasta `ps-buffs` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qb-core
   ensure ps-hud
   ensure ps-buffs
   ```
3. Não há SQL nem itens de inventário. Os buffs vivem só em memória no servidor.

> Buffs não são persistidos. Ao reiniciar o recurso ou o servidor, todos os buffs ativos são perdidos.

---

## Configuração

Arquivo: `shared/config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.TickTime` | number (ms) | Sim | Intervalo do loop que decrementa o tempo de todos os buffs e atualiza o HUD. Padrão: `6000` |
| `Config.Buffs` | tabela | Sim | Catálogo de buffs disponíveis. A chave é o nome usado nos exports |

Cada entrada de `Config.Buffs` aceita:

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `type` | `"buff"` ou `"enhancement"` | Sim | Define qual evento de HUD é disparado. Não altere |
| `maxTime` | number (ms) | Sim | Teto de duração. Somar tempo além disso é truncado |
| `iconColor` | hex | Sim | Cor do ícone no HUD |
| `iconName` | string | Só para `type = "buff"` | Nome do ícone exibido no HUD |
| `progressColor` | hex | Só para `type = "buff"` | Cor da barra de progresso do ícone |

Os nomes dos índices dos `enhancement` (`super-hunger`, `super-thirst`, `super-health`, `super-armor`, `super-stress`) são usados pelo front-end do HUD para saber qual ícone de status colorir — não renomeie. Os índices dos `buff` podem ser renomeados, mas outros recursos que dependam deles precisarão ser ajustados.

---

## Buffs e enhancements

**Buff** (`type = "buff"`) é um efeito novo: ganha um ícone próprio no HUD com barra de progresso. Vêm configurados: `hacking`, `intelligence`, `luck`, `stamina`, `strength`, `swimming`.

**Enhancement** (`type = "enhancement"`) melhora um status que o jogador já tem: o ícone do status existente fica amarelo. Vêm configurados: `super-hunger`, `super-thirst`, `super-health`, `super-armor`, `super-stress`.

Adicionar um buff **apenas mostra o ícone**. O efeito no jogo precisa ser implementado por quem chama — o recurso já traz alguns prontos (ver `StaminaBuffEffect`, `SwimmingBuffEffect`, `AddHealthBuff`, `AddArmorBuff`, `AddStressBuff` abaixo).

Os enhancements `super-hunger` e `super-thirst` não têm efeito embutido: eles só marcam o estado. O arquivo `snippets` traz o código que precisa ser **substituído** dentro do `qb-core` (`client/loops.lua`, `client/events.lua` e `server/events.lua`) para que a fome e a sede caiam pela metade enquanto o buff estiver ativo.

---

## Entrypoints para outros recursos

### Exports do client

```lua
-- Aplica um buff ao jogador local (vai ao servidor via callback). Retorna bool.
exports['ps-buffs']:AddBuff('hacking', 15000)

-- true se o jogador local tem o buff ativo.
if exports['ps-buffs']:HasBuff('hacking') then
    -- ex.: dar mais tempo no minigame
end

-- Dados do buff ativo (tempo restante, ícone, cores, progresso). nil se o nome não existe no config.
local data = exports['ps-buffs']:GetBuff('stamina')

-- Tabela com todos os buffs ativos já no formato de NUI. Útil ao relogar.
local nui = exports['ps-buffs']:GetBuffNUIData()
```

Efeitos prontos, também no client:

```lua
-- Corre mais rápido e regenera stamina enquanto ativo.
exports['ps-buffs']:StaminaBuffEffect(15000, 1.4)   -- (tempo em ms, multiplicador de sprint)

-- Nada mais rápido e regenera stamina enquanto ativo.
exports['ps-buffs']:SwimmingBuffEffect(20000, 1.4)  -- (tempo em ms, multiplicador de natação)

-- Regenera vida a cada 5s, até 200.
exports['ps-buffs']:AddHealthBuff(10000, 10)        -- (tempo em ms, HP por tick)

-- Regenera colete a cada 5s, até 100.
exports['ps-buffs']:AddArmorBuff(30000, 10)         -- (tempo em ms, armor por tick)

-- Remove estresse a cada 5s via hud:server:RelieveStress.
exports['ps-buffs']:AddStressBuff(30000, 10)        -- (tempo em ms, estresse por tick)
```

### Exports do servidor

Diferente do client, os exports do servidor recebem o `citizenid` (não o `source`).

```lua
-- (source, citizenid, buffName, time). Se o jogador já tem o buff, soma o tempo até maxTime.
exports['ps-buffs']:AddBuff(source, citizenid, 'strength', 60000)

-- Remove o buff e apaga o ícone do HUD.
exports['ps-buffs']:RemoveBuff(citizenid, 'strength')

-- true se o citizenid tem o buff ativo.
exports['ps-buffs']:HasBuff(citizenid, 'strength')
```

### Callbacks do servidor

```lua
-- Tabela { [buffName] = tempoRestante } do jogador que chamou. nil se não tem nenhum.
QBCore.Functions.TriggerCallback('buffs:server:fetchBuffs', function(buffs) end)

-- Aplica um buff ao jogador que chamou. Retorna bool.
QBCore.Functions.TriggerCallback('buffs:server:addBuff', function(ok) end, buffName, time)
```

---

## Eventos enviados ao HUD

O servidor dispara estes eventos no client do jogador afetado. Quem os implementa é o `ps-hud`.

| Evento | Payload | Quando |
|---|---|---|
| `hud:client:BuffEffect` | `{ buffName, display, iconName, iconColor, progressColor, progressValue }` | Buff adicionado, atualizado a cada tick ou removido (`display = false`) |
| `hud:client:EnhancementEffect` | `{ enhancementName, display, iconColor }` | Enhancement adicionado ou removido (`display = false`) |

O `progressValue` vai de 0 a 100 e é calculado como `tempoRestante * 100 / maxTime`.

---

## Estrutura de arquivos

```
ps-buffs/
├── client/
│   └── main.lua        — exports do client, efeitos prontos (stamina, natação, vida, colete, estresse)
├── server/
│   └── main.lua        — tabela de buffs por citizenid, callbacks, loop de decremento, eventos de HUD
├── shared/
│   └── config.lua      — TickTime e catálogo Config.Buffs
├── snippets            — trechos para substituir no qb-core (fome/sede com super-hunger e super-thirst)
└── fxmanifest.lua
```
