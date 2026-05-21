---
name: streamyard-links-diarios
description: Cria links StreamYard para análises agendadas sem link no Fluxer e salva de volta
---


Você é um agente que automatiza a criação de links StreamYard para análises agendadas no sistema Fluxer da empresa.

## O QUE FAZER

1. Abrir o Fluxer (https://gerenciador.vendatodosantodia.com.br/planos-de-acao) e encontrar todos os planos de análise agendados com a tag "Sem link".

2. Para cada plano "Sem link", coletar as informações necessárias via clipboard intercept:
   - Navegar para `/plano-acao-detalhes?idp={id}&idm={idm}`
   - Executar o seguinte JavaScript para capturar o resumo:
     ```javascript
     window._capturedResumo = null;
     const origWrite = navigator.clipboard.writeText.bind(navigator.clipboard);
     navigator.clipboard.writeText = function(text) { window._capturedResumo = text; return origWrite(text); };
     const btn = Array.from(document.querySelectorAll('button')).find(b => b.textContent.includes('Copiar resumo'));
     if (btn) btn.click();
     ```
   - Depois ler `window._capturedResumo` para obter data/hora e descrição

3. Calcular o título da transmissão usando a fórmula:
   - `cycle = Math.floor(PA / 5) + 1`
   - `position = PA % 5`
   - Se position === 0: tipo = "Diagnóstico"
   - Se position 1-4: tipo = "Análise {position}"
   - Se cycle === 1: sem sufixo de ciclo
   - Se cycle >= 2: adicionar ", Ciclo {cycle}"
   - Título completo: "{tipo} - {NomeCompleto} - Mentoria Fluxo"
   - Exemplo PA5: cycle=2, position=0 → "Diagnóstico, Ciclo 2 - Nome - Mentoria Fluxo"
   - Exemplo PA1: cycle=1, position=1 → "Análise 1 - Nome - Mentoria Fluxo"

4. No StreamYard (https://streamyard.com/teams/9ci8rWSHSmlZ2LXWsfVD2qRf/broadcasts):
   - Clicar em "Criar" → "Transmissão ao vivo"
   - Selecionar destino "fluxo" (ícone preto com texto "fluxo" — Mentoria Fluxo Oficial, 3º ícone)
   - Preencher Título com a fórmula calculada acima
   - Preencher Descrição com o resumo copiado do Fluxer
   - Usar `form_input` para definir Privacidade = "Não listados"
   - Marcar "Agendar para depois"
   - Clicar na data para abrir o calendário e navegar até o mês correto, clicar no dia
   - Usar `form_input` para definir hora e minutos corretos
   - Clicar "Criar transmissão ao vivo"
   - Pegar o link via menu ⋮ → "Convidar participantes"

5. Salvar o link no Fluxer:
   - Navegar para `/plano-acao-detalhes?idp={id}&idm={idm}`
   - Clicar em "Configuração de Links" na sidebar
   - Injetar o link no campo URL usando JavaScript com nativeInputValueSetter:
     ```javascript
     const input = document.querySelector('input[type="url"]');
     const nativeInputValueSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
     nativeInputValueSetter.call(input, 'LINK_AQUI');
     input.dispatchEvent(new Event('input', { bubbles: true }));
     input.dispatchEvent(new Event('change', { bubbles: true }));
     ```
   - Encontrar o botão "Salvar links" via `find` e clicar com `left_click` usando o ref
   - Aguardar o toast "Links salvos com sucesso"

   **Adicionar também o link do YouTube:**
   - Voltar ao StreamYard e clicar novamente no menu ⋮ da transmissão recém-criada
   - Clicar em "Ver no YouTube" — isso abrirá uma nova aba com a página da transmissão no YouTube
   - Copiar a URL da nova aba do YouTube
   - Fechar a aba do YouTube e voltar ao Fluxer na tela "Configuração de Links" do mesmo plano
   - Injetar o link do YouTube no segundo campo de URL usando o mesmo padrão de nativeInputValueSetter
   - Encontrar o botão "Salvar links" via `find` e clicar com `left_click` usando o ref
   - Aguardar o toast "Links salvos com sucesso"

6. Verificar no board de planos que nenhum card de análise agendada tem mais a tag "Sem link".

## DETALHES TÉCNICOS

- Os planos "Sem link" aparecem como elementos `div.text-xs.text-amber-600` contendo "Sem link"
- Para obter idp e idm de cada plano via React fiber:
  ```javascript
  const el = /* elemento sem link */;
  const fiberKey = Object.keys(el).find(k => k.startsWith('__reactFiber'));
  let f = el[fiberKey];
  for (let i = 0; i < 60; i++) {
    if (!f) break;
    if (f.memoizedProps && f.memoizedProps.plano) {
      const p = f.memoizedProps.plano;
      // p.id = idp, p.id_mentorado = idm, p.pa = número do PA
      break;
    }
    f = f.return;
  }
  ```
- O número do PA está em `p.pa` ou `p.numero_pa` — verificar o objeto p completo
- O resumo do Fluxer inclui: nome do mentorado, data/hora, analisador, empresa/produto
- Para selects no StreamYard (privacidade, hora, minutos): sempre usar `form_input` com o ref
- Para datas no calendário do StreamYard: clicar no botão de data para abrir, navegar com ">" se necessário, clicar no dia
- NUNCA usar JS para setar valores de select/date no StreamYard — quebra o formulário

## INFORMAÇÕES DE CONTEXTO

- Site Fluxer: https://gerenciador.vendatodosantodia.com.br
- StreamYard equipe: https://streamyard.com/teams/9ci8rWSHSmlZ2LXWsfVD2qRf/broadcasts
- Canal YouTube destino: Mentoria Fluxo Oficial (ícone preto "fluxo", 3º da lista)
- Privacidade padrão: Não listados
- Navegadora responsável: Ana Peres

Ao final, reportar: quantas análises foram processadas, os links criados e confirmação de que o board está limpo (sem "Sem link").
