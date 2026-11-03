# SquadDesk Enterprise

**BackOffice de chamados, incidentes e solicitações** | Java 21 · Spring Boot 4.1.2 · Angular 22 · Angular Material · JPA/Hibernate · H2/Oracle · Docker

Projeto independente, desenvolvido para estudo, portfólio e prática dos conhecimentos pedidos na vaga de Programador Web Java Jr do Senac São Paulo. **Não é um sistema oficial do Senac.**

## 1. Visão geral

O SquadDesk centraliza solicitações de suporte e BackOffice. Um solicitante abre um chamado, um analista o assume, executa o atendimento, documenta a resolução e conclui o fluxo. Cada operação relevante registra um evento para rastreabilidade.

**Status da versão v0.2:** MVP local com API REST, entidades JPA, políticas de domínio, dashboard e interface Angular. **Ainda não há autenticação real**; os identificadores de operadores são fornecidos pelo cliente. **Não exponha esta versão na internet** e não use dados reais.

## 2. Funcionalidades implementadas

- Cadastro de departamentos e usuários (REQUESTER, ANALYST, ADMIN).
- Abertura e consulta de chamados de incidente, solicitação ou mudança.
- Priorização LOW, MEDIUM, HIGH e CRITICAL.
- Atribuição a analista, atendimento, resolução, reabertura e fechamento.
- Comentários, eventos de auditoria e indicadores por status.
- Validação de entrada, tratamento de erros HTTP e controle de concorrência otimista.
- Banco H2 local, com perfil opcional para Oracle Database.
- Frontend Angular Material para acompanhar os chamados.

**Não implementado nesta versão:** Spring Security/JWT, SOAP, Spring Batch, persistência MongoDB e integração com JSP/JSF/Struts/EJB. Estes itens pertencem ao roadmap.

## 3. Regras de negócio

| Código | Regra |
|---|---|
| **RN-01** | Código do departamento e e-mail de usuário são únicos. |
| **RN-02** | Usuário pertence a departamento ativo; solicitante deve ser REQUESTER ou ADMIN. |
| **RN-03** | Chamado exige título entre 5 e 150 caracteres, descrição entre 10 e 2000, categoria e prioridade. |
| **RN-04** | Todo chamado começa em **OPEN**, sem responsável. |
| **RN-05** | Apenas ANALYST ou ADMIN ativos podem ser responsáveis. ANALYST só assume para si; ADMIN pode atribuir a terceiros. |
| **RN-06** | O fluxo permitido é OPEN → IN_PROGRESS → RESOLVED → CLOSED. RESOLVED pode voltar para IN_PROGRESS. |
| **RN-07** | Para mudar o status é preciso haver responsável ativo; apenas o responsável ou ADMIN pode executar a ação. |
| **RN-08** | Resolução exige justificativa de 10 a 500 caracteres; reabertura exige 5 a 500. |
| **RN-09** | Chamados RESOLVED e CLOSED não podem ser atribuídos novamente. |
| **RN-10** | Somente solicitante, responsável ou ADMIN podem comentar; CLOSED é somente leitura. |
| **RN-11** | Criação, atribuição, mudança de status e comentário geram eventos de auditoria na mesma transação. |
| **RN-12** | Não há exclusão física de chamados pela API; listagem e dashboard permitem acompanhá-los. |
| **RN-13** | O campo de versão JPA previne atualização concorrente silenciosa do mesmo chamado. |

### Máquina de estados

~~~mermaid
stateDiagram-v2
    [*] --> OPEN: abertura
    OPEN --> IN_PROGRESS: atribuição e início
    IN_PROGRESS --> RESOLVED: resolução justificada
    RESOLVED --> IN_PROGRESS: reabertura justificada
    RESOLVED --> CLOSED: encerramento
    CLOSED --> [*]
~~~

> **Atenção:** os perfis acima são verificados pelas regras de domínio com base em actorId, informado pelo cliente. Até existir autenticação e autorização no servidor, **não constituem controle de acesso seguro**.

## 4. DER — Diagrama Entidade-Relacionamento

~~~mermaid
erDiagram
    SD_DEPARTMENT ||--o{ SD_USER : possui
    SD_USER ||--o{ SD_TICKET : solicita
    SD_USER |o--o{ SD_TICKET : atende
    SD_TICKET ||--o{ SD_TICKET_COMMENT : contem
    SD_USER ||--o{ SD_TICKET_COMMENT : escreve
    SD_TICKET ||--o{ SD_TICKET_EVENT : registra
    SD_USER ||--o{ SD_TICKET_EVENT : executa

    SD_DEPARTMENT {
        BIGINT id PK
        VARCHAR code UK
        VARCHAR name
        BOOLEAN active
    }
    SD_USER {
        BIGINT id PK
        BIGINT department_id FK
        VARCHAR name
        VARCHAR email UK
        VARCHAR role
        BOOLEAN active
    }
    SD_TICKET {
        BIGINT id PK
        BIGINT requester_id FK
        BIGINT assignee_id FK
        VARCHAR title
        VARCHAR description
        VARCHAR category
        VARCHAR priority
        VARCHAR status
        TIMESTAMP created_at
        TIMESTAMP updated_at
        TIMESTAMP resolved_at
        TIMESTAMP closed_at
        BIGINT version
    }
    SD_TICKET_COMMENT {
        BIGINT id PK
        BIGINT ticket_id FK
        BIGINT author_id FK
        VARCHAR body
        TIMESTAMP created_at
    }
    SD_TICKET_EVENT {
        BIGINT id PK
        BIGINT ticket_id FK
        BIGINT actor_id FK
        VARCHAR type
        VARCHAR previous_value
        VARCHAR new_value
        VARCHAR details
        TIMESTAMP created_at
    }
~~~

**Relacionamentos:** um departamento tem vários usuários; um usuário solicita vários chamados; um chamado pode ter zero ou um responsável; cada chamado possui comentários e eventos vinculados, com seus autores/atores. Os tipos SQL exatos variam entre H2 e Oracle conforme o mapeamento JPA.

## 5. Arquitetura

~~~text
Angular 22 + Angular Material
            |
            v
 REST Controllers / DTOs / Bean Validation
            |
            v
 Serviços de aplicação + TicketPolicy
            |
            v
 Spring Data JPA / Hibernate
            |
            v
     H2 (local) ou Oracle (Docker)
~~~

A auditoria da v0.2 é persistida relacionalmente em **SD_TICKET_EVENT**, e não no MongoDB.

## 6. Executar localmente no IntelliJ IDEA

**Requisitos:** JDK 21, Maven e IntelliJ IDEA. Para o frontend, Node.js compatível com Angular 22 e npm. Para Oracle, Docker Desktop.

1. Clone este repositório ou extraia o pacote do projeto.
2. Abra a pasta raiz no IntelliJ e configure o SDK Java 21.
3. Importe o projeto Maven e execute a classe **br.com.squaddesk.SquadDeskApplication** no módulo backend.
4. Confirme **http://localhost:8080/actuator/health** (usa H2 local por padrão).
5. Inicie o frontend:

~~~powershell
cd frontend
npm install
npm start
~~~

Abra **http://localhost:4200**. O proxy de desenvolvimento encaminha as chamadas /api ao backend na porta 8080.

### Oracle opcional

~~~powershell
Copy-Item .env.example .env
notepad .env
docker compose up -d oracle
~~~

Após o contêiner ficar saudável, configure as variáveis de ambiente **SPRING_PROFILES_ACTIVE=oracle**, **DB_USER** e **DB_PASSWORD** ao executar o backend. As credenciais locais do arquivo .env não devem ser publicadas.

> A criação/atualização automática do esquema JPA é **apenas para desenvolvimento**. Para ambientes reais, serão necessárias migrations versionadas (Flyway/Liquibase), revisão de índices e estratégia de backup.

## 7. API REST

| Método | Endpoint | Finalidade |
|---|---|---|
| GET / POST | /api/v1/departments | Listar e criar departamentos |
| GET / POST | /api/v1/users | Listar e criar usuários |
| GET / POST | /api/v1/tickets | Listar e criar chamados |
| GET | /api/v1/tickets/{id} | Obter chamado |
| PATCH | /api/v1/tickets/{id}/assignment | Atribuir responsável |
| PATCH | /api/v1/tickets/{id}/status | Alterar status |
| GET / POST | /api/v1/tickets/{id}/comments | Consultar e incluir comentários |
| GET | /api/v1/tickets/{id}/events | Consultar auditoria |
| GET | /api/v1/dashboard/summary | Resumo operacional |
| GET | /actuator/health | Monitoramento básico |

Exemplo de criação de chamado:

~~~json
{
  "title": "Erro no portal acadêmico",
  "description": "O portal apresenta erro ao abrir um formulário de atendimento.",
  "category": "INCIDENT",
  "priority": "HIGH",
  "requesterId": 1
}
~~~

Exemplo de resolução (requer atendimento em andamento e operador autorizado no fluxo demonstrativo):

~~~json
{
  "actorId": 2,
  "status": "RESOLVED",
  "note": "Erro corrigido, testado e validado com o solicitante."
}
~~~

Os IDs 1 e 2 acima são ilustrativos. Consulte **GET /api/v1/users** para obter os identificadores existentes.

## 8. Testes e qualidade

~~~powershell
cd backend
mvn test

cd ../frontend
npm install
npm run test:types
npm run build
~~~

O projeto contém testes das políticas de chamados e uma configuração de pipeline no diretório .github/workflows. **A compilação completa da aplicação e a execução integrada ainda precisam ser validadas em ambiente com acesso às dependências Maven e npm.**

## 9. Organização do repositório

~~~text
squaddesk-enterprise/
├── backend/             # API Java, domínio e testes
├── frontend/            # Angular + Material
├── docs/                # DER, regras, consultas Oracle e roteiro
├── .github/workflows/   # CI
├── docker-compose.yml   # Oracle para desenvolvimento
└── README.md
~~~

## 10. Roadmap

- [x] **v0.2:** regras de negócio, DER, workflow, auditoria relacional e interface inicial.
- [ ] **v0.3:** Spring Security, login, autorização real e Flyway.
- [ ] **v0.4:** testes de integração com banco de dados e melhorias de API.
- [ ] **v0.5:** integração SOAP e processamento Spring Batch.
- [ ] **v0.6:** auditoria complementar em MongoDB e observabilidade.
- [ ] **v1.0:** refinamentos Angular, documentação e pipeline validado.

---

**Projeto de portfólio independente** • Repositório: https://github.com/juceliocoelho2022/squaddesk-enterprise
