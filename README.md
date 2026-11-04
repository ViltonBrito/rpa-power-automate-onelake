# 🚀 Pipeline de Automação e Ingestão de Relatórios (Power Automate & Microsoft Fabric)

## 📌 Visão Geral e Contexto de Negócio
Este projeto nasceu para resolver um gargalo operacional crítico no setor cooperativo: a extração e o processamento manual de relatórios volumosos a partir de sistemas analíticos internos (SOS e Sisbr Analítico). 

Devido à ausência de uma API direta para integração nativa e ao alto tempo de espera no carregamento manual, a rotina exigia esforço humano repetitivo da equipe. 

**A Solução:** Em conjunto com a equipe, estruturamos agendamentos prévios na plataforma para geração de links diretos de download e criei uma lista centralizada no **SharePoint** para atuar como painel de controle e fila de execução do pipeline. A partir disso, desenvolvi este fluxo automatizado end-to-end que realiza o download em segundo plano, aplica validações rigorosas de integridade e ingere os dados diretamente no **Lakehouse do Microsoft Fabric (OneLake)**.

---

## 🏗️ Arquitetura do Fluxo (Versão 2.0)
O diagrama abaixo representa toda a lógica de decisão, resiliência e integração entre o **Power Automate Desktop (PAD)** e a nuvem (**Power Automate Cloud + Microsoft Fabric**):

```mermaid
graph TD
    %% --- PRÉ-REQUISITOS E INÍCIO ---
    Pre1[Pré-requisito: Pasta de Downloads Vazia] -.-> A
    Pre2[Pré-requisito: Edge Logado no Sistema Analítico] -.-> A
    A[Início do Fluxo Desktop PAD] --> B[Ler Lista do SharePoint - GetItems]
    B --> C1{Modo Prioridade Ativo?}

    C1 -- SIM --> C2{Loop: Procura item com EstaemUso = EXECUTAR}
    C1 -- NAO --> C3{Loop: Para cada Relatório da Fila}

    C2 -- Encontrou Item Prioritario --> D
    C2 -- Fim da Lista --> Z([Fim do Processamento])

    C3 -- Fim da Fila --> Z
    C3 -- Próximo Item --> D{Validação 1: Está em Uso?}

    %% --- VALIDAÇÕES DE ESTADO E PRAZO ---
    D -- NAO --> D2[Status: Não Esta Em Uso]
    D2 --> U[Atualizar Item no SharePoint - PatchItem]

    D -- SIM ou EXECUTAR --> E{Validação 2: É Mensal e Fora da 2ª Quarta?}
    E -- Fora do Prazo (Ignora) --> C3
    E -- No Prazo ou Diário --> F[Mapear Arquivos Locais Existentes na Pasta]

    %% --- DOWNLOAD E MONITORAMENTO ---
    F --> G[Abrir Edge Minimizado na URL do Relatório]
    G --> H[Aguardar e Monitorar Arquivos na Pasta]

    H --> I{Validação 3: Timeout > 12 Tentativas?}
    I -- Sim: Falha no Tempo --> I2[Status: Relatório Com Erro]
    I2 --> I3[Fechar Edge - Erro]
    I3 --> U

    I -- Não: Dentro do Tempo --> J{Validação 4: Arquivo Novo e Sem .crdownload?}
    J -- Ainda Baixando --> H
    J -- Download Concluído --> K[Fechar Edge - Sucesso]

    %% --- PÓS-DOWNLOAD E LIMPEZA DE NOME ---
    K --> L{Validação 5: Tipo de Salvamento = Substituição?}
    L -- SIM --> M[Renomear Arquivo Local com Título do SharePoint]
    M --> N
    L -- NAO --> N{Validação 6: Tamanho do Arquivo <= 6KB?}

    N -- Arquivo Inválido / Vazio --> N2[Status: Arquivo Vazio]
    N2 --> U

    N -- Arquivo Válido --> O[Limpar Caracteres Especiais do Nome e Sanitizar Acentos]
    O --> P[Converter Arquivo para formato Base64]
    P --> Q[Disparar HTTP POST para a Nuvem]

    %% --- INTEGRAÇÃO NUVEM E FABRIC ---
    Q -. "HTTP POST (Payload Base64 + Headers)" .-> Cloud1

    subgraph Nuvem ["Power Automate Cloud & Microsoft Fabric"]
        Cloud1([Gatilho HTTP: Recebe JSON]) --> Cloud2[Ação: Converter base64ToBinary]
        Cloud2 --> Cloud3[Ação HTTP: Método PUT no OneLake]
        Cloud3 --> CloudAuth{Autenticação: Service Account}
        CloudAuth -- Requer permissao de Membro --> Cloud4[(Fabric OneLake / Lakehouse)]
    end

    Q --> R[Status: Concluído Com Êxito]
    R --> S[Excluir Arquivo Local da Pasta de Downloads]
    S --> U

    U --> C3

    %% --- ESTILOS VISUAIS ---
    style A fill:#4CAF50,stroke:#388E3C,stroke-width:2px,color:#fff
    style Z fill:#4CAF50,stroke:#388E3C,stroke-width:2px,color:#fff
    style U fill:#2196F3,stroke:#1976D2,stroke-width:2px,color:#fff
    style Cloud4 fill:#FF9800,stroke:#F57C00,stroke-width:2px,color:#fff
    style Nuvem fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style CloudAuth fill:#fff3e0,stroke:#e65100,stroke-width:2px
```

> 🔒 **Nota de Segurança e Governança (InfoSec):** Por boas práticas de segurança da informação e conformidade corporativa, o script bruto da automação (que contém endpoints privados, credenciais de contas de serviço e parâmetros internos) não é disponibilizado publicamente. A documentação foca no desenho de arquitetura, engenharia de fluxo e regras de negócio.

---

## ⚙️ Principais Regras de Negócio e Tratamento de Erros
1. **Orquestração via SharePoint:** Uma lista controla quais relatórios estão ativos, a periodicidade e o status em tempo real de cada execução.
2. **Modo Prioridade (`EXECUTAR`):** Permite furar a fila padrão caso um analista precise rodar um relatório específico com urgência.
3. **Camadas de Resiliência:**
   * **Controle de Janela Temporal:** Valida se relatórios mensais estão no dia correto de execução (ex: 2ª quarta-feira do mês).
   * **Prevenção de Loop Infinito (Timeout):** Monitora a pasta de downloads com limite de 12 tentativas e verifica extensões temporárias (`.crdownload`) antes de prosseguir.
   * **Validação de Integridade (<= 6KB):** Descarta arquivos vazios ou corrompidos gerados por falhas na origem.
4. **Sanitização e Ingestão Moderna (Lakehouse):** Limpa caracteres especiais, converte o arquivo para `Base64`, envia via `HTTP POST` para o Power Automate Cloud, converte em binário (`base64ToBinary`) e grava via `HTTP PUT` diretamente no **Microsoft Fabric (OneLake)**.

---

## 🛠️ Tecnologias Utilizadas
* **RPA & Orquestração:** Power Automate Desktop (PAD) & Power Automate Cloud
* **Arquitetura de Dados:** Microsoft Fabric (OneLake / Lakehouse)
* **Controle de Fila & Logs:** SharePoint Lists
* **Integração & Protocolos:** APIs REST (HTTP POST / PUT), JSON, Codificação Base64

---

## 📈 Evolução do Projeto (Changelog & Roadmap)

* **Versão 1.0 (MVP):** Automação inicial focada apenas no download local via links pré-agendados para eliminar a espera manual da equipe.
* **Versão 2.0 (Atual):** Inclusão da fila de controle no SharePoint, modo de prioridade, 6 camadas de validação de erros (timeout, tamanho mínimo, sanitização de nomes) e ingestão direta no Lakehouse do Microsoft Fabric.
* **Versão 3.0 (Roadmap Futuro):**
  * 📧 **Expansão de Fontes:** Automatizar relatórios que atualmente chegam via e-mail, permitindo que a equipe escolha salvar o destino final tanto no SharePoint quanto no Microsoft Fabric.
  * ⚡ **Execução On-Demand Assistida (Runtime):** Implementar um gatilho interativo onde o colaborador solicita a subida de um relatório específico, o fluxo valida a abertura do sistema na máquina, solicita aprovação rápida e executa a ingestão na hora.

---

## 👨‍💻 Autor
Desenvolvido por **Vilton Brito**  
*Data Analyst & Automation Specialist*  
- 💼 [LinkedIn](https://www.linkedin.com/in/vilton-brito-a2128520b/)  
- 📧 [viltonprjunior@gmail.com](mailto:viltonprjunior@gmail.com)
