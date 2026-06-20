# Arquitetura de E-commerce 24/7 na Microsoft Azure

Proposta de arquitetura em nuvem para o Desafio Final do Bootcamp de Arquitetura de Soluções. O cenário é uma empresa de vendas online cujo sistema precisa ficar no ar 24 horas por dia, resistir a falhas de infraestrutura e absorver picos de acesso sem degradar. Este repositório reúne o diagrama da solução na Azure e a documentação das decisões por trás dele.

## Provedor escolhido

A solução foi desenhada na **Microsoft Azure**. A escolha acompanha o tipo de aplicação — um e-commerce sobre stack .NET com SQL Server — e o vocabulário do próprio enunciado (PaaS, IAM, zonas de disponibilidade), que se traduz diretamente nos serviços gerenciados da plataforma. A mesma topologia poderia ser construída em AWS ou GCP; o que muda são os nomes dos serviços, não o raciocínio.

## Diagrama

![Diagrama da arquitetura Azure](arquitetura-azure-bootcamp.svg)

Arquivo editável: [`arquitetura-azure-bootcamp.drawio`](arquitetura-azure-bootcamp.drawio), que abre no [draw.io](https://app.diagrams.net).

## Visão geral

A arquitetura segue o caminho de uma requisição e separa as responsabilidades em camadas. O usuário chega pela internet e cai primeiro no **Azure Traffic Manager**, que decide, no nível de DNS, para qual região enviar o tráfego. Dentro da região, o **Application Gateway v2 com WAF** recebe a requisição, filtra ataques de camada web e distribui a carga entre as instâncias da aplicação. Essas instâncias vivem num **VM Scale Set Linux**, distribuído por três zonas de disponibilidade e com escala automática. O acesso ao banco não passa pela internet: a aplicação fala com o **Azure SQL Database** por um **Private Endpoint** dentro da rede virtual. Em volta disso, **Microsoft Entra ID**, **Azure Key Vault** e o conjunto **Azure Monitor / Log Analytics / Application Insights** cuidam, respectivamente, de identidade, segredos e observabilidade. Uma segunda região sustenta a estratégia de recuperação de desastres.

## Fluxo de uma requisição

O usuário acessa por HTTPS. O Traffic Manager resolve o nome para a região saudável e o tráfego entra pelo Application Gateway, que aplica o WAF e encaminha para uma das instâncias do VM Scale Set. A instância processa a requisição e, quando precisa de dados, conecta ao Azure SQL Database através do Private Endpoint — uma rota privada, sem exposição pública do banco. Durante todo esse percurso, métricas e logs do gateway, das VMs e do banco seguem para o Azure Monitor.

## Alta disponibilidade

A disponibilidade é tratada em mais de um nível, porque as falhas também acontecem em níveis diferentes. As instâncias da aplicação ficam espalhadas pelas zonas de disponibilidade 1, 2 e 3: se um datacenter inteiro cai, apenas aquela zona é afetada e o sistema continua respondendo pelas outras duas. O Application Gateway e o Azure SQL Database são zona-redundantes pelo mesmo motivo, para que nenhum deles seja um ponto único de falha. Acima da região, o Traffic Manager garante que a perda da região inteira não derrube o serviço, redirecionando para a secundária.

O enunciado pede "alta disponibilidade" e "recuperação de desastres" quase no mesmo fôlego, mas são respostas a falhas diferentes. As múltiplas zonas são alta disponibilidade: protegem contra a perda de um datacenter dentro de uma região. A segunda região é recuperação de desastres: protege contra a perda da região inteira. As zonas protegem contra perder um prédio; a segunda região, contra perder uma cidade.

## Escalabilidade

A camada de aplicação roda em um VM Scale Set com imagem Linux (Ubuntu), e não em máquinas fixas. O autoscale parte de **3 instâncias** e cresce até **6** conforme a demanda — tipicamente acompanhando a CPU, mas a regra também pode considerar memória ou volume de requisições. Na prática, são três máquinas dando conta de uma madrugada tranquila e seis absorvendo o pico de uma promoção, encolhendo de volta quando o movimento passa. Paga-se pela capacidade que a demanda exigiu, não pelo pior cenário o tempo todo. Os limites de 3 e 6 são os definidos no enunciado.

## Banco de dados (PaaS)

O banco é o **Azure SQL Database**, um serviço gerenciado (PaaS). Optar por PaaS tira do time a responsabilidade de cuidar de patch, backup, atualização e alta disponibilidade do servidor — tudo isso fica a cargo da plataforma. O banco é zona-redundante, mantém backups automáticos com restauração para um ponto no tempo (Point-in-Time Restore) e usa criptografia em repouso.

Um ponto de representação que costuma gerar dúvida: o Azure SQL não fica *dentro* da rede virtual, porque é um serviço PaaS. O acesso privado acontece por um Private Endpoint colocado dentro da VNet, que dá ao banco um endereço privado e tira o endpoint público do caminho. Do ponto de vista da aplicação, o banco está na rede; do ponto de vista da internet, ele não existe.

## Segurança e controle de acesso (IAM)

A segurança está distribuída em algumas frentes, e a parte de identidade é a que mais costuma ser desenhada errada. O enunciado pede que as VMs tenham permissão de leitura e escrita no banco. Na Azure isso não é usuário e senha numa string de conexão. Cada VM recebe uma **Managed Identity**, o **Microsoft Entra ID** responde por essa identidade, e a ela são concedidas as permissões no banco (`db_datareader` e `db_datawriter`). A string de conexão não carrega segredo nenhum: a VM comprova quem é e o banco confia nela por causa disso. É o equivalente, na nuvem, à autenticação integrada — a credencial vem da identidade de quem pede, não de um arquivo de configuração.

Os segredos que ainda precisam existir ficam no **Azure Key Vault**, acessado também por Managed Identity, o que reduz credencial fixa espalhada pela aplicação. Na rede, a VNet é segmentada em sub-redes por função (aplicação e dados), com **Network Security Groups** controlando o tráfego entre elas, e o acesso ao banco restrito ao Private Endpoint.

## Monitoramento e observabilidade

A telemetria é centralizada no **Azure Monitor**, com o **Log Analytics** concentrando os logs e o **Application Insights** acompanhando a aplicação. São observados o Application Gateway, as VMs e o banco. Isso não é enfeite: num e-commerce, um checkout que começa a demorar três segundos é dinheiro vazando, e sem métrica e log você fica adivinhando onde está o problema em vez de apontar a causa. Alertas que fazem sentido nesse contexto incluem CPU alta nas VMs, falha de health probe no gateway, aumento de erros HTTP 5xx, latência da aplicação e falhas de conexão com o banco.

## Recuperação de desastres

A estratégia de DR usa uma região secundária. A primária é a **Azure Brazil South** e a secundária, a **Azure Brazil Southeast**. O banco mantém uma geo-réplica na região secundária, coordenada por um **Auto-Failover Group**, além dos backups automáticos. Se a região primária sofre uma falha crítica, o Failover Group promove a réplica e a aplicação passa a operar pela secundária, com perda mínima de dados e sem depender de uma intervenção manual demorada.

## Decisões e trade-offs

Duas escolhas merecem justificativa, porque havia alternativas defensáveis.

Para o failover entre regiões, optei pelo **Traffic Manager** (DNS) em vez do **Azure Front Door**. O Front Door agregaria WAF global, cache de borda e roteamento Anycast, ao custo de mais complexidade e preço. Para o escopo deste desafio, o Traffic Manager cuidando do redirecionamento de região, somado ao Application Gateway com WAF na entrada da região, atende ao requisito sem peso desnecessário. Num cenário com tráfego global de verdade, o Front Door passaria a valer a pena.

No acesso ao banco, em vez de só liberar o endpoint público por regras de firewall de IP, usei **Private Endpoint**. A regra de IP atenderia ao texto do enunciado ("firewall para origens autorizadas"), mas deixaria o banco com endereço público exposto. O Private Endpoint tira o banco da internet por completo, que é a postura mais segura e mais fácil de defender numa avaliação.

## Requisitos do desafio atendidos

| Requisito do enunciado | Como foi atendido |
|---|---|
| Múltiplas zonas de disponibilidade | VM Scale Set distribuído nas zonas 1, 2 e 3; Application Gateway e Azure SQL zona-redundantes |
| Balanceamento de carga entre as VMs | Application Gateway v2 (camada 7) com WAF |
| Escalonamento automático — mín. 3 / máx. 6, imagem Linux | VM Scale Set com Ubuntu e autoscale por CPU (3 → 6 instâncias) |
| Banco de dados gerenciado (PaaS) com HA e segurança | Azure SQL Database zona-redundante, acessível somente via Private Endpoint |
| IAM com leitura e escrita das VMs no banco | Managed Identity autenticada pelo Entra ID (`db_datareader` / `db_datawriter`) |
| Segurança de rede | VNet segmentada em sub-redes, NSGs e acesso privado ao banco |
| Monitoramento e logs | Azure Monitor, Log Analytics e Application Insights |
| Recuperação de desastres | Geo-réplica do Azure SQL, Auto-Failover Group e backups com PITR |

## Estrutura do repositório

```text
BootcampXP/
│
├── README.md
├── arquitetura-azure-bootcamp.svg     # diagrama renderizado (exibido acima)
└── arquitetura-azure-bootcamp.drawio  # diagrama editável (draw.io)
```

## Observação

A Atividade 1 do desafio — o desenho da arquitetura — é o único item obrigatório, e é o que está documentado aqui. O provisionamento real da infraestrutura (VMs, banco, regras de segurança) e a captura de telas são atividades opcionais do enunciado e não fazem parte desta entrega.
