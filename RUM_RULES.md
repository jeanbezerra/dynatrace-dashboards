# Dynatrace RUM — Regras para coleta

> Este Dashboard contabiliza somente dados RUM que chegaram ao Dynatrace. Utilize os filtros de **Frontend** e **Ambiente** antes de interpretar os indicadores.

## Condições obrigatórias

* O RUM deve estar habilitado para a aplicação e para o Process Group responsável pela entrega do HTML.
* O OneAgent precisa estar ativo no primeiro componente instrumentável próximo ao navegador.
* O HTML deve conter e executar o JavaScript do Dynatrace.
* O arquivo `ruxitagentjs_*` deve ser carregado com sucesso.
* Os beacons `/rb_*`, na injeção automática, ou `/bf`, no modo agentless, devem ser enviados sem bloqueios.
* Proxy, WAF, CDN, balanceador e firewall não podem remover, alterar ou reordenar os parâmetros do beacon.
* Cookies, cabeçalhos de correlação e requisições `POST` com conteúdo `text/plain` devem ser permitidos.
* A CSP deve autorizar o carregamento do script em `script-src` e o envio do beacon em `connect-src`.
* Para beacons cross-origin, a origem da aplicação deve estar autorizada no CORS e na Beacon Origin Allowlist.
* Quando o modo de consentimento estiver habilitado, a aplicação deve executar `dtrum.enable()` após o aceite do usuário.

## Separação dos ambientes

HML e PRD não devem ser diferenciados somente por tags compartilhadas.

Este Dashboard utiliza o domínio acessado pelo navegador:

```text
coalesce(view.url.domain, page.url.domain)
```

As regras de detecção de frontend devem preferencialmente separar os ambientes por domínio ou caminho:

```text
Aplicação – HML → dominio-hml.cliente.com.br
Aplicação – PRD → dominio.cliente.com.br
```

## Regras específicas para AngularJS

Para aplicações AngularJS 1.x:

* Ativar `Capture XmlHttpRequest (XHR)`.
* Ativar `Capture fetch() requests`, quando utilizado.
* Avaliar `Timed action support` para chamadas disparadas por `$timeout`, timers ou promises.
* Revisar as regras de exclusão de XHR.
* Utilizar a API `dtrum` para ações assíncronas que não sejam correlacionadas automaticamente.
* Confirmar que a navegação SPA e as alterações de rota estão produzindo eventos.

## Interpretação dos indicadores

| Indicador        | Resultado esperado                                      |
| ---------------- | ------------------------------------------------------- |
| `total_events`   | Deve existir quando houver utilização da aplicação      |
| `sessions`       | Sessões distintas que produziram eventos no período     |
| `actions`        | Carregamentos, ações XHR e ações reportadas             |
| `interactions`   | Cliques, toques e outras interações capturadas          |
| `navigations`    | Carregamentos e transições de páginas/views             |
| `request_events` | Requisições observadas pelo navegador                   |
| `csp_violations` | Violações CSP que conseguiram ser enviadas ao Dynatrace |

### Cenários de diagnóstico

| Comportamento                                     | Possível causa                                                     |
| ------------------------------------------------- | ------------------------------------------------------------------ |
| Todos os indicadores zerados                      | Sem utilização ou interrupção completa da coleta                   |
| Sessões e navegações presentes, mas ações zeradas | Captura XHR/SPA incompleta                                         |
| Ações presentes, mas requests zerados             | XHR/Fetch desabilitado, excluído ou não correlacionado             |
| Queda apenas em um ambiente                       | CSP, proxy, CDN, WAF ou configuração específica do ambiente        |
| Queda apenas em determinados navegadores          | Extensões, bloqueadores, política corporativa ou incompatibilidade |
| Violações CSP aumentando                          | `script-src` ou `connect-src` precisa ser revisado                 |

## Validação no navegador

No DevTools do navegador, validar:

1. Presença do script Dynatrace no HTML.
2. Carregamento de `ruxitagentjs_*` com resposta válida.
3. Envio de requisições `POST` para `/rb_*` ou `/bf`.
4. Ausência de erros CSP, CORS, `403`, `404`, `429` ou `503`.
5. Presença dos cookies Dynatrace, quando aplicáveis.
6. Ausência de bloqueio por proxy, EDR, WAF ou extensão do navegador.

> Um beacon bloqueado antes de chegar ao Dynatrace não aparece no DQL. Quando houver tráfego conhecido e todos os indicadores estiverem zerados, a validação deve continuar pelo navegador e pelos componentes de rede.

## Janela de avaliação

Evite concluir uma falha usando apenas o último intervalo. Considere a coleta degradada quando a ausência persistir por pelo menos **10 a 15 minutos** durante um período de utilização conhecida.

[Documentação oficial do Dynatrace RUM](https://docs.dynatrace.com/docs/observe/digital-experience/rum)
