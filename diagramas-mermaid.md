# Compilado de diagramas — CuidarAI

Este documento reúne os diagramas Mermaid encontrados em [`arquitetura-app-ia.md`](./arquitetura-app-ia.md). Cada diagrama permanece em um bloco independente para facilitar sua revisão e futura exportação para SVG.

## Índice

1. Fluxo principal de uso
2. Camadas da aplicação
3. Arquitetura funcional da sessão
4. Detecção da wake word
5. Implementações do `AiGateway`
6. Limite de responsabilidade da IA
7. Validação da administração
8. Divergência de identidade do paciente
9. Divergência de medicamento
10. Máquina de estados da administração
11. Coordenação da sessão
12. Tratamento de dados faciais
13. Destinos dos eventos de domínio
14. Fluxo de ações da apresentação
15. Fluxo de estado da apresentação
16. Ports & Adapters
17. Topologia de execução — alternativa A
18. Topologia de execução — alternativa B
19. Topologia de execução — alternativa C
20. Estratégia inicial de prototipação
21. Sequência dos próximos passos
22. Resumo da arquitetura
23. Casos de uso do sistema

---

## 1. Fluxo principal de uso

Origem: seção 2, linha 27.

```mermaid
flowchart TD
    A[Cuidador] -->|"Hey CuidarAI"| B[Ativação da sessão]
    B --> C[Captura de áudio]
    B --> D[Captura de vídeo]
    C --> E[IA / Percepção]
    D --> E
    E --> F[Fala: medicamento, dose e paciente]
    E --> G[Visão: medicamento observado]
    E --> H[Face: correspondência com paciente esperado]
    F --> I[Evidências da sessão]
    G --> I
    H --> I
    I --> J[Consulta ao cadastro e à prescrição]
    J --> K[Validação determinística]
    K -->|Válido| L[Registrar administração]
    K -->|Divergente ou inconclusivo| M[Registrar tentativa / evento]
    L --> N[Histórico de doses]
```

## 2. Camadas da aplicação

Origem: seção 5, linha 109.

```mermaid
flowchart TD
    P["Presentation<br/>Jetpack Compose · ViewModel · UiState · Navigation"]
    A["Application<br/>Use Cases · Session Coordinator · Orchestration"]
    D["Domain<br/>Entities · Validation · FSM · Policies · Events"]
    I["Infrastructure<br/>Glasses · IA · Storage · Backend · Android APIs"]

    P --> A
    A --> D
    I -->|implementa Ports / Interfaces definidos para o núcleo| D
```

## 3. Arquitetura funcional da sessão

Origem: seção 6, linha 129.

```mermaid
flowchart TD
    C[Cuidador] -->|"Hey CuidarAI"| W[WakeWordDetector]
    W --> S[AdministrationSessionCoordinator]

    S --> CD[CaptureDevice<br/>áudio + vídeo]
    S --> AI[AiGateway<br/>voz · visão · face]
    CD --> AI
    AI --> E[AdministrationEvidence]

    E --> PR[PatientRepository]
    E --> RX[PrescriptionRepository]
    PR --> V[AdministrationValidator]
    RX --> V
    E --> V

    V --> FSM[AdministrationStateMachine]
    FSM -->|válido| OK[CONFIRMED]
    FSM -->|divergente| DG[DIVERGENT]
    FSM -->|evidência insuficiente| UN[UNCERTAIN]

    OK --> EV[Domain Events]
    DG --> EV
    UN --> EV
    EV --> AR[AdministrationRepository]
    AR --> HB[Histórico / Backend]
```

## 4. Detecção da wake word

Origem: seção 8, linha 205.

```mermaid
flowchart LR
    A[Áudio] --> B{Wake word detectada?}
    B -->|"Hey CuidarAI"| C[WakeWordDetected]
```

## 5. Implementações do `AiGateway`

Origem: seção 10, linha 273.

```mermaid
flowchart LR
    G[AiGateway] --> L[LocalAiAdapter]
    G --> C[CloudAiAdapter]
    G --> S[SeparateAppAiAdapter]
    G --> H[HybridAiAdapter]
    G --> F[FakeAiAdapter]
```

## 6. Limite de responsabilidade da IA

Origem: seção 11, linha 316.

```mermaid
flowchart TD
    AI[IA] --> O[Observações]
    O --> D["Domain<br/>paciente informado<br/>paciente reconhecido<br/>medicamento informado<br/>medicamento observado<br/>dosagem informada<br/>prescrição ativa<br/>dose prevista<br/>horário / janela aplicável"]
    D --> R[Decisão determinística]
```

## 7. Validação da administração

Origem: seção 14, linha 386.

```mermaid
flowchart TD
    E[AdministrationEvidence] --> V[AdministrationValidator]
    P[Patient] --> V
    R[Prescription] --> V
    V -->|regras satisfeitas| OK[Valid]
    V -->|inconsistência| DG[Divergent]
    V -->|evidência insuficiente| UN[Uncertain]
```

## 8. Divergência de identidade do paciente

Origem: seção 15, linha 416.

```mermaid
flowchart TD
    V["Voz: Paciente João"] --> S[spokenPatient = João]
    F[Face observada] --> R[facePatient = Carlos]
    S --> M{Identidades correspondem?}
    R --> M
    M -->|Não| E[PatientIdentityMismatch]
```

## 9. Divergência de medicamento

Origem: seção 15, linha 427.

```mermaid
flowchart TD
    V["Voz: Losartana"] --> M{Medicamentos correspondem?}
    I["Visão: Metformina"] --> M
    M -->|Não| E[MedicationMismatch]
```

## 10. Máquina de estados da administração

Origem: seção 16, linha 444.

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> STARTING: WakeWordDetected
    STARTING --> CAPTURING
    CAPTURING --> COLLECTING_EVIDENCE
    COLLECTING_EVIDENCE --> VALIDATING

    VALIDATING --> UNCERTAIN: evidência insuficiente
    VALIDATING --> DIVERGENT: divergência
    VALIDATING --> CONFIRMED: regras satisfeitas

    CONFIRMED --> PERSISTING
    PERSISTING --> COMPLETED

    COLLECTING_EVIDENCE --> CANCELLED: cancelamento
    COLLECTING_EVIDENCE --> TIMEOUT: timeout
    PERSISTING --> FAILED: falha de persistência

    COMPLETED --> [*]
    CANCELLED --> [*]
    TIMEOUT --> [*]
    FAILED --> [*]
```

## 11. Coordenação da sessão

Origem: seção 17, linha 499.

```mermaid
flowchart TD
    C[CaptureDevice] --> AI[AiGateway]
    AI --> E[AdministrationEvidence]
    E --> U[Application Use Cases]
    U --> V[AdministrationValidator]
    V --> F[AdministrationStateMachine]
    F --> R[Repository]
    F --> D[Domain Events]
    F --> S[Estado da sessão]
    S --> VM[ViewModel / UiState]
```

## 12. Tratamento de dados faciais

Origem: seção 21, linha 660.

```mermaid
flowchart TD
    F[Frame facial] --> V[Verificação / embedding]
    V --> R[Resultado estruturado]
    R --> P[Persistência conforme política]
    V --> D[Descarte do frame após processamento]
```

## 13. Destinos dos eventos de domínio

Origem: seção 22, linha 697.

```mermaid
flowchart LR
    E[Domain Event] --> A[Audit Log]
    E --> U[UI Update]
    E --> S[Sync / Backend]
```

## 14. Fluxo de ações da apresentação

Origem: seção 23, linha 712.

```mermaid
flowchart TD
    U[User Action] --> C[Compose Screen]
    C --> V[ViewModel]
    V --> A[Use Case / Coordinator]
    A --> D[Application / Domain]
```

## 15. Fluxo de estado da apresentação

Origem: seção 23, linha 722.

```mermaid
flowchart TD
    D[Application / Domain] --> V[ViewModel]
    V --> S[UiState]
    S --> C[Compose]
```

## 16. Ports & Adapters

Origem: seção 26, linha 831.

```mermaid
flowchart TD
    DA[Domain / Application] -->|define| P[Ports / Interfaces]
    I[Infrastructure] -->|implementa| P
    I --> M[Meta SDK]
    I --> L[IA local]
    I --> C[IA cloud]
    I --> B[Banco]
    I --> BE[Backend]
```

## 17. Topologia de execução — alternativa A

Origem: seção 27, linha 852.

```mermaid
flowchart LR
    G[Óculos] --> A[Android App]
    A --> I[IA local]
    A --> B[Backend]
```

## 18. Topologia de execução — alternativa B

Origem: seção 27, linha 861.

```mermaid
flowchart LR
    G[Óculos] --> A[Android App]
    A --> L[APK / serviço local de IA]
    A --> C[Cloud AI]
    A --> B[Backend]
```

## 19. Topologia de execução — alternativa C

Origem: seção 27, linha 871.

```mermaid
flowchart LR
    A[Android App] --> S[Speech local]
    A --> F[Face local]
    A --> M[Medication AI cloud]
    A --> B[Backend de dados]
```

## 20. Estratégia inicial de prototipação

Origem: seção 30, linha 1004.

```mermaid
flowchart TD
    A["Simular 'Hey CuidarAI'"] --> B[Iniciar sessão]
    B --> C[Injetar áudio e frames gravados]
    C --> D[Fake AI produz observações]
    D --> E[Montar AdministrationEvidence]
    E --> F[Consultar paciente / prescrição fake]
    F --> G[AdministrationValidator]
    G --> H[AdministrationStateMachine]
    H --> I[Resultado]
    I --> J[Persistência em memória]
    J --> K[Histórico na UI]
```

## 21. Sequência dos próximos passos

Origem: seção 34, linha 1142.

```mermaid
flowchart TD
    S1[1. Refinar AdministrationSession e AdministrationEvidence]
    S2[2. Definir contrato inicial do AiGateway]
    S3[3. Definir CaptureDevice e WakeWordDetector]
    S4[4. Modelar AdministrationValidator]
    S5[5. Modelar AdministrationStateMachine]
    S6[6. Definir Patient / Prescription / Medication / Dosage]
    S7[7. Definir eventos de domínio]
    S8[8. Criar adapters fake]
    S9[9. Implementar AdministrationSessionCoordinator]
    S10[10. Criar ViewModel + UiState]
    S11[11. Criar tela Compose do fluxo]
    S12[12. Criar histórico local fake]
    S13[13. Substituir adapters por implementações reais]

    S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8 --> S9 --> S10 --> S11 --> S12 --> S13
```

## 22. Resumo da arquitetura

Origem: seção 35, linha 1167.

```mermaid
flowchart TD
    P[Presentation] --> A[Application]
    A --> D[Domain]
    DA[Device Adapter] --> C[Ports / Contracts]
    AI[AI Adapter] --> C
    DT[Data Adapter] --> C
    C --> D
```

## 23. Casos de uso do sistema

Origem: rascunho de caso de uso fornecido em 22/08/2026.

```mermaid
flowchart LR
    RESPONSAVEL["👤 Responsável"]
    CUIDADOR["👤 Cuidador"]

    subgraph SISTEMA["Sistema CuidarAI"]
        direction TB

        subgraph GESTAO["Cadastro e acompanhamento"]
            direction TB
            UC_CAD_RES([Cadastrar residente])
            UC_CAD_PRESC([Cadastrar prescrição])
            UC_HIST([Consultar histórico])
            UC_ALERTA_OMISSA([Receber alerta de dose omitida])
            UC_ALERTA_URG([Receber alerta de urgência])
        end

        subgraph CUIDADO["Administração e assistência"]
            direction TB
            UC_LEMBRETE([Receber lembrete de horário])
            UC_ADMIN([Administrar medicamento])
            UC_WAKE([Iniciar sessão por wake word])
            UC_IDENT([Identificar residente])
            UC_VERIF([Verificar medicamento e dose])
            UC_VOZ([Confirmar administração por voz])
            UC_URG([Reportar urgência])
        end

        subgraph APOIO["Comportamentos de apoio"]
            direction TB
            UC_CONS_PRESC([Consultar prescrição ativa])
            UC_MONITORAR([Monitorar janela de administração])
            UC_REGISTRAR([Registrar administração])
        end
    end

    RESPONSAVEL --- UC_CAD_RES
    RESPONSAVEL --- UC_CAD_PRESC
    RESPONSAVEL --- UC_HIST
    RESPONSAVEL --- UC_ALERTA_OMISSA
    RESPONSAVEL --- UC_ALERTA_URG

    CUIDADOR --- UC_LEMBRETE
    CUIDADOR --- UC_ADMIN
    CUIDADOR --- UC_URG

    UC_CAD_PRESC -. "«include»" .-> UC_CONS_PRESC
    UC_LEMBRETE -. "«include»" .-> UC_CONS_PRESC
    UC_ADMIN -. "«include»" .-> UC_IDENT
    UC_ADMIN -. "«include»" .-> UC_VERIF
    UC_ADMIN -. "«include»" .-> UC_CONS_PRESC
    UC_ADMIN -. "«include»" .-> UC_VOZ
    UC_ADMIN -. "«include»" .-> UC_REGISTRAR
    UC_ADMIN -. "«include»" .-> UC_WAKE
    UC_MONITORAR -. "«include»" .-> UC_CONS_PRESC
    UC_ALERTA_OMISSA -. "«extend»" .-> UC_MONITORAR
    UC_URG -. "«include»" .-> UC_ALERTA_URG

    classDef actor fill:#f4f1ea,stroke:#666,color:#444,stroke-width:1px;
    classDef management fill:#eeedff,stroke:#5651b5,color:#403ca0;
    classDef care fill:#e1f5ef,stroke:#15806d,color:#126a5b;
    classDef support fill:#fff8df,stroke:#a77a16,color:#70520f;
    class RESPONSAVEL,CUIDADOR actor;
    class UC_CAD_RES,UC_CAD_PRESC,UC_HIST,UC_ALERTA_OMISSA,UC_ALERTA_URG management;
    class UC_LEMBRETE,UC_ADMIN,UC_WAKE,UC_IDENT,UC_VERIF,UC_VOZ,UC_URG care;
    class UC_CONS_PRESC,UC_MONITORAR,UC_REGISTRAR support;
```

### Justificativa das relações

| Origem | Relação | Destino | Motivo |
|---|---|---|---|
| Cadastrar prescrição | `«include»` | Consultar prescrição ativa | O sistema precisa consultar o estado atual para criar ou alterar uma prescrição sem gerar sobreposição indevida. O residente já cadastrado é tratado como pré-condição, não como `include`, pois não é recadastrado em toda operação. |
| Receber lembrete de horário | `«include»` | Consultar prescrição ativa | O horário e a dose do lembrete vêm necessariamente de uma prescrição vigente. |
| Administrar medicamento | `«include»` | Identificar residente | A identificação é obrigatória para vincular a dose à pessoa correta. |
| Administrar medicamento | `«include»` | Verificar medicamento e dose | A conferência é obrigatória antes da confirmação da administração. |
| Administrar medicamento | `«include»` | Consultar prescrição ativa | Medicamento, dose e janela de horário precisam ser comparados com a prescrição vigente. |
| Administrar medicamento | `«include»` | Confirmar administração por voz | No fluxo desenhado, a confirmação falada é uma etapa obrigatória da administração. |
| Administrar medicamento | `«include»` | Registrar administração | Uma administração concluída precisa gerar histórico e evidência de auditoria. |
| Administrar medicamento | `«include»` | Iniciar sessão por wake word | Toda administração começa obrigatoriamente nos óculos por meio da wake word. Não existe um fluxo equivalente iniciado pela interface do aplicativo. |
| Monitorar janela de administração | `«include»` | Consultar prescrição ativa | O monitoramento depende dos horários definidos na prescrição. |
| Receber alerta de dose omitida | `«extend»` | Monitorar janela de administração | O alerta só acontece sob a condição de a janela terminar sem uma administração confirmada. |
| Reportar urgência | `«include»` | Receber alerta de urgência | Todo reporte aceito deve notificar o responsável; por isso, o envio/recebimento do alerta faz parte obrigatória do fluxo. |

### Observações de modelagem

- As linhas contínuas representam associações entre atores e casos de uso; não indicam ordem de execução.
- `«include»` representa comportamento obrigatório e reutilizado pelo caso de uso de origem.
- `«extend»` representa comportamento condicional ou opcional apontando para o caso de uso base.
- Os casos “Administrar medicamento”, “Consultar prescrição ativa”, “Monitorar janela de administração” e “Registrar administração” foram acrescentados para explicitar o objetivo principal e evitar dependências ambíguas entre etapas isoladas.
- A interação de administração é orientada pelos óculos. A aplicação oferece apoio e acompanhamento, mas não inicia pela interface o fluxo de administração de medicamento.
- “Alerta: dose omissa” e “Alerta de urgência” foram renomeados como ações observáveis pelo ator: “Receber alerta de dose omitida” e “Receber alerta de urgência”.
- “Cadastrar residente” é pré-condição de “Cadastrar prescrição”. Não foi usado `«include»`, pois `include` significaria executar o cadastro do residente sempre que uma prescrição fosse cadastrada.
